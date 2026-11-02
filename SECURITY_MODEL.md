# Threat model: Nicolasara repositories

Scope: `Nicolasara/chibisafe`, `Nicolasara/chibisafe-extension`, `Nicolasara/personalPage`, `Nicolasara/Raft`. No repository has a `SECURITY_MODEL.md`. This model was drafted from the code and the owner's answers.

## Owner-provided facts
- **chibisafe is not deployed.** It is a fork of `chibisafe/chibisafe` and not deployed by the owner.
- **Nothing is out of scope.** Every attacker and every repository matters, and there is no accepted risk.
- **No infrastructure repository.** The only infrastructure code is chibisafe's `docker-compose*.yml`, `Caddyfile` and `docker/`. The AWS API Gateway backend that personalPage calls is not in these repos.

## Components
| Component | Code | Trust boundary |
|---|---|---|
| **chibisafe API** | `chibisafe/packages/backend/src` (Fastify; routes in `routes/`, middlewares `apiKey`, `auth`, `admin`, `owner`, `ban`; Prisma/SQLite in `prisma/`; local or S3 storage) | Internet → API at `/api/*` |
| **chibisafe Web UI** | `chibisafe/packages/next/src` (Next.js: dashboard, public album `/a/[identifier]`, snippet `/s/[identifier]`, link `/l/[identifier]`, file/snippet viewers using `dangerouslySetInnerHTML`) | Browser ↔ UI ↔ API |
| **chibisafe Deployment** | `chibisafe/Caddyfile`, `chibisafe/docker-compose*.yml`, `chibisafe/docker/`, `chibisafe/packages/migration` | Caddy serves `/app/uploads` statically and proxies the API (`:8000`) and UI (`:8001`) |
| **Browser Extension** | `chibisafe-extension/src` (MV3 `background.mv3.js`, MV2 `background.js`, `content.js`, `settings.js`; `<all_urls>` host permissions, `scripting`, `declarativeNetRequest`) | Arbitrary web pages ↔ extension ↔ configured chibisafe instance |
| **personalPage** | `personalPage/src` (static React/Vite site; `services/github.ts` reads from an AWS API Gateway endpoint) | Public visitors → static site → API Gateway |
| **Raft KV store** | `Raft/3700kvstore`, `Raft/run` (Python, UDP on localhost, run by the course harness) | Local processes only |

## Entry points
- **chibisafe API.** Public endpoints include upload (`/api/upload` with `apiKey` and optional `auth`, so anonymous uploads are possible in public mode), login and register, and public album, snippet and link views. Authenticated users can manage their own files, albums, tags, snippets, links and API keys. Admin and owner routes cover users, IP bans, quarantine, settings including S3 credentials, diagnostics, and an unzip route that is commented out. Auth uses a JWT from the `Authorization` header or the `token` cookie, or an `x-api-key` header.
- **chibisafe Web UI.** It renders user-uploaded text and snippets as HTML.
- **Browser Extension.** It acts on context-menu actions on any page: it fetches the right-clicked media URL, takes screenshots, and sends `runtime.onMessage` messages between `content.js` and the background script. Settings store the instance domain and API key in extension storage.
- **personalPage.** It is public, read-only, and has no user input beyond pagination.
- **Raft KV store.** UDP JSON messages from local peers.

## Assets
1. **chibisafe user files and private albums.** Confidentiality and integrity across users.
2. **chibisafe credentials.** JWT secret (`SETTINGS.secret`), user API keys, password hashes (bcrypt), S3 access and secret keys in the settings table, and the default `admin`/`admin` account.
3. **chibisafe server host.** Filesystem under `uploads/`, `database/` and `logs/`, plus any route to RCE or path traversal.
4. **Extension user's chibisafe API key and browsing context.** The extension can script every origin.
5. **personalPage and Raft.** Low value: public data, and a localhost course project.

## Attackers
- **Anonymous internet user** against a chibisafe instance (upload, register, public views).
- **Low-privilege registered chibisafe user** going after other users' data or admin functions (IDOR, privilege escalation).
- **Uploader of malicious content** aimed at viewers: stored XSS via file or snippet rendering, or files served from the upload origin.
- **Malicious web page** visited by an extension user, trying to steal the API key, trigger uploads, or abuse extension privileges.
- **Malicious or compromised dependency** (CVE in npm packages).
- **Local process / network peer** for Raft (minimal).

## Existing controls
- chibisafe: `@fastify/helmet`, `@fastify/cors`, `@fastify/rate-limit`, bcrypt passwords, JWT invalidation on password change (`passwordEditedAt`), `enabled` user flag, IP bans, blocked extensions (`.exe`, `.sh`, ...), quotas, admin quarantine, Zod schemas on routes.
- Extension: the API key is sent only as `x-api-key` to the configured domain; the content script is injected on demand.

## Out of scope / accepted risk
- None, per the owner.

## Open questions
- Whether chibisafe will be deployed later, and in which mode (public, user accounts, or invite-only).
- Whether the extension is published under the owner's name or kept as a fork only.
- The AWS API Gateway/Lambda backend behind personalPage is not covered.
