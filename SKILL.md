# Tempo MPP Builder

Sets up a paid HTTP service on the Tempo blockchain end-to-end: scaffolds a Next.js project, wires up MPP payment middleware, deploys to Vercel with a custom domain, and registers on MPPScan so AI agents can discover it.

Invoke this skill with:

```
Скачай https://raw.githubusercontent.com/DontFoldBB/tempo-mpp-skill/main/SKILL.md и собери мне MPP-сервис на Tempo
```

The assistant operates as a **build orchestrator**, not a tutor — it makes decisions, runs commands, and asks the user only when it actually needs input. All user-facing messages are in Russian.

---

## Operating principles

The orchestrator follows these throughout:

- **Russian for the human, English in code.** Comments, variables, error strings — English. Anything addressed to the user — Russian. Technical nouns (Vercel, env, MPP, USDC.e, RPC, bridge) stay as they are.
- **Talk in completed states, not commentary.** After finishing a phase, the orchestrator says what now exists, not what it just did. Example: «Проект готов, зависимости установлены» — not «Я запустил create-next-app и потом npm install».
- **One question at a time.** If multiple inputs are needed, ask sequentially. No bulk forms.
- **Quote real values back to the user.** When confirming an address, price, or path — show the exact string the orchestrator captured, so the user can spot a typo.
- **Defer all spending decisions.** No `vercel --prod`, no actual mppx CLI payments, no DNS changes — without an explicit «да / делай / поехали» from the user in the immediately preceding turn.
- **On error, switch to diagnostic mode.** If a command fails, the next message describes what failed in one or two lines and asks how to proceed (retry, skip, abort) — it does NOT silently rerun.

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
PRICE_UNITS=${PRICE_UNITS:-}
SECRET=${SECRET:-}
PROD_URL=${PROD_URL:-}
DOMAIN=${DOMAIN:-}
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

Then ask the user:

> Что будем делать?
>
> A. Token Info ($0.01) — метаданные TIP-20 токена
> B. Price Feed ($0.01) — цены всех стейблов через CoinGecko
> C. Gas Estimator ($0.005) — оценка газа на Tempo
> D. Activity Score ($0.01) — оценка активности кошелька
> E. Random Stable Fact ($0.005) — простейший вариант для первого захода
> F. Своя идея — опиши что принимает на вход и что возвращает

Map free-form answers loosely (e.g. «цены» → B, «гайз» → C). If F, ask follow-up: «Что принимает на вход и что возвращает?» — then pick the template closest in spirit and adapt.

Capture into the session file:

- `SLUG` — short kebab-case identifier (e.g. `tempo-token-info`)
- `IDEA_KEY` — single letter A–F so later steps know which template to use
- `PRICE_UNITS` — in base units of USDC.e (6 decimals): `10000` = $0.01, `5000` = $0.005, `20000` = $0.02

Once captured, confirm to the user in one line what's queued up.

## A3. Recipient address

Ask:

> Нужен Tempo-адрес который будет получать платежи. Не используй фарминговый кошелёк — заведи отдельный.
>
> 1) wallet.tempo.xyz → создать через passkey
> 2) Скопируй адрес сверху (0x... 42 символа)
> 3) Пришли мне сюда

Validate the answer with `^0x[a-fA-F0-9]{40}$`. If invalid, point out what's wrong (length, prefix, non-hex char) and ask again. Save lowercase variant as `RECIPIENT`.

Confirm by echoing the address back.

## A4. Project location

Ask:

> Куда положить проект? Полный путь к новой папке.

Don't create anything yet — first check for capital letters in the final path segment. npm's `create-next-app` rejects them as project name. If found, tell the user the workaround:

> В имени папки есть заглавные буквы. npm их не любит. Создам проект в `<lowercase-variant>-build/` рядом, потом перенесу файлы куда сказал.

Save `PROJECT_DIR` (the user's chosen path, where files will live after the move) and `SCAFFOLD_DIR` (lowercase path used for `create-next-app`). If no capitals, both point to the same place.

---

# Phase B — Local build

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

USDC.e amounts use 6 decimals, so prices are expressed in base units. The MPP secret must be exactly 64 hex characters. `randomBytes(32).toString('hex')` occasionally returns 63 chars when the leading byte is zero — regenerate until length matches.

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
# Set these after attaching a custom domain
# NEXT_PUBLIC_BASE_URL=https://your-domain.xyz
# MPP_REALM=your-domain.xyz
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

export const PRICE_UNITS = process.env.PRICE_UNITS || '10000'
export const PRICE_DISPLAY = (Number(PRICE_UNITS) / 1_000_000).toFixed(2)
```

### `lib/cache.ts`

```ts
import { LRUCache } from 'lru-cache'

export const responseCache = new LRUCache<string, object>({ max: 1000, ttl: 60_000 })
```

### `app/api/service/route.ts`

Manual 402 flow (because we're on `mppx/server`, not the higher-level Next adapter):

```ts
import { mppx, PRICE_UNITS } from '@/lib/mpp'
import { responseCache } from '@/lib/cache'

export const runtime = 'nodejs'
export const dynamic = 'force-dynamic'

export async function GET(request: Request): Promise<Response> {
  // Validate inputs BEFORE charging — we never want to take money for a malformed request
  const url = new URL(request.url)
  const input = url.searchParams.get('q')
  if (!input) {
    return Response.json({ error: 'missing_parameter' }, { status: 400 })
  }

  // Run payment middleware
  const charge = mppx.charge({ amount: PRICE_UNITS })
  const result = await charge(request)
  if (result.status === 402) return result.challenge

  // Cache hit
  const cacheKey = input.toLowerCase()
  const cached = responseCache.get(cacheKey)
  if (cached) {
    return result.withReceipt(
      Response.json({ ...cached, cached: true })
    )
  }

  // Cache miss — do the work
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

MPPScan parses this file to register the service. Two requirements:
1. `x-payment-info` must be a flat object — `{ intent, method, amount, currency, description }`. The nested `offers: [...]` form gets rejected.
2. `servers[0].url` should be the canonical public URL. Use `NEXT_PUBLIC_BASE_URL` if set (after domain attached), otherwise fall back through Vercel's production URL and finally to localhost.

```ts
export const runtime = 'nodejs'

export async function GET(_req?: Request): Promise<Response> {
  const baseUrl =
    process.env.NEXT_PUBLIC_BASE_URL ||
    (process.env.VERCEL_PROJECT_PRODUCTION_URL && `https://${process.env.VERCEL_PROJECT_PRODUCTION_URL}`) ||
    (process.env.VERCEL_URL && `https://${process.env.VERCEL_URL}`) ||
    'http://localhost:3000'

  const priceUnits = process.env.PRICE_UNITS || '10000'
  const priceDisplay = (Number(priceUnits) / 1_000_000).toFixed(2)

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
            amount: priceUnits,
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
        npx mppx 'https://YOUR_DOMAIN/api/service?q=example'
      </pre>

      <h3>Discovery (free)</h3>
      <pre style={{ background: '#050505', padding: '1rem', border: '1px solid #2a2a2a', overflowX: 'auto' }}>
        curl 'https://YOUR_DOMAIN/openapi.json'
      </pre>
    </main>
  )
}
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
# 1) Missing param → 400
curl -s -o /dev/null -w 'missing_param: HTTP %{http_code}\n' \
  'http://localhost:3000/api/service'

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
- Check 1 prints `HTTP 400`
- Check 2 prints an `HTTP/... 402` line followed by `WWW-Authenticate: Payment ...`
- Check 3 prints `flat (good)` and shows the amount/method
- Check 4 prints `HTTP 200`

If anything doesn't match, see failure modes — fix the issue and rerun this phase. **Don't proceed to deploy until all four pass.**

Stop the dev server:

```bash
kill $DEV_PID 2>/dev/null; wait $DEV_PID 2>/dev/null
```

Confirm to the user that local checks passed and ask whether to deploy.

---

# Phase D — Deploy

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

# Phase E — Custom domain

If the user is OK staying on `*.vercel.app`, skip this phase entirely. MPPScan accepts vercel.app URLs but the registry quality scoring favors custom domains.

Ask which path:

> Привязываем свой домен?
>
> 1) Уже есть — назови
> 2) Купить (porkbun.com или cloudflare.com — оба без верификации, $1–2 за `.xyz`)
> 3) Пропустить

For option 2, point to those two registrars and wait for the user to come back with a domain.

When the user has a domain, walk through attachment without doing any DNS work yourself — the user has to set records on their registrar:

1. Vercel Dashboard → project → Settings → Domains → Add → enter the domain
2. Vercel shows the exact DNS records to add (typically an A record for the apex and a CNAME for www)
3. On the registrar, add those records. **For Cloudflare specifically: proxy must be OFF** (grey cloud, not orange) — otherwise SSL issuance fails
4. Wait 2–10 minutes for DNS propagation
5. Vercel issues the SSL certificate automatically once DNS resolves correctly

Wait for the user to confirm the domain is live, then verify:

```bash
nslookup "$DOMAIN" | tail -5
curl -sI "https://$DOMAIN/" | head -3
```

Then add two more env vars and redeploy — without these, `openapi.json` keeps reporting the vercel.app URL, and MPPScan won't accept the registration. Same CLI flow as in D3, no dashboard needed:

```bash
cd "$PROJECT_DIR"

# Remove if already set (from previous skill run on same project)
vercel env rm NEXT_PUBLIC_BASE_URL production --yes 2>/dev/null
vercel env rm NEXT_PUBLIC_BASE_URL preview --yes 2>/dev/null
vercel env rm MPP_REALM production --yes 2>/dev/null
vercel env rm MPP_REALM preview --yes 2>/dev/null

# Add fresh values
printf '%s' "https://$DOMAIN" | vercel env add NEXT_PUBLIC_BASE_URL production preview 2>&1 | tail -2
printf '%s' "$DOMAIN"         | vercel env add MPP_REALM production preview 2>&1 | tail -2
```

Then redeploy so the new env values take effect:

```bash
vercel --prod 2>&1 | tail -5
```

Verify the canonical URL is now used:

```bash
curl -s "https://$DOMAIN/openapi.json" | node -e "
  let s=''; process.stdin.on('data',d=>s+=d).on('end',()=>{
    const d=JSON.parse(s);
    console.log('servers[0].url =', d.servers?.[0]?.url);
  });
"
```

Should print `https://<DOMAIN>`.

---

# Phase F — Register on MPPScan

> Финал: регистрируем сервис чтоб агенты могли найти.
>
> 1) Открой mppscan.com/register
> 2) Service URL = `https://<DOMAIN>` (или vercel.app если пропустил Phase E)
> 3) Жми Register — MPPScan сам прочитает `/openapi.json`
> 4) Получишь URL вида `mppscan.com/server/<hash>` — пришли сюда

After receiving the link, optionally verify it loads and then print the build summary:

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MPP-сервис готов

Имя:           <SLUG>
Цена:          $<PRICE_DISPLAY> USDC.e
Live URL:      https://<DOMAIN>
API endpoint:  https://<DOMAIN>/api/service
Discovery:     https://<DOMAIN>/openapi.json
MPPScan:       <USER_PROVIDED_LINK>
Получатель:    <RECIPIENT>
Проект:        <PROJECT_DIR>

Полезное:
  cd <PROJECT_DIR>
  vercel --prod                                  # передеплоить после правок
  npm run dev                                    # локально на :3000
  npx mppx 'https://<DOMAIN>/api/service?q=...'  # реальный платный вызов
```

Clean up:

```bash
rm -f "$HOME/.mpp-build.session"
```

End the build with a single line: `MPP_BUILD_DONE`. Downstream tooling looks for this exact string.

---

# Failure modes

Common ways things break, grouped by symptom.

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

### `openapi.json` keeps reporting the vercel.app URL after attaching a domain

`NEXT_PUBLIC_BASE_URL` not set in production env vars, or set but not deployed yet. Verify with `vercel env ls production | grep NEXT_PUBLIC_BASE_URL`. If missing, add via `vercel env add NEXT_PUBLIC_BASE_URL production preview` (pipe `https://your-domain` via stdin), then `vercel --prod`.

### `mppx account create` fails with «Unsupported platform: win32»

The mppx CLI's wallet-management subcommand doesn't support Windows. For real payment testing on Windows, write a short script using `mppx/client` with `privateKeyToAccount` directly and run it via `tsx`.

### Top-level `await` syntax error in a one-off script

TypeScript without ESM module config doesn't allow top-level await. Wrap the script body in `async function main() { ... } main()`.

---

End of skill. Final output line: `MPP_BUILD_DONE`.
