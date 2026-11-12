# Threat model: johnbest-io repositories

Status: v2. Items marked *(owner)* come from the owner's answers; items marked *(inferred)* are a reading of the code, not confirmed by the owner.

## Scope

| Repository | Contents | Notes |
|---|---|---|
| `johnbest-io/garment-factory` | React SPA + Hono API on Cloudflare Workers, Postgres via Hyperdrive | Cloudflare "stitch" books template |
| `johnbest-io/empathic-voice-interface-starter` | Next.js 14 App Router app using Hume EVI | Hume starter template, Vercel deploy |
| `johnbest-io/mcp-server-apple-events` | Local MCP server (TypeScript + Swift EventKit CLI) | Fork of FradSer/mcp-server-apple-events |
| `johnbest-io/desktop-tutorial`, `johnbest-io/gpt-app` | README only | Nothing to scan |
| `johnbest-io/johnbest-io` | Empty (no commits) | Nothing to scan |

No separate infrastructure repository. Deployment config in scope: `garment-factory/wrangler.jsonc`, `garment-factory/docker-compose.yml` (local dev only).

## Components

- **Books Worker API**: `garment-factory/api/**` (Hono routes `/api/books`, `/api/books/:id`, `/api/books/:id/related`, Hyperdrive/Postgres client, mock fallback).
- **Books SPA**: `garment-factory/src/**`, `index.html`, Workers static assets.
- **Books DB/Infra**: `garment-factory/init.sql`, `docker-compose.yml`, `wrangler.jsonc`.
- **EVI App**: `empathic-voice-interface-starter/**` (server-side token minting in `utils/getHumeAccessToken.ts` / `app/page.tsx`, client voice UI in `components/**`).
- **MCP Server**: `mcp-server-apple-events/src/{server,tools,validation,utils,types}/**` (stdio transport, tool routing, Zod schemas, `cliExecutor.ts`, `binaryValidator.ts`).
- **EventKit Bridge**: `mcp-server-apple-events/src/swift/**`, `bin/run.cjs`, `scripts/{build-swift,postinstall}.mjs`.

## Trust boundaries and entry points

1. **Internet → Books Worker**. Not currently deployed *(owner)*; anonymous internet users still matter if it is deployed *(owner)*. Anonymous GET endpoints; query params `genre`, `sort`, path `id`. SQL uses `postgres` tagged templates (parameterized). Errors return `e.message` to the client (`api/lib/utils.js`). No auth, no writes.
2. **Internet → EVI App**. Not currently deployed *(owner)*; anonymous internet users still matter if it is deployed *(owner)*. Every page load runs `fetchAccessToken` with `HUME_API_KEY`/`HUME_SECRET_KEY` server-side and passes a short-lived Hume access token to the browser. No user auth, so any visitor can get a token and spend Hume quota.
3. **LLM client → MCP Server (stdio)**. Tool calls come from a local LLM client (e.g. Claude Desktop). Arguments are model-controlled and so can be steered by prompt injection from content the model reads, including reminder/calendar text returned by this server. Arguments are Zod-validated and passed to the Swift binary via `execFile` (no shell).
4. **MCP Server → macOS EventKit / osascript**. Runs with the user's Reminders and Calendar TCC permissions; can create, update and delete reminders/events.
5. **Supply chain**. npm dependencies; `postinstall` builds Swift on macOS; `bin/run.cjs` runs TypeScript via `tsx` at runtime.

## Assets

- Hume API key and secret key (server env on Vercel) *(inferred: high value)*.
- Hume usage quota / billing.
- User's Apple Reminders and Calendar data (personal, read/write/delete) *(inferred: high value)*.
- Postgres credentials via Hyperdrive binding; book catalog data (public, low sensitivity).
- Integrity of the local Mac via the MCP binary path and build scripts.

## Attacker profiles

All three of the following matter equally *(owner)*:
- Anonymous internet user against the Worker / EVI app, if deployed.
- Prompt-injection author: anyone who can get text in front of the LLM driving the MCP server.
- Supply chain: malicious npm dependency, postinstall/build scripts, compromised upstream template.

Also considered: local attacker on the Mac *(inferred: low priority, needs an already-compromised machine)*.

## Existing controls

- Parameterized SQL (`postgres` tagged templates) in `garment-factory/api/routes/*`.
- `server-only` guard on `getHumeAccessToken.ts`; secrets only via env (`.env.example`).
- MCP: Zod input validation (`src/validation/schemas.ts`), `execFile` without shell, binary path allowlist + size/hash checks (`src/utils/binaryValidator.ts`), EventKit permission gating in the Swift CLI.

## Out of scope / accepted risk

Nothing is out of scope; report everything *(owner)*. Issues inherited from upstream templates and Hume quota abuse are in scope.

Deployment status *(owner)*: nothing is deployed today; all repositories are local experiments or templates. This does not rule out any attacker (the owner wants internet-facing issues reported).

## Open questions

- Which assets matter most (Hume keys vs. Reminders/Calendar data)? Currently treated as equally important.
