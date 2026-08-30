# Peanut Butter

**Spread the pace.** A voice-led running coach that reacts to your GPS pace in real time.

**Live site:** [https://peanutbutter.fitness](https://peanutbutter.fitness)

**Repository:** [github.com/LLLA-hackathon/Running-Hackathon](https://github.com/LLLA-hackathon/Running-Hackathon) (default branch: `main`)

Set an aim pace before you head out. Two voice modes push you along:

- **Chase (arch-enemy)** — drop below your aim pace and they close in on you.
- **Cheer (loved one)** — hold your pace and they carry you toward a PB.

The repo is a monorepo with two clients on one Supabase-backed API:

| Client | Path | Role |
| --- | --- | --- |
| **Web** | repo root | Landing page, onboarding, dashboard, REST API |
| **Mobile** | `mobile/` | Native GPS tracking and the run experience (Expo) |

## Project status

This is an active prototype. A few things to know before you treat it as production-ready:

- **Username-only identity** — no passwords or accounts yet. Anyone who types a username becomes that player.
- **Landing page** — the voice showcase replays scripted lines, the pace timeline is static, and the waitlist form does not persist submissions.
- **Mobile runs** — GPS tracking works in the Expo app, but runs are not yet synced to the database.
- **Voice cloning** — optional ElevenLabs integration; without an API key, coach audio and cloning are disabled but the app still runs.

## Quick start

**Prerequisites:** Node.js 22 (see `.nvmrc`), a [Supabase](https://supabase.com) project, and optionally an [ElevenLabs](https://elevenlabs.io) API key.

```bash
npm install
cp .env.example .env.local   # fill in Supabase URL, secret key and session secret
npm run dev
```

| URL | What |
| --- | --- |
| http://localhost:3000 | Landing page |
| http://localhost:3000/start | Claim a username and enter the app |

Apply the database schema from `supabase/migrations/` — see [docs/backend.md](docs/backend.md).

### Mobile app

```bash
cd mobile
npm install
npx expo start            # same Wi-Fi as the phone
npx expo start --tunnel   # phone on mobile data / different network
```

Open the printed `exp://…` link in **Expo Go** (SDK 57). To reach the web API from a physical phone:

```bash
# Dev — replace with your machine's LAN IP
EXPO_PUBLIC_API_URL=http://192.168.1.20:3000 npx expo start

# Production
EXPO_PUBLIC_API_URL=https://peanutbutter.fitness npx expo start
```

See [mobile/README.md](mobile/README.md) and [TESTING.md](TESTING.md) for the full mobile and end-to-end test guide.

## Environment variables

Copy [`.env.example`](.env.example) to `.env.local` for local development. **Never commit real values.**

### Web (`.env.local`)

| Variable | Required | Exposure | Purpose |
| --- | --- | --- | --- |
| `SUPABASE_URL` | Yes | Server only | Supabase project URL |
| `SUPABASE_SECRET_KEY` | Yes | Server only | Secret key (`sb_secret_…`) — bypasses RLS; never prefix with `NEXT_PUBLIC_` |
| `SESSION_SECRET` | Yes | Server only | HMAC signing for the player session cookie (`openssl rand -base64 32`) |
| `ELEVENLABS_API_KEY` | No | Server only | Coach TTS and voice cloning; without it, audio routes return 503 |
| `NEXT_PUBLIC_SITE_URL` | No | Public | Canonical site URL for metadata and sitemaps. Defaults to `http://localhost:3000` in dev; set to `https://peanutbutter.fitness` on Vercel |

Commented placeholders in `.env.example` for future Supabase Auth client usage:

- `NEXT_PUBLIC_SUPABASE_URL`
- `NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY`

### Mobile

| Variable | Required | Purpose |
| --- | --- | --- |
| `EXPO_PUBLIC_API_URL` | No | Next.js API base URL. Defaults to `http://localhost:3000`; set to `https://peanutbutter.fitness` for production builds |

## Deployment

The production site runs on **Vercel** with **Supabase** as the database and storage backend.

### Vercel

1. Connect the GitHub repo to a Vercel project.
2. Set environment variables in the Vercel dashboard:

   | Variable | Value |
   | --- | --- |
   | `SUPABASE_URL` | Your Supabase project URL |
   | `SUPABASE_SECRET_KEY` | Secret key from Supabase → Project Settings → API Keys |
   | `SESSION_SECRET` | Random 32-byte string (`openssl rand -base64 32`) |
   | `NEXT_PUBLIC_SITE_URL` | `https://peanutbutter.fitness` |
   | `ELEVENLABS_API_KEY` | Optional — enables coach audio and voice cloning |

3. Point DNS for `peanutbutter.fitness` (and `www` if used) at Vercel.
4. **Redeploy** after adding or changing `NEXT_PUBLIC_SITE_URL` — it is baked into the build for metadata and sitemaps.

Build and start commands (Vercel defaults):

```bash
npm run build
npm start
```

### Supabase

1. Create a project and apply migrations from `supabase/migrations/`.
2. Confirm the `voice-samples` storage bucket is **private** (created by migration).
3. When Supabase Auth is added later, set **Site URL** and **Redirect URLs** to `https://peanutbutter.fitness` in the Supabase dashboard.

### Production URLs

| Resource | URL |
| --- | --- |
| Web app | https://peanutbutter.fitness |
| Sitemap | https://peanutbutter.fitness/sitemap.xml |
| Robots | https://peanutbutter.fitness/robots.txt |

## Security

### What is in place

- **No secrets in source** — only placeholders in `.env.example`; `.gitignore` excludes `.env*`.
- **Server-only Supabase access** — `SUPABASE_SECRET_KEY` never reaches the browser. RLS is enabled with zero policies, so anon/publishable keys cannot read or write data.
- **Signed sessions** — httpOnly cookies with HMAC-SHA256 (`timingSafeEqual` verification), `sameSite: lax`, `secure` in production.
- **Scoped queries** — player ID is resolved from the session, never from the request body. Run points are computed server-side.
- **Restricted TTS** — `/api/coach-voice` only serves predefined coach/line combinations, not arbitrary text.
- **Security headers** — `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, HSTS, and `Permissions-Policy` via `next.config.ts`.
- **Rate limiting** — IP-based limits on `/api/auth/session`, `/api/coach-voice`, `/api/voices`, and `/api/runs` via `src/proxy.ts`. Per-isolate in-memory buckets curb casual abuse; consider [Vercel Firewall](https://vercel.com/docs/security/firewall) for edge-wide limits.
- **Upload validation** — voice sample MIME types and size are checked server-side.

### Known limitations

- **Username impersonation** — until Supabase Auth ships, a username is a handle, not an account. Do not store sensitive personal data.
- **No account recovery** — suitable for a hackathon demo, not for production user accounts without further work.
- **ElevenLabs cost** — `/api/coach-voice` can trigger TTS calls when an API key is configured. Rate limiting and long cache headers mitigate abuse; pre-generated static clips are planned for a future release.

See [docs/backend.md](docs/backend.md) for the full security model and the auth migration path.

## Scripts

```bash
npm run dev       # development server
npm run build     # production build
npm start         # serve the production build
npm run lint      # ESLint
npm run test      # Vitest unit tests
npx tsc --noEmit  # typecheck (run a build first so route types are generated)
```

Mobile typecheck:

```bash
cd mobile && npm run typecheck
```

CI (GitHub Actions) runs lint, test, build, and typecheck on every push and pull request.

## Project structure

```
src/app/              Next.js routes (landing, onboarding, dashboard, API)
src/components/       UI components (landing, onboarding, runs, shadcn/ui)
src/lib/              Server logic (player, runs, voice, Supabase, rate limiting)
shared/               GPS maths and coach voice config (web + mobile)
supabase/migrations/  Postgres schema
mobile/               Expo (React Native) app
docs/                 Backend reference and planning notes
```

## Stack

Next.js 16 (App Router), React 19, TypeScript, Tailwind CSS v4, shadcn/ui, Supabase, ElevenLabs, Expo SDK 57.

## Documentation

| Doc | Contents |
| --- | --- |
| [docs/backend.md](docs/backend.md) | Supabase setup, identity model, onboarding, API reference |
| [TESTING.md](TESTING.md) | Team test script for web, mobile, and API |
| [mobile/README.md](mobile/README.md) | Expo app screens, coach voices, background location |

## License

The Expo mobile app inherits the MIT license in [mobile/LICENSE](mobile/LICENSE) (Expo template). The root project does not yet have its own license file.
