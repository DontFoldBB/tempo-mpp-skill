# Idea templates — готовые шаблоны для `buildResponse()` в `app/api/service/route.ts`

Этот файл читается агентом на Шаге 1 SKILL.md. Под выбранную пользователем идею агент подставляет соответствующую функцию `buildResponse()` плюс при необходимости меняет имя параметра запроса (например, `q` → `address`, `q` → `token`).

Все цены указаны в USDC.e как decimal string для `PRICE` env var: `'0.01'` = один цент, `'0.005'` = полцента, `'0.02'` = два цента. Это значение прокидывается в `mppx.charge({ amount: PRICE })` без дополнительной конвертации — библиотека сама знает что USDC.e имеет 6 decimals.

---

## 1. Token Info ($0.01, PRICE = '0.01')

**Что делает:** принимает адрес TIP-20 токена, возвращает метаданные (symbol, name, decimals, total supply, contract).

**Параметр запроса:** `token` (адрес токена)

**SERVICE_NAME suggestion:** `tempo-token-info`

**`lib/tempo.ts`** (создать дополнительно к стандартным `lib/chain.ts`, `lib/mpp.ts`, `lib/cache.ts`):

```ts
import { createPublicClient, http } from 'viem'
import { tempoMainnet } from './chain'

export const publicClient = createPublicClient({
  chain: tempoMainnet,
  transport: http(undefined, { batch: true, retryCount: 2, timeout: 15_000 }),
})

const ERC20_ABI = [
  { name: 'symbol', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'string' }] },
  { name: 'name', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'string' }] },
  { name: 'decimals', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint8' }] },
  { name: 'totalSupply', type: 'function', stateMutability: 'view', inputs: [], outputs: [{ type: 'uint256' }] },
] as const

export async function readTokenInfo(address: `0x${string}`) {
  const [symbol, name, decimals, totalSupply] = await Promise.all([
    publicClient.readContract({ address, abi: ERC20_ABI, functionName: 'symbol' }) as Promise<string>,
    publicClient.readContract({ address, abi: ERC20_ABI, functionName: 'name' }) as Promise<string>,
    publicClient.readContract({ address, abi: ERC20_ABI, functionName: 'decimals' }) as Promise<number>,
    publicClient.readContract({ address, abi: ERC20_ABI, functionName: 'totalSupply' }) as Promise<bigint>,
  ])
  return { symbol, name, decimals, totalSupply }
}
```

**`buildResponse()` для `app/api/service/route.ts`:**

```ts
import { formatUnits, getAddress } from 'viem'
import { readTokenInfo } from '@/lib/tempo'

async function buildResponse(input: string): Promise<object> {
  if (!/^0x[a-fA-F0-9]{40}$/.test(input)) {
    throw new Error('invalid_token_address')
  }
  const address = getAddress(input) as `0x${string}`
  const info = await readTokenInfo(address)
  return {
    address,
    chain: 'tempo-mainnet',
    chain_id: 4217,
    fetched_at: new Date().toISOString(),
    symbol: info.symbol,
    name: info.name,
    decimals: info.decimals,
    total_supply: {
      raw: info.totalSupply.toString(),
      formatted: formatUnits(info.totalSupply, info.decimals),
    },
  }
}
```

**Параметр в route.ts:** замени `searchParams.get('q')` на `searchParams.get('token')`.

**Параметр в openapi.json:** `name: 'token'` вместо `'q'`, `example: '0x20c000000000000000000000b9537d11c60e8b50'`.

---

## 2. Price Feed ($0.01, PRICE = '0.01')

**Что делает:** возвращает текущие USD цены всех известных Tempo-стейблов через CoinGecko (бесплатное API, ~10 req/min лимит).

**Параметр запроса:** нет (всегда возвращает все цены)

**SERVICE_NAME suggestion:** `tempo-price-feed`

**`lib/tokenlist.ts`** (создать):

```ts
import { LRUCache } from 'lru-cache'

const cache = new LRUCache<string, object>({ max: 2, ttl: 3_600_000 })

export type Token = {
  symbol: string
  name: string
  address: `0x${string}`
  decimals: number
  extensions?: { coingeckoId?: string }
}

export async function getTokenList(): Promise<Token[]> {
  const cached = cache.get('mainnet') as Token[] | undefined
  if (cached) return cached
  const res = await fetch('https://tokenlist.tempo.xyz/list/4217')
  if (!res.ok) throw new Error('tokenlist_fetch_failed')
  const data = await res.json() as { tokens?: Token[] }
  if (!Array.isArray(data.tokens)) throw new Error('tokenlist_shape')
  cache.set('mainnet', data.tokens)
  return data.tokens
}
```

**`buildResponse()`:**

```ts
import { getTokenList } from '@/lib/tokenlist'

async function buildResponse(_input: string): Promise<object> {
  const tokens = await getTokenList()
  const coingeckoIds = tokens
    .map(t => t.extensions?.coingeckoId)
    .filter((x): x is string => !!x)

  let prices: Record<string, { usd: number }> = {}
  if (coingeckoIds.length > 0) {
    const url = `https://api.coingecko.com/api/v3/simple/price?ids=${coingeckoIds.join(',')}&vs_currencies=usd`
    const res = await fetch(url, { next: { revalidate: 60 } })
    if (res.ok) prices = await res.json()
  }

  return {
    fetched_at: new Date().toISOString(),
    chain: 'tempo-mainnet',
    tokens: tokens.map(t => ({
      symbol: t.symbol,
      address: t.address,
      coingecko_id: t.extensions?.coingeckoId || null,
      usd_price: t.extensions?.coingeckoId ? (prices[t.extensions.coingeckoId]?.usd ?? null) : 1.0,
    })),
  }
}
```

**Параметр в route.ts:** убери валидацию `if (!input)`, эндпоинт не требует параметров. В openapi.json — пустой `parameters: []`.

---

## 3. Gas Estimator ($0.005, PRICE = '0.005')

**Что делает:** возвращает текущую оценку газа на Tempo для разных типов операций.

**Параметр запроса:** нет

**SERVICE_NAME suggestion:** `tempo-gas-estimator`

**`lib/tempo.ts`** (как в Token Info, или просто):

```ts
import { createPublicClient, http } from 'viem'
import { tempoMainnet } from './chain'

export const publicClient = createPublicClient({
  chain: tempoMainnet,
  transport: http(),
})
```

**`buildResponse()`:**

```ts
import { formatUnits } from 'viem'
import { publicClient } from '@/lib/tempo'

async function buildResponse(_input: string): Promise<object> {
  const [gasPrice, block] = await Promise.all([
    publicClient.getGasPrice(),
    publicClient.getBlock({ blockTag: 'latest' }),
  ])

  // Approximate gas usage for common ops on Tempo
  const ops = {
    transfer_to_existing: 100_000n,
    transfer_to_new: 300_000n, // state creation higher on Tempo
    contract_call_simple: 150_000n,
    contract_call_complex: 500_000n,
  }

  const estimates = Object.fromEntries(
    Object.entries(ops).map(([k, gas]) => [
      k,
      {
        gas_units: gas.toString(),
        cost_wei: (gas * gasPrice).toString(),
        cost_usd_approx: parseFloat(formatUnits(gas * gasPrice, 6)).toFixed(6), // gas in stablecoin
      },
    ])
  )

  return {
    fetched_at: new Date().toISOString(),
    chain: 'tempo-mainnet',
    block_number: block.number.toString(),
    gas_price_wei: gasPrice.toString(),
    estimates,
    note: 'Gas on Tempo is paid in stablecoins, not native token',
  }
}
```

**В route.ts:** убери валидацию параметра.

---

## 4. Activity Score ($0.01, PRICE = '0.01')

**Что делает:** простой скор активности кошелька от 0 до 100 на основе количества транзакций и баланса.

**Параметр запроса:** `address` (кошелёк)

**SERVICE_NAME suggestion:** `tempo-activity-score`

**`lib/tempo.ts`** (создать):

```ts
import { createPublicClient, http } from 'viem'
import { tempoMainnet, USDC_E_ADDRESS } from './chain'

export const publicClient = createPublicClient({
  chain: tempoMainnet,
  transport: http(undefined, { batch: true }),
})

const BALANCE_OF_ABI = [
  { name: 'balanceOf', type: 'function', stateMutability: 'view',
    inputs: [{ name: 'account', type: 'address' }], outputs: [{ type: 'uint256' }] },
] as const

export async function readUSDCBalance(address: `0x${string}`): Promise<bigint> {
  return publicClient.readContract({
    address: USDC_E_ADDRESS,
    abi: BALANCE_OF_ABI,
    functionName: 'balanceOf',
    args: [address],
  }) as Promise<bigint>
}

export async function readNonce(address: `0x${string}`): Promise<number> {
  return publicClient.getTransactionCount({ address })
}

export async function readCode(address: `0x${string}`): Promise<string> {
  const code = await publicClient.getCode({ address })
  return code || '0x'
}
```

**`buildResponse()`:**

```ts
import { formatUnits, getAddress } from 'viem'
import { readUSDCBalance, readNonce, readCode } from '@/lib/tempo'

async function buildResponse(input: string): Promise<object> {
  if (!/^0x[a-fA-F0-9]{40}$/.test(input)) {
    throw new Error('invalid_address')
  }
  const address = getAddress(input) as `0x${string}`

  const [balance, nonce, code] = await Promise.all([
    readUSDCBalance(address),
    readNonce(address),
    readCode(address),
  ])

  const isContract = code !== '0x'
  const balanceFloat = parseFloat(formatUnits(balance, 6))

  // Simple scoring: nonce contributes up to 60, balance up to 40
  const nonceScore = Math.min(60, Math.floor(Math.log10(nonce + 1) * 20))
  const balanceScore = Math.min(40, Math.floor(Math.log10(balanceFloat + 1) * 15))
  const score = Math.min(100, nonceScore + balanceScore)

  return {
    address,
    chain: 'tempo-mainnet',
    fetched_at: new Date().toISOString(),
    score, // 0-100
    breakdown: {
      account_type: isContract ? 'contract' : 'EOA',
      nonce, // opaque on Tempo (2D nonces)
      usdc_balance: balanceFloat,
      nonce_contribution: nonceScore,
      balance_contribution: balanceScore,
    },
    note: 'Score is heuristic, intended for relative comparison',
  }
}
```

**Параметр в route.ts и openapi.json:** `address` вместо `q`, `example: '0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266'`.

---

## 5. Random Stable Fact ($0.005, PRICE = '0.005')

**Что делает:** возвращает случайный факт про стейблкоины. Самый простой вариант для первого MPP-сервиса.

**Параметр запроса:** нет

**SERVICE_NAME suggestion:** `random-stable-fact`

Не нужны дополнительные lib-файлы.

**`buildResponse()`:**

```ts
const FACTS = [
  { fact: 'USDC.e on Tempo is a Stargate-bridged version of Ethereum USDC, not native USDC issued by Circle.', source: 'tokenlist.tempo.xyz' },
  { fact: 'Tempo gas fees are paid in stablecoins, not in a native gas token. This is unusual for EVM chains.', source: 'docs.tempo.xyz' },
  { fact: 'Stablecoin market cap exceeded $200 billion in 2024 — larger than many national currencies in circulation.', source: 'defillama.com' },
  { fact: 'Tether (USDT) processes more daily transaction volume than Visa on some days, mostly on Tron.', source: 'public chain data' },
  { fact: 'A "stable" coin is only as stable as its backing. Algorithmic stablecoins like UST collapsed in 2022.', source: 'history' },
  { fact: 'PYUSD is a stablecoin launched by PayPal in 2023, issued by Paxos.', source: 'paypal.com' },
  { fact: 'On Tempo, the same TIP-20 token can be used to pay gas fees, instead of a separate native asset.', source: 'docs.tempo.xyz' },
  { fact: 'EURC is a euro-denominated stablecoin issued by Circle, the same company behind USDC.', source: 'circle.com' },
  { fact: 'Stablecoins held in DeFi protocols can earn yield through lending, but the yield comes from borrowers, not the issuer.', source: 'defi basics' },
  { fact: 'The Tempo blockchain launched in 2026 with $500M funding from Stripe and Paradigm.', source: 'announcement' },
]

async function buildResponse(_input: string): Promise<object> {
  const idx = Math.floor(Math.random() * FACTS.length)
  return {
    fact: FACTS[idx].fact,
    source: FACTS[idx].source,
    fetched_at: new Date().toISOString(),
    fact_id: idx + 1,
    total_facts: FACTS.length,
  }
}
```

**В route.ts:** убери валидацию параметра, убери кэш (каждый раз случайный факт).

В openapi.json: `parameters: []`, `summary: 'Get a random stablecoin fact'`.

---

## 6. Свой вариант

Если пользователь выбрал «свой вариант» на Шаге 1 — попроси описать в 1-2 предложениях:
- Что принимает на вход (адрес? токен? число? ничего?)
- Что возвращает (одно поле? объект? массив?)

Создай минимальный шаблон по аналогии с одной из 5 идей выше. Старайся переиспользовать `lib/tempo.ts` если нужны RPC-вызовы.

**Общие правила для любой идеи:**

1. **Не используй `eth_getBalance`** — он сломан на Tempo. Только `balanceOf` через viem `readContract`.
2. **Все RPC-вызовы через batch** — `transport: http(undefined, { batch: true })`.
3. **Кэшируй ответы** — стандартный `responseCache` из `lib/cache.ts`, ключ ≈ нормализованный input.
4. **Возвращай `fetched_at`** — ISO timestamp, помогает агентам понимать свежесть данных.
5. **Возвращай `chain` и `chain_id`** — `'tempo-mainnet'` и `4217`.
6. **Не возвращай `BigInt` напрямую** — JSON.stringify не умеет. Используй `.toString()` или `formatUnits()`.
