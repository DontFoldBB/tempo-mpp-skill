# Tempo MPP Builder

Sets up a paid HTTP service on the Tempo blockchain end-to-end: scaffolds a Next.js project, wires up MPP payment middleware, deploys to Vercel, and registers on MPPScan so AI agents can discover it.

Invoke this skill with:

```
Скачай https://raw.githubusercontent.com/DontFoldBB/tempo-mpp-skill/main/SKILL.md и собери мне MPP-сервис на Tempo
```

The assistant operates as a **build orchestrator**, not a tutor — it makes decisions, runs commands, and asks the user only when it actually needs input. All user-facing messages are in Russian.

---

## Operating principles

The orchestrator follows these throughout:

- **Russian for the human, English in code.** Comments, variables, error strings — English. Anything addressed to the user — Russian. Technical nouns (Vercel, env, MPP, USDC.e, RPC, bridge) stay as they are.
- **Announce each step before doing it.** Before starting a meaningful step (asking for input, running install, deploying, etc.), print a Russian step header like `📍 Шаг 1. Выбор идеи сервиса` followed by a one-line explanation of what's about to happen. This gives the user a clear sense of progress through the build. Don't announce trivial bookkeeping (file moves, single commands), only logical sub-steps.
- **Talk in completed states, not commentary.** After finishing a step, the orchestrator says what now exists, not what it just did. Example: «Проект готов, зависимости установлены» — not «Я запустил create-next-app и потом npm install».
- **When asking for input, remind where to get it.** If the value was set up earlier in the user's guide (Tempo address, project path), explicitly reference that step: «Вставь Tempo-адрес который ты создал на Шаге 4 гайда (wallet.tempo.xyz)». Don't assume the user remembers.
- **One question at a time.** If multiple inputs are needed, ask sequentially. No bulk forms.
- **Quote real values back to the user.** When confirming an address, price, or path — show the exact string the orchestrator captured, so the user can spot a typo.
- **Defer all spending decisions.** No `vercel --prod`, no actual mppx CLI payments — without an explicit «да / делай / поехали» from the user in the immediately preceding turn.
- **On error, switch to diagnostic mode.** If a command fails, the next message describes what failed in one or two lines and asks how to proceed (retry, skip, abort) — it does NOT silently rerun.

### Step naming scheme

Internal phase codes (A1, B2, D3, E1...) are for orchestrator memory only — never shown to the user. When announcing a step, translate to a friendly Russian name with sequential numbering starting from 1. The skill has its own step numbering independent of the published guide.

| Internal | What to announce to user |
| --- | --- |
| A1 | (skip — silent environment check) |
| A2 | 📍 Шаг 1. Выбор идеи сервиса |
| A3 | 📍 Шаг 2. Tempo-адрес для платежей |
| A4 | 📍 Шаг 3. Путь к папке проекта |
| B1–B6 | 📍 Шаг 4. Сборка проекта (одно общее сообщение в начале, без подшагов) |
| C | 📍 Шаг 5. Локальная проверка |
| D1–D4 | 📍 Шаг 6. Деплой на Vercel |
| E1 | (auto — prints summary) |
| E2 | 📍 Шаг 7. Что делаем дальше |
| E3 | 📍 Шаг 8. Реальный тест платежа |
| E4 | 📍 Шаг 8. Регистрация на MPPScan |
| E5 | 📍 Готово |

When asking for data the user prepared before launching the skill (Tempo address, project path), don't reference external "step numbers" — just remind where the value comes from in plain words ("адрес кошелька с wallet.tempo.xyz", "папка которую ты создал перед запуском Claude Code").

## Passing data between commands

Each shell invocation runs in a fresh process, so anything captured from one command needs to be re-captured or persisted for the next. Use a small file in the user's home directory:

```bash
SESSION="$HOME/.mpp-build.session"
touch "$SESSION"

# Read all known values
. "$SESSION" 2>/dev/null || true

# Write the full snapshot back (overwrite, not append — avoids duplicates)
cat > "$SESSION" <<EOF
PROJECT_DIR=${PROJECT_DIR:-}
RECIPIENT=${RECIPIENT:-}
SLUG=${SLUG:-}
PRICE=${PRICE:-}
SECRET=${SECRET:-}
PROD_URL=${PROD_URL:-}
EOF
```

Wipe the file at the very end of the build.

---

# Phase A — Discovery

## A1. Environment audit

Run these and verify each one:

```bash
node --version    # need >=20
npm --version
git --version
curl --version | head -1
```

If Node is missing or below 20, tell the user to install it from nodejs.org and stop. Don't try to install Node — too system-specific.

## A2. Service idea

Pull the idea catalog so concrete templates are available later:

```bash
curl -fsSL https://raw.githubusercontent.com/DontFoldBB/tempo-mpp-skill/main/docs/idea-templates.md -o /tmp/mpp-ideas.md
```

Then announce the step and ask the user (use this exact format):

> 📍 **Шаг 1. Выбор идеи сервиса**
>
> Сейчас выберем что именно будет делать твой MPP-сервис. Есть пять готовых вариантов с уже написанным кодом, можно взять любой. Каждый агент будет платить указанную цену в USDC.e за один вызов.
>
> **A. Token Info — $0.01** — агент шлёт адрес TIP-20 токена, получает метаданные (символ, decimals, total supply)
> **B. Price Feed — $0.01** — текущие цены всех Tempo-стейблов через CoinGecko
> **C. Gas Estimator — $0.005** — оценка стоимости газа на Tempo для разных операций
> **D. Activity Score — $0.01** — простой скор активности кошелька от 0 до 100
> **E. Random Stable Fact — $0.005** — случайные факты про стейблкоины. Самый простой вариант если делаешь в первый раз
> **F. Своя идея** — опиши что принимает и что возвращает, адаптирую ближайший шаблон
>
> Какую делаем? (буква или название)

Map free-form answers loosely (e.g. «цены» → B, «гайз» → C). If F, ask follow-up: «Что принимает на вход и что возвращает?» — then pick the template closest in spirit and adapt.

Capture into the session file:

- `SLUG` — short kebab-case identifier (e.g. `tempo-token-info`)
- `IDEA_KEY` — single letter A–F so later steps know which template to use
- `PRICE` — decimal string in USDC.e tokens (NOT base units). Examples: `'0.01'`, `'0.005'`, `'0.02'`. The mppx library handles the 6-decimal conversion internally — passing `'10000'` would mean **ten thousand USDC.e**, not one cent.

Once captured, confirm to the user in one line what's queued up.

## A3. Recipient address

Announce and ask:

> 📍 **Шаг 2. Tempo-адрес для платежей**
>
> Нужен адрес Tempo-кошелька на который будут приходить платежи от агентов. Используй тот, который ты создал на wallet.tempo.xyz через passkey (Touch ID, Windows Hello, отпечаток на телефоне).
>
> Адрес начинается с `0x` и состоит из 42 символов. Вставь его сюда:

Validate the answer with `^0x[a-fA-F0-9]{40}$`. If invalid, point out what's wrong (length, prefix, non-hex char) and ask again. Save lowercase variant as `RECIPIENT`.

Confirm by echoing the address back.

## A4. Project location

Announce and ask:

> 📍 **Шаг 3. Путь к папке проекта**
>
> Куда положить код. Это должна быть пустая папка которую ты создал перед запуском Claude Code.
>
> Полный путь, например:
> - macOS: `/Users/твоё_имя/Documents/tempo-mpp`
> - Windows: `C:\Users\твоё_имя\Documents\tempo-mpp`

Don't create anything yet — first check for capital letters in the final path segment. npm's `create-next-app` rejects them as project name. If found, tell the user the workaround:

> В имени папки есть заглавные буквы. npm их не любит. Создам проект в `<lowercase-variant>-build/` рядом, потом перенесу файлы куда сказал.

Save `PROJECT_DIR` (the user's chosen path, where files will live after the move) and `SCAFFOLD_DIR` (lowercase path used for `create-next-app`). If no capitals, both point to the same place.

---

# Phase B — Local build

Before starting, announce to the user:

> 📍 **Шаг 4. Сборка проекта**
>
> Получил все данные, запускаю сборку. Сейчас создам Next.js проект, поставлю зависимости, напишу код твоего сервиса, сгенерирую секрет и сделаю первый git-коммит. Это займёт пару минут, делаю всё сам, иногда буду спрашивать разрешение на команды — жми «yes» / «1».

## B1. Scaffolding

```bash
mkdir -p "$(dirname "$SCAFFOLD_DIR")"
cd "$(dirname "$SCAFFOLD_DIR")"
npx create-next-app@latest "$(basename "$SCAFFOLD_DIR")" \
  --typescript --eslint --tailwind --app --no-src-dir \
  --import-alias "@/*" --use-npm --yes
```

If `SCAFFOLD_DIR != PROJECT_DIR`, move:

```bash
mkdir -p "$PROJECT_DIR"
cp -a "$SCAFFOLD_DIR/." "$PROJECT_DIR/"
rm -rf "$SCAFFOLD_DIR"
```

## B2. Dependencies

```bash
cd "$PROJECT_DIR"
npm install mppx viem lru-cache
npm install -D tsx
```

If any install fails — most often a registry timeout — retry once before bailing.

## B3. Secret generation

The MPP secret must be exactly 64 hex characters. `randomBytes(32).toString('hex')` occasionally returns 63 chars when the leading byte is zero — regenerate until length matches.

```bash
while :; do
  SECRET=$(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
  [ ${#SECRET} -eq 64 ] && break
done
```

Write env files:

```bash
cat > "$PROJECT_DIR/.env.local" <<EOF
MPP_RECIPIENT=$RECIPIENT
MPP_SECRET_KEY=$SECRET
EOF

cat > "$PROJECT_DIR/.env.example" <<'EOF'
MPP_RECIPIENT=
MPP_SECRET_KEY=
EOF
```

Make sure `.env.local` is gitignored:

```bash
grep -qE '^\.env(\*\.local|\.local)$' "$PROJECT_DIR/.gitignore" || echo ".env*.local" >> "$PROJECT_DIR/.gitignore"
```

## B4. Source files

Generate the following files. The patterns below reflect what actually works with the current `mppx` package — there are several non-obvious gotchas that have been pre-resolved.

### `lib/chain.ts`

```ts
import { defineChain } from 'viem'

export const tempoMainnet = defineChain({
  id: 4217,
  name: 'Tempo Mainnet',
  nativeCurrency: { name: 'USD', symbol: 'USD', decimals: 18 },
  rpcUrls: { default: { http: ['https://rpc.tempo.xyz'] } },
  blockExplorers: { default: { name: 'Tempo Explorer', url: 'https://explore.tempo.xyz' } },
})

export const USDC_E_ADDRESS = '0x20c000000000000000000000b9537d11c60e8b50' as const
```

### `lib/mpp.ts`

Two critical points:
1. Import from `mppx/server`, not `mppx/nextjs` — the `/nextjs` subpath doesn't exist in the published package.
2. Use `tempo.charge({ ... })`, not `tempo({ ... })`. The shorthand registers both `charge` and `session` intents; `session` requires server-side signing infrastructure that doesn't exist here, and the build will fail.

```ts
import { Mppx, tempo } from 'mppx/server'
import { USDC_E_ADDRESS } from './chain'

const RECIPIENT = process.env.MPP_RECIPIENT
if (!RECIPIENT || !/^0x[a-fA-F0-9]{40}$/.test(RECIPIENT)) {
  throw new Error('MPP_RECIPIENT env var is required and must be a 0x address')
}

export const mppx = Mppx.create({
  methods: [
    tempo.charge({
      currency: USDC_E_ADDRESS,
      recipient: RECIPIENT as `0x${string}`,
    }),
  ],
})

// PRICE is a decimal string in USDC.e tokens, NOT base units.
// '0.01' = one cent. mppx handles the 6-decimal conversion internally.
export const PRICE = process.env.PRICE || '0.01'
```

### `lib/cache.ts`

```ts
import { LRUCache } from 'lru-cache'

export const responseCache = new LRUCache<string, object>({ max: 1000, ttl: 60_000 })
```

### `app/api/service/route.ts`

Manual 402 flow (because we're on `mppx/server`, not the higher-level Next adapter).

**Critical ordering:** the payment middleware must run **before** input validation. The MPP spec requires that an unauthenticated request returns 402 with a `WWW-Authenticate: Payment` header regardless of request body validity — registries like MPPScan probe the endpoint without any params to discover pricing, and will reject the service if they get a 400 instead of a 402.

```ts
import { mppx, PRICE } from '@/lib/mpp'
import { responseCache } from '@/lib/cache'

export const runtime = 'nodejs'
export const dynamic = 'force-dynamic'

export async function GET(request: Request): Promise<Response> {
  // 1. Payment middleware FIRST — must return 402 before any validation,
  //    so MPPScan and agents can discover pricing on an empty request.
  const charge = mppx.charge({ amount: PRICE })
  const result = await charge(request)
  if (result.status === 402) return result.challenge

  // 2. Payment confirmed — now validate inputs. Wrap any error response
  //    with result.withReceipt() so the client gets proof of payment
  //    even when the request itself is malformed.
  const url = new URL(request.url)
  const input = url.searchParams.get('q')
  if (!input) {
    return result.withReceipt(
      Response.json({ error: 'missing_parameter' }, { status: 400 })
    )
  }

  // 3. Cache hit
  const cacheKey = input.toLowerCase()
  const cached = responseCache.get(cacheKey)
  if (cached) {
    return result.withReceipt(
      Response.json({ ...cached, cached: true })
    )
  }

  // 4. Cache miss — do the work
  try {
    const payload = await buildResponse(input)
    responseCache.set(cacheKey, payload)
    return result.withReceipt(
      Response.json(payload, { headers: { 'Cache-Control': 'private, no-store' } })
    )
  } catch (err) {
    console.error('service_error', err)
    return result.withReceipt(
      Response.json({ error: 'internal_error' }, { status: 500 })
    )
  }
}

async function buildResponse(input: string): Promise<object> {
  // Replaced per-idea — see docs/idea-templates.md
  return { input, result: 'TODO' }
}
```

Replace `buildResponse` with the body from `/tmp/mpp-ideas.md` matching `IDEA_KEY`. If the idea uses a different query parameter (`token`, `address` instead of `q`), update the `searchParams.get(...)` call and the OpenAPI parameter name too.

### `app/openapi.json/route.ts`

MPPScan parses this file to register the service. Three requirements:
1. `x-payment-info` must be a flat object — `{ intent, method, amount, currency, description }`. The nested `offers: [...]` form gets rejected.
2. `servers[0].url` should resolve to the Vercel production URL at runtime — use `VERCEL_PROJECT_PRODUCTION_URL` first, fall back to `VERCEL_URL`, then to localhost for dev.
3. **`amount` in `x-payment-info` must be in base units (integer string)**, not decimal — `'10000'` for $0.01, not `'0.01'`. This is the inverse of what `mppx.charge()` expects in the route handler, where amount is a decimal string. The code below handles the conversion via `Math.round(Number(PRICE) * 1_000_000)`.

```ts
export const runtime = 'nodejs'

export async function GET(_req?: Request): Promise<Response> {
  const baseUrl =
    (process.env.VERCEL_PROJECT_PRODUCTION_URL && `https://${process.env.VERCEL_PROJECT_PRODUCTION_URL}`) ||
    (process.env.VERCEL_URL && `https://${process.env.VERCEL_URL}`) ||
    'http://localhost:3000'

  // PRICE is decimal string from env (used by mppx.charge), but openapi requires
  // base units (integer string in token units * 10^decimals). USDC.e has 6 decimals,
  // so PRICE='0.01' becomes priceBaseUnits='10000' for the discovery document.
  const PRICE = process.env.PRICE || '0.01'
  const priceBaseUnits = Math.round(Number(PRICE) * 1_000_000).toString()
  const priceDisplay = Number(PRICE).toFixed(PRICE.includes('.') ? PRICE.split('.')[1].length : 2)

  const doc = {
    openapi: '3.1.0',
    info: {
      title: process.env.SERVICE_NAME || 'MPP Service',
      version: '1.0.0',
      'x-guidance': 'Paid endpoints require MPP payment. Use npx mppx <url> or any MPP-compatible client.',
    },
    servers: [{ url: baseUrl }],
    'x-service-info': {
      categories: ['data'],
      docs: { homepage: baseUrl, apiReference: baseUrl },
    },
    paths: {
      '/api/service': {
        get: {
          summary: 'Main paid endpoint',
          parameters: [
            { name: 'q', in: 'query', required: true, schema: { type: 'string' }, example: 'example-input' },
          ],
          'x-payment-info': {
            intent: 'charge',
            method: 'tempo',
            amount: priceBaseUnits,
            currency: '0x20c000000000000000000000b9537d11c60e8b50',
            description: `$${priceDisplay} per request`,
          },
          responses: {
            '200': { description: 'Success' },
            '400': { description: 'Invalid input' },
            '402': { description: 'Payment Required' },
          },
        },
      },
    },
  }

  return Response.json(doc, { headers: { 'Cache-Control': 'public, max-age=300' } })
}
```

### `app/page.tsx`

This is a landing page primarily for *agents and crawlers* (MPPScan, mpp.dev) to see when they hit the root. Keep it minimal — a wallet-connect UI is pointless here because regular users don't pay via browser, agents do.

```tsx
const SERVICE_NAME = 'MPP Service' // adjust per build

export default function Home() {
  return (
    <main style={{
      fontFamily: 'ui-monospace, SFMono-Regular, Menlo, monospace',
      background: '#0a0a0a', color: '#e5e5e5', minHeight: '100vh',
      padding: '2rem', maxWidth: '720px', margin: '0 auto',
    }}>
      <h1 style={{ color: '#5dcaa5', marginBottom: '0.5rem' }}>{SERVICE_NAME}</h1>
      <p style={{ color: '#888', marginBottom: '2rem' }}>Paid HTTP API on Tempo via MPP.</p>

      <h3>Call (paid)</h3>
      <pre style={{ background: '#050505', padding: '1rem', border: '1px solid #2a2a2a', overflowX: 'auto' }}>
        npx mppx 'https://{baseUrl}/api/service?q=example'
      </pre>

      <h3>Discovery (free)</h3>
      <pre style={{ background: '#050505', padding: '1rem', border: '1px solid #2a2a2a', overflowX: 'auto' }}>
        curl 'https://{baseUrl}/openapi.json'
      </pre>
    </main>
  )
}
```

`baseUrl` resolves at build time from `process.env.VERCEL_PROJECT_PRODUCTION_URL` (set automatically by Vercel) — add this at the top of `page.tsx`:

```tsx
const SERVICE_NAME = 'MPP Service' // adjust per build
const baseUrl = process.env.VERCEL_PROJECT_PRODUCTION_URL || 'localhost:3000'
```

### `vercel.json`

Important: do NOT include a `runtime` field. Vercel rejects `"runtime": "nodejs20.x"` with «Function Runtimes must have a valid version». Only `maxDuration` is needed; Vercel infers the runtime from the project's Node version.

```json
{
  "functions": {
    "app/api/service/route.ts": { "maxDuration": 30 },
    "app/openapi.json/route.ts": { "maxDuration": 10 }
  }
}
```

## B5. Build check

```bash
cd "$PROJECT_DIR"
npm run build 2>&1 | tail -25
```

If it fails, consult the failure modes section below before retrying anything. Don't blindly rerun the same command — `npm run build` is slow and most failures need a code fix.

## B6. First commit

```bash
cd "$PROJECT_DIR"
git init 2>/dev/null || true
git add -A
git commit -m "scaffold $SLUG"
```

---

# Phase C — Local verification

Announce to the user:

> 📍 **Шаг 5. Локальная проверка**
>
> Перед деплоем на Vercel прогоню тесты локально — это быстрее чем итерировать через деплой. Запущу dev-сервер, проверю что 402 challenge возвращается правильно, что openapi.json валидный и т.д.

Bring up the dev server and prove that the wiring is correct *before* burning a Vercel deployment.

```bash
cd "$PROJECT_DIR"
npm run dev > /tmp/mpp-dev.log 2>&1 &
DEV_PID=$!
sleep 6
ps -p $DEV_PID >/dev/null || { tail -30 /tmp/mpp-dev.log; echo "dev server failed to start"; exit 1; }
```

Run four checks against `http://localhost:3000`:

```bash
# 1) Empty request (no params) → 402 with WWW-Authenticate
#    This is what MPPScan probes to discover pricing — must return 402, NOT 400.
curl -s -i 'http://localhost:3000/api/service' | \
  awk 'NR==1 || tolower($0) ~ /www-authenticate/'

# 2) Valid request without payment → 402 with WWW-Authenticate
curl -s -i 'http://localhost:3000/api/service?q=test' | \
  awk 'NR==1 || tolower($0) ~ /www-authenticate/'

# 3) Discovery doc is valid and uses flat payment info
curl -s 'http://localhost:3000/openapi.json' > /tmp/mpp-openapi.json
node -e "
  const d = require('/tmp/mpp-openapi.json');
  const op = d.paths?.['/api/service']?.get;
  const p = op?.['x-payment-info'];
  console.log('openapi:', d.openapi);
  console.log('servers[0]:', d.servers?.[0]?.url);
  console.log('payment shape:', p && !Array.isArray(p.offers) ? 'flat (good)' : 'nested offers[] (will break MPPScan)');
  console.log('payment amount:', p?.amount, 'method:', p?.method);
"

# 4) Landing returns 200
curl -s -o /dev/null -w 'landing: HTTP %{http_code}\n' http://localhost:3000/
```

Expected:
- Check 1 prints `HTTP/... 402` with `WWW-Authenticate: Payment ...` (NOT 400 — payment middleware must run first)
- Check 2 prints an `HTTP/... 402` line followed by `WWW-Authenticate: Payment ...`
- Check 3 prints `flat (good)` and shows the amount/method
- Check 4 prints `HTTP 200`

If Check 1 returns 400 instead of 402, the handler has validation before `mppx.charge()` — this will cause MPPScan to reject the service. Fix the order in `app/api/service/route.ts` (charge first, validation after) and rerun.

Stop the dev server:

```bash
kill $DEV_PID 2>/dev/null; wait $DEV_PID 2>/dev/null
```

Confirm to the user that local checks passed and ask whether to deploy.

---

# Phase D — Deploy

Announce to the user:

> 📍 **Шаг 6. Деплой на Vercel**
>
> Локально всё работает. Сейчас залью на Vercel и пропишу env vars через CLI. Если ещё не залогинен в Vercel — откроется браузер для авторизации.

## D1. Vercel CLI

```bash
command -v vercel >/dev/null || npm install -g vercel
```

## D2. First deploy

Tell the user that `vercel` will open a browser for login if needed, and that the first run creates a fresh project with default settings.

```bash
cd "$PROJECT_DIR"
vercel
```

After the command finishes, capture the URL it prints (`https://<slug>-<hash>.vercel.app`) into `PROD_URL` — actually this is the preview URL. The user is still about to do the production deploy.

## D3. Env vars

`MPP_RECIPIENT` and `MPP_SECRET_KEY` must exist in Vercel production env before `vercel --prod` succeeds — otherwise the recipient validation in `lib/mpp.ts` throws at module load and the deploy crashes.

These can be added entirely via CLI, no dashboard visit required. `vercel env add` accepts the variable name and target environments as args, then reads the value from stdin:

```bash
cd "$PROJECT_DIR"

# Pipe values via stdin to avoid interactive prompts
printf '%s' "$RECIPIENT" | vercel env add MPP_RECIPIENT production preview 2>&1 | tail -3
printf '%s' "$SECRET"    | vercel env add MPP_SECRET_KEY production preview 2>&1 | tail -3
```

If either command fails because the variable already exists (the user re-ran the skill in the same project), remove first then re-add:

```bash
vercel env rm MPP_RECIPIENT production --yes 2>/dev/null
vercel env rm MPP_RECIPIENT preview --yes 2>/dev/null
printf '%s' "$RECIPIENT" | vercel env add MPP_RECIPIENT production preview
```

Same pattern for `MPP_SECRET_KEY`.

Verify both are now set:

```bash
vercel env ls production 2>&1 | grep -E 'MPP_RECIPIENT|MPP_SECRET_KEY'
```

Should print both names.

Quickly explain to the user what was just done — they didn't see the secret values, so a one-liner acknowledging that env is set on Production+Preview is enough. No need to ask them to confirm anything in the dashboard.

Now trigger the production deploy with the new env values:

```bash
cd "$PROJECT_DIR"
vercel --prod 2>&1 | tail -8
```

Capture the production URL into `PROD_URL`.

## D4. Production smoke test

Same four checks as Phase C, but against `$PROD_URL`. If anything fails, it's almost always env vars that didn't propagate to Production scope — ask the user to double-check the Environment Variables page and recheck both `Production` and `Preview` boxes.

---

# Phase E — Hand-off to user

By this point the service is live. The remaining steps depend on what the user wants to do next.

## E1. Result summary

Print a comprehensive summary so the user can poke around and see their work. Use the actual values captured during the build, not placeholders.

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Готово, MPP-сервис задеплоен

Имя:           <SLUG>
Идея:          <SERVICE_IDEA в одну строку>
Цена:          $<PRICE> USDC.e за запрос
Получатель:    <RECIPIENT>

Куда зайти посмотреть:

  Главная страница (открой в браузере):
    https://<PROD_URL>

  OpenAPI документ (это читают агенты):
    https://<PROD_URL>/openapi.json

  Vercel Dashboard (живые логи каждого запроса):
    https://vercel.com/dashboard

  Tempo Explorer (входящие транзакции на твой адрес):
    https://explore.tempo.xyz/address/<RECIPIENT>

  Папка с кодом проекта на компе:
    <PROJECT_DIR>

Полезные команды (если захочешь править):
  cd <PROJECT_DIR>
  vercel --prod                                  # передеплоить после правок
  npm run dev                                    # локально на порту 3000
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

`PROD_URL` is the `*.vercel.app` URL captured after `vercel --prod` in D3.

## E2. Choose next step

Ask the user which path they want:

> 📍 **Шаг 7. Что делаем дальше**
>
> Сервис уже задеплоен и работает. Можно зайти посмотреть результат по ссылкам выше. Дальше есть три варианта:
>
> 1) **Реальный тест платежа + регистрация на MPPScan.** Полный путь до конца. Создам тестовый кошелёк, ты бриджнёшь $0.50-1 USDC через Stargate, я сделаю первый платёж в твой сервис, потом зарегистрируем в каталоге. Самый кайфовый, рекомендую.
>
> 2) **Только регистрация на MPPScan.** Закинем сервис в каталог чтоб его могли найти агенты. Без теста платежа.
>
> 3) **Завершить.** Сервис уже работает, остальное сделаю позже сам.

Branch on the answer:

- 1 → go to E3 (Payment test), then automatically continue to E4 (MPPScan registration), then E5 (wrap up)
- 2 → go to E4 (MPPScan), then E5
- 3 → go to E5 (wrap up)

## E3. Real payment test

Announce to the user:

> 📍 **Шаг 8. Реальный тест платежа**
>
> Сейчас сделаем первый настоящий платёж через твой сервис. Я создам тестовый кошелёк, ты закинешь в него немного USDC.e через Stargate, потом я этим тестовым кошельком оплачу запрос к твоему API. Деньги придут на твой recipient адрес.

Create a fresh wallet for testing — separate from the recipient wallet. The user will fund this one and the assistant will use it to call the live service.

```bash
cd "$PROJECT_DIR"
cat > test-payment.ts <<'EOF'
import { privateKeyToAccount, generatePrivateKey } from 'viem/accounts'
import { Mppx, tempo } from 'mppx/client'

async function main() {
  const pk = generatePrivateKey()
  const account = privateKeyToAccount(pk)
  console.log('TEST_PRIVATE_KEY:', pk)
  console.log('TEST_ADDRESS:', account.address)
}
main()
EOF

npx tsx test-payment.ts 2>&1 | tail -5
```

Capture `TEST_ADDRESS` from output. Tell the user:

> Создал тестовый кошелёк: `<TEST_ADDRESS>`
>
> Теперь надо закинуть туда немного USDC.e в сеть Tempo. Идём на Stargate:
>
> https://stargate.finance/?dstChain=tempo&dstToken=0x20c000000000000000000000b9537d11c60e8b50
>
> 1) Подключи свой обычный EVM кошелёк (MetaMask / Rabby) с USDC на любой сети
> 2) В поле "To Address" вставь: `<TEST_ADDRESS>`
> 3) Сумма: $0.50-1 USDC. Меньше нет смысла, потому что Stargate берёт минимум $0.20-0.30 комиссии за бридж независимо от суммы
> 4) Подтверди транзакцию в своём кошельке
> 5) Когда деньги придут (1-3 минуты), напиши «готово»

Wait. When user confirms, verify balance:

```bash
node -e "
const { createPublicClient, http } = require('viem')
const { defineChain } = require('viem')
const tempo = defineChain({
  id: 4217, name: 'Tempo',
  nativeCurrency: { name: 'USD', symbol: 'USD', decimals: 18 },
  rpcUrls: { default: { http: ['https://rpc.tempo.xyz'] } }
})
const client = createPublicClient({ chain: tempo, transport: http() })
const USDC_E = '0x20c000000000000000000000b9537d11c60e8b50'
const ABI = [{ name: 'balanceOf', type: 'function', stateMutability: 'view', inputs: [{ name: 'a', type: 'address' }], outputs: [{ type: 'uint256' }] }]
client.readContract({ address: USDC_E, abi: ABI, functionName: 'balanceOf', args: ['$TEST_ADDRESS'] })
  .then(b => console.log('balance:', Number(b) / 1e6, 'USDC.e'))
  .catch(e => console.error('error:', e.message))
"
```

If balance is 0 — bridge not done yet, ask user to wait another minute and check Stargate status. If non-zero — proceed.

Make the actual paid call using the test wallet:

```bash
cat > test-call.ts <<EOF
import { privateKeyToAccount } from 'viem/accounts'
import { Mppx, tempo } from 'mppx/client'

async function main() {
  const account = privateKeyToAccount('TEST_PK_HERE' as \`0x\${string}\`)
  const mppx = Mppx.create({ methods: [tempo.charge({ account })] })
  const url = 'https://<PROD_URL>/api/service?q=test-value'
  const res = await mppx.fetch(url)
  const data = await res.json()
  console.log('STATUS:', res.status)
  console.log('DATA:', JSON.stringify(data, null, 2))
}
main()
EOF

# Substitute the captured private key in the file, then run
sed -i.bak "s|TEST_PK_HERE|$TEST_PK|" test-call.ts
npx tsx test-call.ts 2>&1 | tail -30
```

Adjust the query parameter (`q=test-value`) to whatever is valid for the chosen idea — e.g. an address for Activity Score, a token for Token Info.

Show the response to the user and explain:

> 🎉 Платёж прошёл, агент получил ответ от твоего сервиса.
>
> Что произошло:
> 1. Тестовый кошелёк (`<TEST_ADDRESS>`) заплатил $<PRICE> USDC.e
> 2. Деньги пришли на твой recipient `<RECIPIENT>`
> 3. Сервис вернул JSON ответ (выше)
>
> Где это видно прямо сейчас:
> - Explorer: https://explore.tempo.xyz/address/<RECIPIENT> (последняя входящая транза)
> - Vercel Logs: https://vercel.com/dashboard (запрос с 200 ответом)
> - MPPScan (если зарегистрировал): на странице сервиса в Transactions

Before deleting the test wallet files, sweep remaining balance back to the recipient — otherwise that $0.40+ is locked forever (test private key gets deleted). On Tempo gas is paid in stablecoins, so we send `balance - small reserve` to cover the transfer fee itself.

Tell the user:

> На тестовом кошельке ещё остались деньги ($0.40-0.90 после теста). Сейчас переведу их на твой основной recipient адрес, чтоб не потерялись.

```bash
cd "$PROJECT_DIR"
cat > sweep-test.ts <<EOF
import { privateKeyToAccount } from 'viem/accounts'
import { createWalletClient, createPublicClient, http, parseAbi, encodeFunctionData } from 'viem'
import { defineChain } from 'viem'

const tempo = defineChain({
  id: 4217, name: 'Tempo',
  nativeCurrency: { name: 'USD', symbol: 'USD', decimals: 18 },
  rpcUrls: { default: { http: ['https://rpc.tempo.xyz'] } }
})

const USDC_E = '0x20c000000000000000000000b9537d11c60e8b50' as \`0x\${string}\`
const RECIPIENT = 'RECIPIENT_HERE' as \`0x\${string}\`
const PK = 'TEST_PK_HERE' as \`0x\${string}\`

const erc20Abi = parseAbi([
  'function balanceOf(address) view returns (uint256)',
  'function transfer(address to, uint256 amount) returns (bool)',
])

async function main() {
  const account = privateKeyToAccount(PK)
  const publicClient = createPublicClient({ chain: tempo, transport: http() })
  const walletClient = createWalletClient({ account, chain: tempo, transport: http() })

  const balance = await publicClient.readContract({
    address: USDC_E, abi: erc20Abi, functionName: 'balanceOf', args: [account.address]
  }) as bigint
  console.log('Balance on test wallet:', Number(balance) / 1e6, 'USDC.e')

  if (balance === 0n) {
    console.log('Nothing to sweep')
    return
  }

  // Reserve a small amount for gas on Tempo (~$0.005 in USDC.e base units = 5000)
  const GAS_RESERVE = 5000n
  if (balance <= GAS_RESERVE) {
    console.log('Balance too small to sweep (would not cover gas)')
    return
  }
  const sendAmount = balance - GAS_RESERVE

  const hash = await walletClient.writeContract({
    address: USDC_E, abi: erc20Abi, functionName: 'transfer',
    args: [RECIPIENT, sendAmount],
  })
  console.log('Sweep tx:', hash)
  console.log('Sent:', Number(sendAmount) / 1e6, 'USDC.e to', RECIPIENT)
}
main().catch(e => { console.error('Sweep failed:', e.message); process.exit(1) })
EOF

sed -i.bak "s|TEST_PK_HERE|$TEST_PK|" sweep-test.ts
sed -i.bak "s|RECIPIENT_HERE|$RECIPIENT|" sweep-test.ts
npx tsx sweep-test.ts 2>&1 | tail -10
```

If sweep succeeds, tell the user:

> Перевёл оставшиеся средства на твой recipient `<RECIPIENT>`. Tx: `<sweep_hash>`. Можешь проверить в explorer.

If sweep fails (RPC issue, gas problem) — don't block the flow, just tell the user:

> Sweep не получилось ($<balance> остались на тестовом кошельке). Если хочешь их забрать позже, приватный ключ был: `<TEST_PK>`. Сохрани в надёжное место. Иначе они там и зависнут.

Now clean up test files (they contain a private key):

```bash
rm -f test-payment.ts test-call.ts test-call.ts.bak sweep-test.ts sweep-test.ts.bak
```

After cleanup, proceed directly to E4 (MPPScan registration). Don't return to E2 menu — payment test naturally leads into listing the service in the catalog. Just announce the next step and continue:

> Платёж прошёл, остаток вернул, теперь самое время зарегистрировать сервис в каталоге MPPScan чтобы агенты могли найти автоматически.

Then continue with the E4 flow below.

## E4. MPPScan registration

> 📍 **Шаг 8. Регистрация на MPPScan**
>
> Закидываем сервис в каталог MPPScan чтоб агенты могли его найти автоматически. Это бесплатно, без модерации, один клик.
>
> 1) Открой https://mppscan.com/register
> 2) В поле Service URL введи: `https://<PROD_URL>` (это твой vercel.app URL)
> 3) Жми **+ Add Server**
> 4) MPPScan сам прочитает твой `/openapi.json`, увидит все эндпоинты и цены, сделает запись в каталоге
> 5) Получишь страницу вида `mppscan.com/server/<hash>` — это твоя карточка. Пришли её ссылку сюда, чтоб я добавил в саммари

Wait for the link. Verify it's a valid mppscan URL with `grep -qE '^https://(www\.)?mppscan\.com/server/[a-f0-9]+'`. If yes, save into the session as `MPPSCAN_URL`.

> ✅ Зарегистрировано. Твоя карточка: <MPPSCAN_URL>
>
> Теперь агенты могут найти сервис через каталог. На странице видны:
> - все эндпоинты и цены
> - все транзакции (история платежей)
> - статус сервиса (отвечает / не отвечает)

Continue to E5 (final wrap-up).

## E5. Wrap up

Print final reminder of where everything lives:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Всё, готово.

Сайт:       https://<PROD_URL>
Получатель: <RECIPIENT>
MPPScan:    <MPPSCAN_URL or "не зарегистрирован">
Код:        <PROJECT_DIR>

Если захочешь править сервис — открой <PROJECT_DIR>,
правь файлы, потом `vercel --prod` для деплоя.
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Clean up the session file:

```bash
rm -f "$HOME/.mpp-build.session"
```

End with a single line: `MPP_BUILD_DONE`. Downstream tooling looks for this exact string.

---

# Failure modes

Common ways things break, grouped by symptom.

### MPPScan warning: «Fixed-price endpoint missing WWW-Authenticate header on 402»

MPPScan probes the endpoint with an empty request and expects 402 with `WWW-Authenticate: Payment ...` before reading any params. If validation runs first and returns 400, MPPScan can't auto-discover pricing and shows this warning.

Fix: in `app/api/service/route.ts`, ensure `mppx.charge()` runs **before** any input validation. The first block of the handler should be:

```ts
const charge = mppx.charge({ amount: PRICE })
const result = await charge(request)
if (result.status === 402) return result.challenge
```

Validation comes after, and error responses wrap with `result.withReceipt(...)` so paying clients get proof of payment even when the body is malformed.

### Payment fails with `InsufficientBalance` even when wallet has funds

The `amount` passed to `mppx.charge()` is being treated as a number that's too large. This happens when code passes base units (e.g. `'10000'` for $0.01) instead of a decimal string. The `mppx.charge()` API takes a **decimal string in token units**, not base units — `'0.01'` is one cent, not `'10000'`. The library handles the 6-decimal conversion internally.

Quick check:
```bash
curl -i 'https://<PROD_URL>/api/service?q=test' | grep -i www-authenticate
```

If the `amount` in the 402 challenge has many more zeros than expected (e.g. `10000000000` instead of `10000` for one cent), the route handler is passing base units to `mppx.charge`. Fix `lib/mpp.ts`:

```ts
// WRONG — passes base units
export const PRICE = process.env.PRICE || '10000'

// RIGHT — passes decimal token amount
export const PRICE = process.env.PRICE || '0.01'
```

The openapi.json route still emits base units (per the MPP discovery spec), but converts them from the same `PRICE` decimal — never store base units in env directly.

### Build fails with `Cannot find module 'mppx/nextjs'`

The `/nextjs` subpath was never published. Imports must come from `mppx/server`. Look at `lib/mpp.ts` — the first line should be `import { Mppx, tempo } from 'mppx/server'`.

### Build fails complaining about session signing

`tempo()` (the shorthand) registers both `charge` and `session` intents, and `session` requires a server-side signer that doesn't exist in this skill. In `lib/mpp.ts`, change `tempo({ ... })` to `tempo.charge({ ... })`.

### Vercel deploy fails with «Function Runtimes must have a valid version»

`vercel.json` contains a `runtime` field (e.g. `"runtime": "nodejs20.x"`). Remove it — only `maxDuration` should remain.

### `npm install` won't run, error mentions «name can no longer contain capital letters»

Project folder contains uppercase letters. Scaffold into a lowercase sibling and move files (see B1).

### Generated secret is 63 characters instead of 64

`randomBytes(32).toString('hex')` occasionally drops the leading zero. Regenerate in a loop until length is 64 (see B3).

### Phase C check 2 returns 200 instead of 402

The payment middleware isn't engaging. Most likely causes, in order of probability:
- `.env.local` missing or not read — confirm `MPP_RECIPIENT` and `MPP_SECRET_KEY` are both there
- `lib/mpp.ts` imports from `mppx/nextjs` (see above)
- Used `tempo()` instead of `tempo.charge()` (see above)
- The handler returned before the charge ran — verify the validation block is before the charge call

### Phase C check 3 prints `nested offers[]`

`app/openapi.json/route.ts` is emitting the wrong shape. The fix is straightforward — replace the `offers: [...]` array with a flat object containing the same fields. MPPScan will not register a service whose discovery doc uses `offers[]`.

### Production endpoint returns 500 after deploy

Almost always env vars. Verify both are present in production scope:

```bash
vercel env ls production | grep -E 'MPP_RECIPIENT|MPP_SECRET_KEY'
```

If either is missing, add via `vercel env add NAME production preview` (with the value piped via stdin). Then `vercel --prod` to pick them up.

### `mppx account create` fails with «Unsupported platform: win32»

The mppx CLI's wallet-management subcommand doesn't support Windows. For real payment testing on Windows, write a short script using `mppx/client` with `privateKeyToAccount` directly and run it via `tsx`.

### Top-level `await` syntax error in a one-off script

TypeScript without ESM module config doesn't allow top-level await. Wrap the script body in `async function main() { ... } main()`.

---

End of skill. Final output line: `MPP_BUILD_DONE`.
