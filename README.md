# prelaunch-security

A [Claude Code](https://claude.com/claude-code) skill that runs a **20-point security hardening pass on your app before you launch it**, fixes what it safely can, and hands you a pass/fail report.

It is aimed at the kind of app that gets shipped fast (Next.js + Supabase, Firebase, Express, Django, Rails, Laravel, Expo...) and at the mistakes that most often get those apps breached in their first week: leaked keys, missing Row Level Security, auth that only lives in the UI, and endpoints that trust whatever the client sends.

## The 20 checks

| # | Check | # | Check |
|---|-------|---|-------|
| 1 | Hide API keys | 11 | Rate limit login |
| 2 | Purge Git secrets | 12 | Add bot protection |
| 3 | Use public DB key | 13 | Parameterize queries |
| 4 | Enable RLS | 14 | Validate all input |
| 5 | Encrypt sensitive data | 15 | Escape user content |
| 6 | Enforce server-side auth | 16 | Restrict file uploads |
| 7 | Lock record access | 17 | Trim API responses |
| 8 | Block field tampering | 18 | Add security headers |
| 9 | Secure session cookies | 19 | Force HTTPS |
| 10 | Hash passwords | 20 | Scan dependencies |

Each one has concrete search patterns, bad signs, per-stack fixes and gotchas in [`references/checklist.md`](references/checklist.md).

## What it does

1. **Maps your app** from the files: stack, data layer, auth provider, what ships to the client, hosting.
2. **Runs the checks** against the real code (not a generic lecture), in an order that front-loads the ones that leak data.
3. **Fixes** local, reversible issues directly: moving keys server-side, adding auth guards, scoping queries by owner, allowlisting writable fields, adding headers, validation schemas, rate limiters, and so on.
4. **Asks first** before anything risky: rewriting Git history, changing live database policies, migrating stored data, adding a paid service, or logging users out.
5. **Reports** a table with `Pass` / `Fixed` / `FAIL` / `Manual` / `N/A` for every item, a "must do before launch" list (including which keys to rotate), and the files it changed.

It never commits or pushes on its own, and never prints full secret values back to you.

## Install

Personal skill (available in every project):

```bash
git clone https://github.com/mechichimohamed/prelaunch-security.git ~/.claude/skills/prelaunch-security
```

On Windows (PowerShell):

```powershell
git clone https://github.com/mechichimohamed/prelaunch-security.git "$env:USERPROFILE\.claude\skills\prelaunch-security"
```

Or per project: clone it into `.claude/skills/prelaunch-security` inside the repo.

Update later with `git pull` in that folder.

## Use

In Claude Code, inside your app's folder:

```
/prelaunch-security
```

or just ask in plain words:

- "Run the pre-launch security checklist on this app"
- "Is my app ready to ship? Harden it."
- "Audit only, don't change anything"
- "Just check RLS and rate limiting"

## Example report

```
| #  | Check               | Status | Where / what changed                              |
|----|---------------------|--------|---------------------------------------------------|
| 1  | Hide API keys       | Fixed  | OpenAI call moved from client to app/api/chat     |
| 3  | Use public DB key   | Pass   | service_role only in lib/supabase/admin.ts        |
| 4  | Enable RLS          | FAIL   | `orders` has RLS off - migration written, not run |
| 11 | Rate limit login    | Fixed  | @upstash/ratelimit on /api/auth/*                 |
| 12 | Add bot protection  | Manual | needs a Cloudflare Turnstile site key             |
| 16 | Restrict uploads    | N/A    | no file uploads                                   |
```

## Limits

This is a pre-launch hygiene pass, not a penetration test or a compliance audit. It reads code and config; things that live only in dashboards (Supabase RLS on a hosted project without migrations, Vercel/Cloudflare settings, provider rate limits) are marked **Manual** with directions. For a deeper review, pair it with a dedicated security audit.

## Layout

```
prelaunch-security/
├── SKILL.md                 # workflow Claude follows
├── references/
│   └── checklist.md         # detailed checks, patterns and fixes for all 20 items
├── README.md
└── LICENSE                  # MIT
```

## License

MIT
