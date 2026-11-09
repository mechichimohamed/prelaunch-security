# The 20 checks in detail

For each item: **Look for** (what to search), **Bad signs**, **Fix** (per stack), **Gotchas**. Patterns are ripgrep regexes; run them with the Grep tool and exclude `node_modules`, `.next`, `dist`, `build`, `vendor`, `.venv`, lockfiles.

---

## 1. Hide API keys

**Look for**
- Known key shapes: `sk-[A-Za-z0-9_-]{20,}` (OpenAI/Anthropic-style), `sk-ant-`, `sk_live_`, `rk_live_` (Stripe), `AKIA[0-9A-Z]{16}` (AWS), `AIza[0-9A-Za-z_-]{35}` (Google), `ghp_|gho_|github_pat_` (GitHub), `xox[baprs]-` (Slack), `SG\.[A-Za-z0-9_-]{20,}` (SendGrid), `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`, `eyJhbGciOi` (JWTs, incl. Supabase service_role).
- Generic: `(api[_-]?key|secret|token|password|passwd|private[_-]?key)\s*[:=]\s*['"][^'"]{8,}`.
- Client-exposed env prefixes holding secrets: `NEXT_PUBLIC_.*(SECRET|SERVICE|PRIVATE|OPENAI|STRIPE_SECRET)`, same for `VITE_`, `EXPO_PUBLIC_`, `REACT_APP_`, `PUBLIC_` (SvelteKit), `NUXT_PUBLIC_`.
- Third-party AI/payment calls made from the browser (`fetch("https://api.openai.com`, `new OpenAI(` in a client component, `dangerouslyAllowBrowser: true`).
- `.gitignore` covering `.env`, `.env.local`, `.env.*.local`, `*.pem`, `service-account*.json`, `google-services.json` (where secret).
- Mobile: keys in `app.json`, `Info.plist`, `strings.xml`, `.dart` constants.

**Fix**
- Move the call behind a server route / server action / edge function; the client calls your endpoint, the server holds the key in an env var.
- Rename env vars so secrets have no public prefix; add `.env*` (except `.env.example`) to `.gitignore`; create `.env.example` with blank values.
- Next.js: add `import "server-only"` to modules that read secrets.

**Gotchas**: anything with a public prefix or in a mobile bundle is public, full stop. Publishable keys (Stripe `pk_`, Supabase anon, Firebase web config) are fine to ship *if* item 4 is done.

---

## 2. Purge Git secrets

**Look for**
- If installed, run `gitleaks detect --source . -v` or `trufflehog git file://. --only-verified`. Otherwise:
  - `git log -p --all -S "sk_live_"` (repeat per key shape from item 1)
  - `git log --all --diff-filter=A --name-only -- '*.env' '.env*' '*.pem' '*service-account*'`
- Check if the remote is public: `gh repo view --json visibility`.

**Fix (ask first: rewrites history)**
1. **Rotate every leaked key first** at the provider. This is the actual fix.
2. Remove with `git filter-repo --invert-paths --path .env` or `git filter-repo --replace-text replacements.txt` (BFG is an alternative).
3. Force-push all branches and tags, ask collaborators to re-clone, and on GitHub contact support / delete cached PR refs if the repo was public.

**Gotchas**: never run a history rewrite or force-push without explicit approval. Forks and clones keep the old history; that is why rotation is mandatory.

---

## 3. Use public DB key

**Look for**
- Supabase: `SUPABASE_SERVICE_ROLE_KEY`, `service_role`, `createClient(` called with a service key in any file that reaches the client (`"use client"`, `src/components`, `pages/` non-API, mobile code).
- Firebase: `firebase-admin` or service account JSON imported in client code.
- Direct DB URLs (`postgres://`, `mongodb+srv://`) in client code or public env vars.

**Fix**: client uses the anon/publishable key only; the service-role/admin client lives in a server-only module and is used only for jobs that must bypass RLS, always after its own auth check.

---

## 4. Enable RLS

**Look for**
- Supabase/Postgres migrations: every `create table` in an exposed schema (`public`) followed by `alter table ... enable row level security` and at least one `create policy`.
- Policies that are effectively open: `using (true)`, `with check (true)` on insert/update/delete, policies granted `to anon` on private data, `auth.uid() is not null` used where ownership is meant.
- Views and functions: views bypass RLS unless `security_invoker = true` (PG15+); `security definer` functions in `public` need careful checks.
- Firebase: `firestore.rules` / `storage.rules` containing `allow read, write: if true;` or `if request.auth != null` for per-user data.
- Storage buckets marked public that hold private files.

**Fix**
```sql
alter table public.orders enable row level security;
create policy "owner reads" on public.orders for select using (auth.uid() = user_id);
create policy "owner inserts" on public.orders for insert with check (auth.uid() = user_id);
create policy "owner updates" on public.orders for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
```
Firebase: `allow read, write: if request.auth != null && request.auth.uid == resource.data.userId;` (and `request.resource.data` on create).

**Gotchas**: if the project has no migrations in the repo, mark **Manual** and point to Supabase Dashboard → Database → Tables (RLS badge) and the Security Advisor. Write the SQL as a new migration file; do not apply it to a live DB without approval.

---

## 5. Encrypt sensitive data

**Look for**: columns/fields named `ssn`, `national_id`, `dob`, `card`, `iban`, `access_token`, `refresh_token`, `api_key`, `secret`, health data; third-party OAuth tokens stored plaintext; `console.log`/`logger` calls printing `req.body`, tokens or passwords; DB connections without TLS (`sslmode=disable`).

**Fix**: don't store what you don't need (cards → Stripe). Encrypt app-level secrets with AES-256-GCM using a key from env/KMS (Supabase Vault / `pgsodium`, Rails `encrypts`, Django `django-cryptography`, Laravel `encrypted` cast). Scrub logs. Require TLS to the DB.

**Gotchas**: encrypting existing rows is a data migration; ask first.

---

## 6. Enforce server-side auth

**Look for**
- API routes / route handlers / server actions / tRPC procedures / Express routers / Django views that read or write user data with no session check (`getServerSession`, `auth()`, `supabase.auth.getUser()`, `req.user`, `@login_required`, `before_action :authenticate_user!`).
- Auth that exists only in the UI (hidden buttons, client-side redirects, `if (!user) router.push('/login')`).
- Next.js `middleware.ts` treated as the only guard (it can be bypassed / mis-matched); server actions are public endpoints.
- Supabase: `getSession()` trusted on the server (use `getUser()` / `getClaims()`, which verify the JWT).
- Admin routes checking a role from client input.

**Fix**: verify the session at the top of every protected handler and server action, ideally via one shared helper (`requireUser()`), and authorize roles from the server-side session/DB.

---

## 7. Lock record access

**Look for**: handlers that take an `id` from params/body and do `findUnique({ where: { id } })`, `.eq('id', id)`, `Model.objects.get(pk=id)`, `findById(id)` without also filtering by `userId`/`orgId`. Sequential integer IDs exposed in URLs.

**Fix**: scope every query: `where: { id, userId: session.user.id }`; return 404 (not 403) when not found for this user. Multi-tenant: always filter by tenant. With Supabase + RLS (item 4) this is enforced in the DB, but service-role code paths still need the filter.

---

## 8. Block field tampering (mass assignment)

**Look for**: `prisma.x.update({ data: req.body })`, `Model.create(req.body)`, `{ ...body }` spread into DB writes, `supabase.from(t).update(body)`, Django `ModelForm`/serializers with `fields = '__all__'`, Rails `params.permit!`, Laravel `$guarded = []`. Sensitive fields: `role`, `is_admin`, `plan`, `credits`, `balance`, `price`, `amount`, `user_id`, `owner_id`, `org_id`, `verified`, `email_verified`, `stripe_customer_id`.

**Fix**: parse the body with a schema that allowlists writable fields (item 14) and set ownership/price/role fields server-side. Prices come from the server/DB or Stripe, never from the client. In Postgres, also use column privileges or a trigger to stop users updating `role`.

---

## 9. Secure session cookies

**Look for**: `cookies().set(`, `res.cookie(`, `Set-Cookie`, `express-session`, `cookie-session`, `SESSION_COOKIE_*`, JWTs stored in `localStorage`/`AsyncStorage`.

**Fix**: `HttpOnly; Secure; SameSite=Lax` (or `Strict`), `Path=/`, reasonable `Max-Age`, `__Host-` prefix where possible; strong secret from env; rotate session on login; invalidate on logout and password change. Django: `SESSION_COOKIE_SECURE`, `CSRF_COOKIE_SECURE`, `SESSION_COOKIE_HTTPONLY`. Mobile: tokens in Keychain/Keystore (`expo-secure-store`), not AsyncStorage. If cookies are `SameSite=None`, CSRF protection is required.

---

## 10. Hash passwords

Skip (N/A) if a managed auth provider handles passwords.

**Look for**: `md5`, `sha1`, `sha256`, `createHash(` near `password`; password columns compared with `===`; `bcrypt` with cost < 10; passwords logged or emailed.

**Fix**: argon2id (`argon2`, `argon2-cffi`) or bcrypt cost ≥ 12; compare with the library's constant-time verify; rehash legacy hashes on next login (ask before a migration). Enforce a minimum length (≥ 8, ideally 12) and allow long passphrases.

---

## 11. Rate limit login

**Look for**: login, signup, password reset, magic link, OTP/2FA verify, and any expensive or paid endpoint (AI calls, email/SMS sending) with no limiter.

**Fix by stack**
- Next.js/edge/serverless: `@upstash/ratelimit` + Redis (in-memory limits don't work across serverless instances).
- Express/Fastify: `express-rate-limit` / `@fastify/rate-limit` (Redis store in production).
- Django: `django-ratelimit` or DRF throttling; Rails: `rack-attack`; Laravel: `throttle` middleware.
- Supabase Auth / Firebase Auth: built-in limits exist; check them in the dashboard (Manual) and still limit your own custom endpoints.
Limit per IP and per account/email; return 429; use generic error messages ("invalid email or password") to avoid account enumeration.

---

## 12. Add bot protection

**Look for**: public signup, login, contact, newsletter, comment and checkout forms with no CAPTCHA/challenge.

**Fix**: Cloudflare Turnstile (free, privacy-friendly), hCaptcha or reCAPTCHA; **verify the token on the server** (client-only widgets do nothing). Add a honeypot field as a cheap extra. Needs a site key from the provider → mark Manual for the key, implement the code.

---

## 13. Parameterize queries

**Look for**: template literals or `+` concatenation inside SQL (`` `SELECT ... ${` ``, `"WHERE id = " +`), `prisma.$queryRawUnsafe`, `$executeRawUnsafe`, `sequelize.query(` with interpolation, `knex.raw(` with interpolation, Python f-strings / `%` in `cursor.execute`, Django `.raw()`/`.extra()`, Rails `where("name = '#{`, Mongo `$where` or passing raw `req.body` objects as filters (operator injection like `{ "$ne": null }`), Supabase `.or()`/`.filter()` built from raw user strings.

**Fix**: use bound parameters (`$queryRaw` tagged template, `?`/`$1` placeholders, `cursor.execute(sql, params)`), cast Mongo filter values to primitives, allowlist sortable column names.

---

## 14. Validate all input

**Look for**: handlers reading `req.body`, `req.query`, `formData`, `searchParams`, webhook payloads without a schema; webhooks (Stripe, GitHub) without signature verification.

**Fix**: one schema per endpoint at the boundary (zod/valibot for TS, pydantic for FastAPI, DRF serializers, Rails strong params), with types, lengths, formats, enums and `.strict()` to drop unknown keys. Verify webhook signatures (`stripe.webhooks.constructEvent`). Validate on the server even if the client validates too.

---

## 15. Escape user content

**Look for**: `dangerouslySetInnerHTML`, `innerHTML =`, `v-html`, `{@html` (Svelte), `|safe` / `mark_safe` (Django/Jinja), `html_safe`/`raw` (Rails), `{!! !!}` (Blade), markdown renderers with HTML enabled, `href={userUrl}` (allows `javascript:` URLs), user data inside `<script>` JSON, emails built from user input.

**Fix**: render as text by default; when HTML is needed, sanitize with DOMPurify (`isomorphic-dompurify` on the server) or `bleach`/`nh3`; allowlist URL schemes (`http`, `https`, `mailto`); serialize JSON into scripts safely. CSP (item 18) is the backstop.

---

## 16. Restrict file uploads

**Look for**: `multer`, `formidable`, `busboy`, `request.formData()` with files, `UploadFile`, Active Storage, Supabase/Firebase Storage uploads from the client, presigned URLs.

**Fix**: max size; allowlist MIME types *and* check magic bytes server-side (`file-type`); generate random filenames (never use the user's path); store outside the web root / in a private bucket and serve with signed URLs; block SVG/HTML or serve with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`; for Supabase Storage, add bucket policies + `file_size_limit` / `allowed_mime_types`; for presigned S3 URLs, set content-length and content-type conditions. Re-encode images to strip EXIF if privacy matters.

---

## 17. Trim API responses

**Look for**: `return Response.json(user)`, `res.json(row)`, `select('*')`, `findMany()` without `select`, Django serializers with `__all__`, GraphQL types exposing internal fields; responses including `password_hash`, `reset_token`, `stripe_*`, other users' emails/phones, internal flags; verbose error responses with stack traces or SQL in production.

**Fix**: explicit `select` / DTO mapping per endpoint; paginate list endpoints with a max page size; generic error messages in production with details only in server logs.

---

## 18. Add security headers

**Look for**: `next.config.*` `headers()`, `helmet` in Express, `vercel.json` / `netlify.toml` / `_headers` / `nginx.conf`, Django `SecurityMiddleware` settings.

**Fix (baseline)**
```
Content-Security-Policy: default-src 'self'; frame-ancestors 'none'; object-src 'none'; base-uri 'self'   (then add the app's real sources)
Strict-Transport-Security: max-age=63072000; includeSubDomains; preload
X-Content-Type-Options: nosniff
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: camera=(), microphone=(), geolocation=()
X-Frame-Options: DENY   (legacy, alongside frame-ancestors)
```
Express: `app.use(helmet())`. Also: tight CORS (no `*` with credentials; allowlist origins), remove `X-Powered-By`.

**Gotchas**: a strict CSP can break the app (inline scripts, analytics, Stripe, Supabase URLs). Start with `Content-Security-Policy-Report-Only` if unsure, or build the allowlist from the app's actual third-party origins, and tell the user to test.

---

## 19. Force HTTPS

**Look for**: hard-coded `http://` URLs to your own API or assets (mixed content), nginx/Apache config without an HTTP→HTTPS redirect, `secure: false` cookies in production config, Django `SECURE_SSL_REDIRECT`, Rails `config.force_ssl`, Express behind a proxy without `trust proxy`, mobile `NSAllowsArbitraryLoads` / `usesCleartextTraffic="true"`.

**Fix**: redirect all HTTP to HTTPS (Vercel/Netlify do this by default → Manual: confirm in dashboard; Cloudflare: SSL mode "Full (strict)" + "Always Use HTTPS"), HSTS from item 18, `https://` everywhere, remove cleartext exceptions from mobile configs.

---

## 20. Scan dependencies

**Run** whichever applies (they need network; report if unavailable):
- `npm audit --omit=dev` / `pnpm audit --prod` / `yarn npm audit`
- `pip-audit` / `uv pip audit`; `bundle audit`; `composer audit`; `cargo audit`; `govulncheck ./...`; `flutter pub outdated`
- `osv-scanner -r .` if installed (multi-ecosystem).

**Fix**: upgrade high/critical issues in production dependencies with the smallest version bump that fixes them; don't `npm audit fix --force` blindly (it can do major upgrades), run tests after. Confirm the lockfile is committed. Suggest enabling Dependabot/Renovate and GitHub secret scanning + push protection.
