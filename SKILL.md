---
name: prelaunch-security
description: Pre-launch security hardening for web and mobile apps. Walks the codebase through a 20-point checklist (hide API keys, purge secrets from Git history, public vs service DB keys, Supabase/Postgres RLS, encrypting sensitive data, server-side auth, per-record access control, mass-assignment / field tampering, secure session cookies, password hashing, login rate limiting, bot protection, parameterized queries, input validation, output escaping / XSS, file upload restrictions, trimming API responses, security headers, forcing HTTPS, dependency scanning), fixes what it safely can, and reports a pass/fail table. Use whenever the user says "before I launch", "pre-launch checklist", "harden my app", "is my app ready to ship", "launch security", "secure my Supabase/Firebase/Next.js app", "/prelaunch-security", or asks to check any of these 20 items, even if they only name one or two of them.
---

# Pre-launch security hardening

Run a practical, fix-it-now security pass over the app in the current project before it goes live. This is not a deep pentest: it is the 20 things that most often get vibe-coded and indie apps breached in the first week, checked against the real code and fixed in place.

The detailed checks, grep patterns and per-stack fixes for each item live in [references/checklist.md](references/checklist.md). Read it before starting step 3.

## Workflow

### 1. Map the app (keep it short)

Find out, from the files rather than by asking:

- **Stack**: framework(s) (Next.js, Remix, SvelteKit, Express, Fastify, NestJS, Django, Flask, FastAPI, Rails, Laravel, Expo/React Native, Flutter...), language, package manager.
- **Data layer**: Supabase, Firebase, Postgres/MySQL via ORM (Prisma, Drizzle, TypeORM, Sequelize, SQLAlchemy, Django ORM), MongoDB, raw SQL.
- **Auth**: Supabase Auth, Firebase Auth, NextAuth/Auth.js, Clerk, Lucia, Passport, Devise, Django auth, custom.
- **Where code runs**: which files ship to the browser/device (client components, `NEXT_PUBLIC_*`, `VITE_*`, `EXPO_PUBLIC_*`, mobile bundles) and which run only on the server.
- **Hosting**: Vercel, Netlify, Cloudflare, Fly, Render, a VPS behind nginx, Hostinger, etc. (matters for headers and HTTPS).
- **Is it a git repo**, and is there a remote.

State the map in 3-5 lines to the user, then move on. If the user named only some items ("just check RLS and rate limiting"), run only those.

### 2. Decide the mode

- Default: **audit + fix**. Fix anything that is local, reversible and low-risk directly in the working tree.
- **Always ask before**: rewriting Git history (item 2), changing database policies on a live project (item 4), migrating stored data (items 5 and 10), adding a new paid/third-party service (item 12), or anything that would log existing users out.
- If the user said "audit only" / "just tell me", change nothing and only report.
- Never commit or push unless the user asks. Never print a secret's full value back to the user; show the first 4 characters and the file:line.

### 3. Run the 20 checks

Work through them in this order (it front-loads the ones that leak data). For each item: search, judge, fix or flag, and record the result. Use parallel Grep/Glob calls; don't read the whole repo.

| # | Check | Short version |
|---|-------|---------------|
| 1 | Hide API keys | No secret keys in client code, bundles or committed files; `.env*` ignored |
| 2 | Purge Git secrets | No secrets anywhere in Git history; leaked ones rotated |
| 3 | Use public DB key | Client only uses anon/publishable key; service-role/admin key server-only |
| 4 | Enable RLS | Every exposed table has RLS on with real policies (or equivalent rules) |
| 5 | Encrypt sensitive data | PII/tokens encrypted at rest; TLS to DB; no secrets in logs |
| 6 | Enforce server-side auth | Every protected route/action verifies the session on the server |
| 7 | Lock record access | Queries scope by owner/tenant; no IDOR via IDs in URL/body |
| 8 | Block field tampering | No mass assignment; allowlist writable fields (role, price, owner_id...) |
| 9 | Secure session cookies | `HttpOnly`, `Secure`, `SameSite`, sane expiry; no tokens in localStorage if avoidable |
| 10 | Hash passwords | bcrypt/argon2/scrypt with proper cost; never plain, MD5 or SHA |
| 11 | Rate limit login | Login, signup, reset, OTP endpoints throttled per IP + account |
| 12 | Add bot protection | CAPTCHA/Turnstile/hCaptcha or equivalent on public forms |
| 13 | Parameterize queries | No string-built SQL/NoSQL; ORM raw calls use bindings |
| 14 | Validate all input | Schema validation (zod, valibot, pydantic, Joi...) at every boundary |
| 15 | Escape user content | No `dangerouslySetInnerHTML`/`v-html`/`|safe` on user data without sanitizing |
| 16 | Restrict file uploads | Type + size limits, server-side checks, random names, private buckets |
| 17 | Trim API responses | Return DTOs, never whole rows (password_hash, tokens, emails of others) |
| 18 | Add security headers | CSP, HSTS, X-Content-Type-Options, frame-ancestors, Referrer-Policy |
| 19 | Force HTTPS | HTTP redirects to HTTPS, HSTS, no mixed content, secure cookies |
| 20 | Scan dependencies | `npm audit` / `pip-audit` / etc. clean of high/critical; lockfile committed |

Recommended order: 1, 2, 3, 4, 6, 7, 8, 17, 13, 15, 14, 10, 9, 11, 12, 16, 5, 18, 19, 20.

Mark an item **N/A** only with a reason (e.g. "no file uploads in the app"). Mark **Manual** when it can only be verified in a dashboard (Supabase RLS toggles on a hosted project, Vercel domain settings, Cloudflare SSL mode) and tell the user exactly where to click.

### 4. Fix

- Prefer the stack's native, boring mechanism (framework middleware, ORM features, the platform's header config) over new dependencies. If a dependency is genuinely needed (e.g. a rate limiter), pick the standard one for the stack and say why.
- Keep each fix minimal and in the style of the surrounding code.
- When a secret was ever committed or shipped to clients, **rotating it is the real fix**: removing it from code is not enough. Tell the user which keys to rotate and where, before or alongside any history rewrite.
- After fixing, re-run the relevant search to confirm, and run the project's typecheck/lint/tests if they exist.

### 5. Report

End with a single table and a short list of next actions:

```
| #  | Check                  | Status   | Where / what changed                          |
|----|------------------------|----------|-----------------------------------------------|
| 1  | Hide API keys          | Fixed    | moved OPENAI key from src/lib/ai.ts to server |
| 4  | Enable RLS             | FAIL     | `profiles`, `orders` have RLS off (SQL below) |
| 12 | Add bot protection     | Manual   | needs Turnstile site key from Cloudflare      |
| 16 | Restrict file uploads  | N/A      | app has no uploads                            |
```

Statuses: **Pass**, **Fixed**, **FAIL** (found, not fixed - say why), **Manual**, **N/A**.

Then: "Must do before launch" (every FAIL and Manual, highest risk first, including keys to rotate), and the list of files changed. Be honest: if something could not be verified from the code, say so instead of marking it Pass.
