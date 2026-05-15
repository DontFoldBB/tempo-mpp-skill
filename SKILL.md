# Tempo MPP Service — Setup Skill

**Цель:** за одну команду пользователь получает живой платный MPP-сервис на Tempo с собственным доменом, задеплоенный на Vercel и зарегистрированный на MPPScan.

**Как вызывается:**
```
Скачай https://raw.githubusercontent.com/DontFoldBB/tempo-mpp-skill/main/SKILL.md и собери мне свой MPP-сервис на Tempo
```

Claude Code (далее «агент») следует шагам **строго по порядку**. В каждом шаге:
- какая команда запускается,
- что сказать пользователю (всегда **на русском**),
- когда остановиться и ждать ответа,
- как проверить результат, прежде чем идти дальше.

---

## Правила общения с пользователем (читать до всех шагов)

1. **Язык — только русский.** Все сообщения, подсказки, ошибки — по-русски. Технические термины (passkey, env, deploy, USDC.e, bridge, MPP, Vercel) оставляй как есть.
2. **После каждого шага печатай статус-строку** в формате:
   ```
   ✅ Шаг N: <что сделано> — <ключевое значение, если есть>
   ```
   Пример: `✅ Шаг 1: проект создан в C:\projects\my-mpp-service`.
3. **Перед интерактивными шагами** (выбор идеи, ввод адреса, деплой) — коротко объясни, что сейчас будет, и чего ждёшь от пользователя.
4. **Не дублируй команды**, которые уже выполнил — показывай пользователю только результат и следующий шаг.
5. **На ошибке** — сразу перейди в секцию «Шпаргалка по ошибкам» в конце файла, покажи пользователю матч + как чинить. Не повторяй проблемную команду молча.
6. **Никаких трат без подтверждения** — реальные платежи делаем только после явного «да» от пользователя.

---

## Техническое замечание: state между шагами

Каждый Bash-вызов — **новый shell**; переменные не переживают. Сохраняй значения в `~/.tempo-mpp-state.env` (chmod 600 на Linux/Mac, на Windows просто файл в `$HOME`) и читай в начале следующего шага.

**Важно:** используй паттерн *«прочитать → обновить → перезаписать»* (`>`), а не append (`>>`):

```bash
STATE="$HOME/.tempo-mpp-state.env"
touch "$STATE"
. "$STATE" 2>/dev/null || true

# ...обновляешь нужные переменные...
PROJECT_DIR="..."
RECIPIENT_ADDRESS="0x..."
SERVICE_NAME="..."
DOMAIN="..."

# атомарная перезапись:
{
  echo "PROJECT_DIR=${PROJECT_DIR:-}"
  echo "RECIPIENT_ADDRESS=${RECIPIENT_ADDRESS:-}"
  echo "SERVICE_NAME=${SERVICE_NAME:-}"
  echo "SERVICE_IDEA=${SERVICE_IDEA:-}"
  echo "PRICE_BASE_UNITS=${PRICE_BASE_UNITS:-10000}"
  echo "DOMAIN=${DOMAIN:-}"
  echo "VERCEL_URL=${VERCEL_URL:-}"
  echo "MPP_SECRET_KEY=${MPP_SECRET_KEY:-}"
} > "$STATE"
```

После полной установки можно удалить (`rm "$HOME/.tempo-mpp-state.env"`).

---

## Шаг 0 — Preflight

Проверь зависимости параллельно:

```bash
command -v node    >/dev/null && node --version  || echo MISSING:node
command -v npm     >/dev/null && npm --version   || echo MISSING:npm
command -v git     >/dev/null && git --version   || echo MISSING:git
command -v curl    >/dev/null && echo OK:curl    || echo MISSING:curl
```

Требования: Node.js ≥ 20, npm, git, curl.

Если `node` < 20 или отсутствует — останови пользователя:
> ❌ Нужен Node.js 20 или выше. Установи с nodejs.org и перезапусти терминал.

**Статус:** `✅ Шаг 0: окружение готово (Node X, npm Y, git, curl)`

---

## Шаг 1 — Выбор идеи сервиса

Скачай шаблоны идей и покажи пользователю:

```bash
curl -fsSL "https://raw.githubusercontent.com/DontFoldBB/tempo-mpp-skill/main/docs/idea-templates.md" -o /tmp/idea-templates.md
```

Прочитай файл и **спроси пользователя** (через AskUserQuestion если есть):

> 🎯 Какой сервис будем делать?
>
> 1. **Token Info** ($0.01) — инфа по любому TIP-20 токену на Tempo: total supply, holders count, decimals
> 2. **Price Feed** ($0.01) — текущие цены всех Tempo-стейблов через CoinGecko
> 3. **Gas Estimator** ($0.005) — оценка стоимости газа на Tempo для разных типов транзакций
> 4. **Activity Score** ($0.01) — скор активности кошелька от 0 до 100
> 5. **Random Stable Fact** ($0.005) — случайный факт про стейблкоины (самый простой)
> 6. **Свой вариант** — опиши идею в свободной форме

**Fallback:** если пользователь ответил не числом, пытайся сопоставить смысл (`/токен.*инфо/i` → 1, `/цен/i` → 2, `/газ/i` → 3, `/актив/i` → 4, `/факт/i` → 5, иначе уточни).

Если **6 (свой вариант)** — попроси описать в 1-2 предложениях: что принимает на вход, что возвращает. Затем создай минимальный шаблон по аналогии с Token Info.

Сохрани в state:
- `SERVICE_NAME` (короткое slug, например `tempo-token-info`)
- `SERVICE_IDEA` (одно предложение что делает)
- `PRICE_BASE_UNITS` (10000 = $0.01, 5000 = $0.005, USDC.e имеет 6 decimals)

**Статус:** `✅ Шаг 1: выбран сервис «<SERVICE_NAME>», цена <PRICE> USDC.e`

---

## Шаг 2 — Получение Tempo адреса для платежей

**Сообщение пользователю:**

> 💼 Нужен Tempo-кошелёк который будет получать платежи. **Не используй фарминговый кошелёк** — создай отдельный.
>
> 1. Открой https://wallet.tempo.xyz
> 2. Создай кошелёк через passkey (Touch ID / Windows Hello / YubiKey)
> 3. Скопируй адрес из верхней части (формат `0x...` 42 символа)
> 4. Вставь адрес в этот чат

Дождись ответа. Провалидируй регулярка `^0x[a-fA-F0-9]{40}$`:

```bash
echo "$USER_INPUT" | grep -qE '^0x[a-fA-F0-9]{40}$' && echo "ok" || echo "invalid"
```

Если `invalid` — попроси ещё раз. Если `ok` — сохрани в state как `RECIPIENT_ADDRESS` и нижний регистр.

**Статус:** `✅ Шаг 2: адрес получения платежей <RECIPIENT_ADDRESS>`

---

## Шаг 3 — Создание Next.js проекта

**Сообщение пользователю:**

> 📁 Сейчас создам Next.js проект. Где разместить папку?
>
> Подскажи путь, например:
> - macOS/Linux: `/Users/me/projects/my-mpp-service`
> - Windows: `F:\projects\my-mpp-service`

Дождись ответа. Сохрани в `PROJECT_DIR`. Если путь содержит заглавные буквы — предупреди:

> ⚠️ В пути есть заглавные буквы. npm может ругаться при scaffold (нельзя использовать заглавные в названии). Я создам проект во временной папке рядом и потом перенесу файлы, всё ок.

Создай родительскую директорию если её нет:

```bash
mkdir -p "$(dirname "$PROJECT_DIR")"
```

Если путь содержит заглавные буквы:

```bash
PARENT="$(dirname "$PROJECT_DIR")"
TMP_NAME="$(basename "$PROJECT_DIR" | tr '[:upper:]' '[:lower:]')-tmp"
cd "$PARENT" && npx create-next-app@latest "$TMP_NAME" --typescript --eslint --tailwind --app --no-src-dir --import-alias "@/*" --use-npm --yes
# Перенести файлы в нужное место
mkdir -p "$PROJECT_DIR"
cp -r "$PARENT/$TMP_NAME/." "$PROJECT_DIR/"
rm -rf "$PARENT/$TMP_NAME"
```

Иначе обычный scaffold:

```bash
mkdir -p "$PROJECT_DIR"
cd "$PROJECT_DIR" && npx create-next-app@latest . --typescript --eslint --tailwind --app --no-src-dir --import-alias "@/*" --use-npm --yes
```

Установи зависимости:

```bash
cd "$PROJECT_DIR" && npm install mppx viem lru-cache
npm install -D tsx
```

**Статус:** `✅ Шаг 3: Next.js проект создан в <PROJECT_DIR>`

---

## Шаг 4 — Генерация секрета и конфига

Сгенерируй `MPP_SECRET_KEY` (ровно 64 hex символа):

```bash
MPP_SECRET_KEY=$(node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
# Проверь длину
[ ${#MPP_SECRET_KEY} -eq 64 ] || { echo "BAD_KEY_LENGTH"; exit 1; }
```

**Если длина не 64** — перегенерируй (это известный баг, иногда выдаёт 63 символа из-за leading zero). Повторяй до получения 64.

Создай `.env.local`:

```bash
cat > "$PROJECT_DIR/.env.local" << EOF
MPP_RECIPIENT=$RECIPIENT_ADDRESS
MPP_SECRET_KEY=$MPP_SECRET_KEY
EOF
```

Создай `.env.example` (без значений):

```bash
cat > "$PROJECT_DIR/.env.example" << 'EOF'
MPP_RECIPIENT=
MPP_SECRET_KEY=
# Optional: для канонического URL после привязки своего домена
# NEXT_PUBLIC_BASE_URL=https://your-domain.xyz
# MPP_REALM=your-domain.xyz
EOF
```

Убедись что `.env.local` в `.gitignore`:

```bash
grep -q "^\.env\*\.local$\|^\.env\.local$" "$PROJECT_DIR/.gitignore" || echo ".env*.local" >> "$PROJECT_DIR/.gitignore"
```

Сохрани секрет в state (понадобится при деплое на Vercel).

**Статус:** `✅ Шаг 4: секрет сгенерирован, .env.local создан`

---

## Шаг 5 — Написание кода сервиса

Прочитай документацию MPP (получи актуальные паттерны):

```bash
curl -fsSL https://mpp.dev/payment-methods/tempo -o /tmp/mpp-tempo.html
curl -fsSL https://mpp.dev/quickstart/server -o /tmp/mpp-server.html
```

Затем напиши файлы. **Критические нюансы из реального опыта:**

1. `mppx/nextjs` НЕ существует в текущей версии — используй `mppx/server` с ручным flow
2. `tempo()` shorthand регистрирует session intent (требует server signing) — используй `tempo.charge()`
3. `vercel.json` НЕ должен содержать `"runtime": "nodejs20.x"` — оставь только `maxDuration`
4. `x-payment-info` в `openapi.json` должен быть **плоским объектом**, не `offers:[]` (для MPPScan)

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

```ts
import { Mppx } from 'mppx/server'
import { tempo } from 'mppx/server'
import { USDC_E_ADDRESS } from './chain'

const RECIPIENT = process.env.MPP_RECIPIENT
if (!RECIPIENT || !/^0x[a-fA-F0-9]{40}$/.test(RECIPIENT)) {
  throw new Error('MPP_RECIPIENT env var is required')
}

export const mppx = Mppx.create({
  methods: [
    tempo.charge({
      currency: USDC_E_ADDRESS,
      recipient: RECIPIENT as `0x${string}`,
    }),
  ],
})

export const PRICE_BASE_UNITS = process.env.PRICE_BASE_UNITS || '10000' // $0.01 in 6-decimal units
export const PRICE_DISPLAY = (Number(PRICE_BASE_UNITS) / 1_000_000).toFixed(2)
```

### `lib/cache.ts`

```ts
import { LRUCache } from 'lru-cache'

export const responseCache = new LRUCache<string, object>({ max: 1000, ttl: 60_000 })
```

### `app/api/service/route.ts`

Базовый шаблон с ручным 402 flow:

```ts
import { mppx, PRICE_BASE_UNITS } from '@/lib/mpp'
import { responseCache } from '@/lib/cache'

export const runtime = 'nodejs'
export const dynamic = 'force-dynamic'

export async function GET(request: Request): Promise<Response> {
  // 1. Валидация входных параметров ДО оплаты
  const url = new URL(request.url)
  const input = url.searchParams.get('q') // или 'address', 'token', и т.д. — зависит от идеи
  if (!input) {
    return Response.json({ error: 'missing_parameter' }, { status: 400 })
  }

  // 2. Платная зона
  const charge = mppx.charge({ amount: PRICE_BASE_UNITS })
  const result = await charge(request)
  if (result.status === 402) return result.challenge

  // 3. Кэш
  const cacheKey = input.toLowerCase()
  const cached = responseCache.get(cacheKey)
  if (cached) {
    return result.withReceipt(Response.json({ ...cached, cached: true }))
  }

  // 4. Бизнес-логика (определяется идеей сервиса — см. idea-templates.md)
  try {
    const payload = await buildResponse(input)
    responseCache.set(cacheKey, payload)
    return result.withReceipt(Response.json(payload, { headers: { 'Cache-Control': 'private, no-store' } }))
  } catch (err) {
    console.error('service_error', err)
    return result.withReceipt(Response.json({ error: 'internal_error' }, { status: 500 }))
  }
}

async function buildResponse(input: string): Promise<object> {
  // ЗАПОЛНИТЬ ПОД ИДЕЮ — см. docs/idea-templates.md
  return { input, result: 'TODO' }
}
```

Для функции `buildResponse` подставь код из `docs/idea-templates.md` под выбранную идею.

### `app/openapi.json/route.ts`

```ts
export const runtime = 'nodejs'

export async function GET(_req?: Request): Promise<Response> {
  const baseUrl =
    process.env.NEXT_PUBLIC_BASE_URL ||
    (process.env.VERCEL_PROJECT_PRODUCTION_URL ? `https://${process.env.VERCEL_PROJECT_PRODUCTION_URL}` : null) ||
    (process.env.VERCEL_URL ? `https://${process.env.VERCEL_URL}` : 'http://localhost:3000')

  const doc = {
    openapi: '3.1.0',
    info: {
      title: process.env.SERVICE_NAME || 'My MPP Service',
      version: '1.0.0',
      'x-guidance': 'Use npx mppx <url> to call paid endpoints. Free endpoints work with curl.',
    },
    servers: [{ url: baseUrl }],
    'x-service-info': {
      categories: ['data'],
      docs: {
        homepage: baseUrl,
        apiReference: `${baseUrl}/`,
      },
    },
    paths: {
      '/api/service': {
        get: {
          summary: 'Main paid endpoint',
          parameters: [
            { name: 'q', in: 'query', required: true, schema: { type: 'string' }, example: 'example-value' },
          ],
          'x-payment-info': {
            intent: 'charge',
            method: 'tempo',
            amount: process.env.PRICE_BASE_UNITS || '10000',
            currency: '0x20c000000000000000000000b9537d11c60e8b50',
            description: `$${((Number(process.env.PRICE_BASE_UNITS || '10000')) / 1_000_000).toFixed(2)} per request`,
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

### `app/page.tsx` (минимальный landing, для агентов)

```tsx
export default function Home() {
  return (
    <main style={{ fontFamily: 'ui-monospace, monospace', padding: '2rem', maxWidth: '720px', margin: '0 auto', color: '#e5e5e5', background: '#0a0a0a', minHeight: '100vh' }}>
      <h1 style={{ color: '#5dcaa5' }}>{process.env.SERVICE_NAME || 'MPP Service'}</h1>
      <p style={{ color: '#888' }}>Paid API on Tempo blockchain via MPP.</p>
      <h3 style={{ marginTop: '2rem' }}>Usage</h3>
      <pre style={{ background: '#050505', padding: '1rem', border: '1px solid #2a2a2a' }}>
        npx mppx 'https://your-url/api/service?q=example'
      </pre>
      <h3 style={{ marginTop: '2rem' }}>Discovery</h3>
      <pre style={{ background: '#050505', padding: '1rem', border: '1px solid #2a2a2a' }}>
        curl 'https://your-url/openapi.json'
      </pre>
    </main>
  )
}
```

### `vercel.json` (БЕЗ `runtime` поля!)

```json
{
  "functions": {
    "app/api/service/route.ts": { "maxDuration": 30 },
    "app/openapi.json/route.ts": { "maxDuration": 10 }
  }
}
```

### Сборка

```bash
cd "$PROJECT_DIR"
npm run build 2>&1 | tail -30
```

Если есть ошибки — иди в Шпаргалку по ошибкам.

**Сообщение пользователю:** компактный отчёт что создано (без вставки полного кода).

**Статус:** `✅ Шаг 5: код сервиса написан, npm run build прошёл`

---

## Шаг 6 — Git инициализация и первый коммит

```bash
cd "$PROJECT_DIR"
git init 2>/dev/null || true
git add -A
git commit -m "scaffold: $SERVICE_NAME MPP service"
```

**Статус:** `✅ Шаг 6: git repo инициализирован`

---

## Шаг 7 — Деплой на Vercel

Проверь установлен ли Vercel CLI:

```bash
command -v vercel >/dev/null && vercel --version || echo MISSING:vercel
```

Если отсутствует:

> 📦 Устанавливаю Vercel CLI...

```bash
npm install -g vercel
```

**Сообщение пользователю:**

> 🚀 Сейчас задеплою на Vercel. Если не залогинен, тебя попросят авторизоваться через браузер. Зарегистрируйся на vercel.com (проще через GitHub) если ещё нет аккаунта.
>
> Я запущу `vercel` без флагов — это создаст новый проект с дефолтными настройками. После первого деплоя добавим env vars и сделаем production deploy.

```bash
cd "$PROJECT_DIR" && vercel
```

После того как пользователь пройдёт интерактив — извлеки URL из вывода (`https://[project]-[hash]-[user].vercel.app`).

Сохрани в state как `VERCEL_PREVIEW_URL`.

**Сообщение пользователю:**

> 🔑 Теперь добавим env vars. Открой dashboard.vercel.com → твой проект → **Settings** → **Environment Variables**.
>
> Добавь две переменные для **Production** и **Preview**:
>
> ```
> MPP_RECIPIENT = $RECIPIENT_ADDRESS
> MPP_SECRET_KEY = $MPP_SECRET_KEY
> ```
>
> Когда добавишь — напиши **готово**.

Дождись `готово`. Затем production deploy:

```bash
cd "$PROJECT_DIR" && vercel --prod 2>&1 | tail -10
```

Извлеки production URL. Сохрани как `VERCEL_PROD_URL`.

**Статус:** `✅ Шаг 7: задеплоено на <VERCEL_PROD_URL>`

---

## Шаг 8 — Проверка работоспособности

Проверь что сервис отвечает правильно:

```bash
# 1. 400 на некорректный запрос
RESP=$(curl -s -o /dev/null -w '%{http_code}' "$VERCEL_PROD_URL/api/service")
echo "Test 1 (missing param): HTTP $RESP" # ожидается 400

# 2. 402 на корректный без оплаты
RESP=$(curl -s -i "$VERCEL_PROD_URL/api/service?q=test" | head -20)
echo "$RESP" | grep -i "www-authenticate.*payment" && echo "Test 2: PASS" || echo "Test 2: FAIL"

# 3. openapi.json
RESP=$(curl -s "$VERCEL_PROD_URL/openapi.json")
echo "$RESP" | grep -q '"openapi"' && echo "Test 3 (openapi): PASS" || echo "Test 3: FAIL"

# 4. Landing
RESP=$(curl -s -o /dev/null -w '%{http_code}' "$VERCEL_PROD_URL/")
echo "Test 4 (landing): HTTP $RESP" # ожидается 200
```

Все тесты должны пройти. Если что-то FAIL — иди в Шпаргалку.

**Сообщение пользователю:**

> ✅ Сервис задеплоен и отвечает правильно:
> - `GET /api/service?q=...` без оплаты → 402 (нужна оплата)
> - `GET /openapi.json` → 200 (discovery работает)
> - `GET /` → 200 (landing работает)
>
> Реальный тест с платежом сможем сделать после Шага 9 (домен + регистрация).

**Статус:** `✅ Шаг 8: сервис проверен, все эндпоинты отвечают`

---

## Шаг 9 — Привязка своего домена

**Спроси пользователя:**

> 🌐 Хочешь привязать свой домен? Это нужно для регистрации на MPPScan и для солидного вида (vercel.app выглядит как «сделано на коленке»).
>
> 1. Да, у меня уже есть домен — напишу название
> 2. Да, помоги купить (можно за $1-2 на porkbun.com)
> 3. Нет, оставлю vercel.app

Логика:

| Ответ | Действия |
|---|---|
| 1 | Перейти к подключению (см. ниже) |
| 2 | Показать инструкцию покупки на Porkbun или Cloudflare, потом подключение |
| 3 | Пропустить шаг, сразу к Шагу 10 |

### Покупка домена (вариант 2)

> 💡 Самые дешёвые варианты без верификации:
> - **porkbun.com** — `.xyz` от $1 в первый год
> - **Cloudflare Registrar** (cloudflare.com) — продают по себестоимости
>
> Купи домен, потом возвращайся и напиши его в чат.

### Подключение (варианты 1 и 2)

Дождись названия домена. Сохрани в `DOMAIN`.

> 🔗 Подключаю домен к Vercel:
>
> 1. Открой dashboard.vercel.com → твой проект → **Settings** → **Domains**
> 2. Нажми **Add**, введи `$DOMAIN`
> 3. Vercel покажет какие DNS записи добавить (обычно A запись с IP типа `216.198.79.1` и CNAME для www)
> 4. Открой панель управления своего регистратора (где купил домен)
> 5. Добавь эти DNS записи. **Важно:** на Cloudflare обязательно выключи proxy (серое облачко, не оранжевое), иначе SSL не выпустится.
> 6. Когда добавишь — напиши **готово**, проверю.

Дождись `готово`. Проверь DNS и SSL:

```bash
# DNS должен резолвиться
nslookup "$DOMAIN" 2>&1 | tail -5

# Сайт должен открываться
curl -sI "https://$DOMAIN/" | head -5
```

Если 404 / неправильный IP — попроси пользователя нажать **Refresh** в Vercel Domains и подождать ещё 2-5 минут.

После того как домен работает, **обнови env vars в Vercel**:

> 🔧 Финальный штрих — добавь ещё две env vars в Vercel (Production + Preview):
>
> ```
> NEXT_PUBLIC_BASE_URL = https://$DOMAIN
> MPP_REALM = $DOMAIN
> ```
>
> Это нужно чтобы `openapi.json` ссылался на твой домен, а не на vercel.app. Без этого MPPScan не примет сервис.
>
> Когда добавишь — напиши **готово**.

После `готово`:

```bash
cd "$PROJECT_DIR" && vercel --prod 2>&1 | tail -5
```

Проверь openapi:

```bash
curl -s "https://$DOMAIN/openapi.json" | grep -o "\"url\":\"[^\"]*\"" | head -1
# должно вывести "url":"https://$DOMAIN"
```

**Статус:** `✅ Шаг 9: домен <DOMAIN> подключен, openapi.json указывает на канонический URL`

---

## Шаг 10 — Регистрация на MPPScan

**Сообщение пользователю:**

> 📋 Финальный шаг — регистрируем сервис на MPPScan, чтобы агенты могли тебя найти.
>
> 1. Открой https://mppscan.com/register
> 2. В поле **Service URL** введи: `https://$DOMAIN` (или твой Vercel URL если без домена)
> 3. Жми **Register**
> 4. MPPScan сам сходит на твой `/openapi.json`, прочитает endpoints и payment info
> 5. После регистрации получишь URL вида `mppscan.com/server/<hash>` — пришли его в чат

Дождись URL. Извлеки hash, проверь что сервер живой:

```bash
curl -s "$USER_URL" | grep -o "<title>[^<]*</title>" | head -1
```

**Сообщение пользователю:**

> 🎉 Готово! Твой сервис в каталоге: `<MPPSCAN_URL>`
>
> Что у тебя есть теперь:
> - 🌐 Сайт: `https://$DOMAIN`
> - 💰 Платный API: `https://$DOMAIN/api/service`
> - 🔍 Discovery: `https://$DOMAIN/openapi.json`
> - 📋 В каталоге MPPScan: `<MPPSCAN_URL>`
>
> Реальный тест платежа:
> ```
> npx mppx 'https://$DOMAIN/api/service?q=example'
> ```
> Запросит USDC.e на Tempo, заплатит, ты получишь деньги на `$RECIPIENT_ADDRESS`.

**Статус:** `✅ Шаг 10: сервис зарегистрирован на MPPScan`

---

## Шаг 11 — Финальная сводка

Покажи итог:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ MPP-сервис на Tempo готов

Имя:          <SERVICE_NAME>
Идея:         <SERVICE_IDEA>
Цена:         $<PRICE_DISPLAY> USDC.e за запрос
Домен:        https://<DOMAIN>
Получатель:   <RECIPIENT_ADDRESS>
MPPScan:      <MPPSCAN_URL>
Папка кода:   <PROJECT_DIR>

Полезные команды:
  cd "<PROJECT_DIR>"                    # вернуться в проект
  vercel --prod                         # передеплоить после изменений
  npm run dev                           # локальный запуск (порт 3000)
  npx mppx 'https://<DOMAIN>/api/service?q=...'   # реальный платный тест
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Удали state file:

```bash
rm -f "$HOME/.tempo-mpp-state.env"
```

**Напечатай буквально** в конце: `MPP service setup: DONE`

---

## Шпаргалка по ошибкам

| Ошибка | Причина | Что делать |
|---|---|---|
| `Cannot find module 'mppx/nextjs'` | Этого экспорта нет в текущей версии mppx | В `lib/mpp.ts` использовать `import { Mppx, tempo } from 'mppx/server'` и ручной 402 flow |
| `Build failed: tempo() needs signer` | `tempo()` shorthand регистрирует session intent (нужен server signing) | Заменить на `tempo.charge({ currency, recipient })` |
| `Vercel build: Function Runtimes must have a valid version` | В `vercel.json` есть `"runtime": "nodejs20.x"` | Убрать поле `runtime`, оставить только `maxDuration` |
| `npm name can no longer contain capital letters` | В пути проекта есть заглавные буквы | Скаффолд во временной папке с lowercase именем, потом перенести файлы |
| `MPP_SECRET_KEY length is 63 not 64` | `randomBytes(32).toString('hex')` иногда теряет leading zero | Перегенерировать пока не получишь ровно 64 символа |
| `MPPScan: invalid openapi` | `x-payment-info` написан как `offers:[...]` | Сделать плоский объект: `{ intent, method, amount, currency, description }` без массива |
| `MPPScan: realm mismatch` | `MPP_REALM` env var не совпадает с доменом | Добавить в Vercel env var `MPP_REALM=$DOMAIN` (без https://) |
| `openapi.json shows vercel.app URL after domain attached` | По умолчанию использует `VERCEL_URL` | Добавить env var `NEXT_PUBLIC_BASE_URL=https://$DOMAIN` и передеплоить |
| `SSL pending forever on custom domain` | Cloudflare proxy включён (оранжевое облачко) | На Cloudflare DNS: переключи в DNS only (серое облачко) |
| `Test 2: FAIL (no 402)` | env vars не подгрузились в production | Проверь что `MPP_RECIPIENT` и `MPP_SECRET_KEY` добавлены в Vercel для Production, а не только Preview |
| `mppx account create: Unsupported platform: win32` | mppx CLI не работает на Windows | Для тестов платежей использовать скрипт с `mppx/client` + `privateKeyToAccount` через tsx |
| Vercel deploy: `git ref does not exist` | Не сделан первый коммит | `cd $PROJECT_DIR && git add -A && git commit -m "init"` |
| Top-level await error in tests | TypeScript не поддерживает top-level await без ESM | Обернуть в `async function main() { ... } main()` |

---

**Конец скилла.** Последняя строка вывода должна быть `MPP service setup: DONE`.
