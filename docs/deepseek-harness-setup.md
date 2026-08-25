# Runbook: Installing OpenDesign from source and integrating DeepSeek Harness

> **Scope:** This runbook records exactly what was done on the Ubuntu workstation
> (`cm-workstation`, x86_64) on 2026-08-24, from `main @ e34d82316` of this repo,
> so the reader can **learn the process** and reproduce it.
> Unlike [`install-guide.md`](install-guide.md) (the Docker one-click build) and
> [`deepseek-harness-one-click-install.zh-CN.md`](deepseek-harness-one-click-install.zh-CN.md)
> (the release installer), this document follows the **run-from-source** path and
> integrates **DeepSeek Harness as a native runtime** of OpenDesign.
>
> Every command, path, and output in this document was actually executed and
> verified during that install session. Anything **not confirmed** is explicitly
> marked with ⚠️.
>
> Vietnamese translation: [deepseek-harness-setup.vi.md](deepseek-harness-setup.vi.md).

---

## 1. Goal

- Install OpenDesign (web + daemon + `od` CLI) running from source via `pnpm tools-dev`.
- Integrate **DeepSeek Harness (`dsh`)** as an agent runtime: the daemon spawns
  `dsh --profile open-design --stdio`, streams over the `dsh-profile-jsonl`
  protocol, with model discovery, cancellation, and session resume.
- Verify end-to-end: create a project → send a chat → the `deepseek-v4-flash`
  model replies.

## 2. Concepts to understand first

| Concept | Role |
|---|---|
| **apps/web** | Next.js 16 App Router — the user interface (chat, iframe preview, settings). |
| **apps/daemon** | Express + SQLite daemon; owns `/api/*`, agent CLI spawning, skills, design systems. The only privileged "local server". |
| **`pnpm tools-dev`** | The single lifecycle entry point (start/stop/status/logs). Do not use the removed legacy aliases (`pnpm dev`, `pnpm start`…). |
| **Runtime registry** | `apps/daemon/src/runtimes/registry.ts` — the list of supported agent CLIs; each runtime has a def under `runtimes/defs/`. |
| **dsh profile** | An install under `~/.dsh/profiles/<name>/`. OpenDesign uses the profile named `open-design`, which carries the `@open-design/dsh-runtime` plugin — the "connection component" that lets OpenDesign talk to dsh. |
| **`dsh-profile-jsonl`** | The JSONL stream format the runtime def uses to parse events (thinking, text, usage…) from the dsh process. |

Architecture details: [docs/architecture.md](architecture.md) · adapter contract:
[docs/agent-adapters.md](agent-adapters.md).

## 3. Prerequisites (checked on this machine)

| Item | Requirement | This machine |
|---|---|---|
| Node.js | `~24` (required, via `engines` in `package.json`) | ✅ **v24.19.0** |
| pnpm | `10.33.x` (pinned via `packageManager: pnpm@10.33.2`; use Corepack) | ✅ **10.33.2** |
| `dsh` CLI | Official DeepSeek Harness CLI, on PATH | ✅ **0.1.1-rc.2** (at `~/.npm/_npx/…/node_modules/.bin/dsh`) |
| Free disk | ~50 GB (node_modules + pnpm store + Electron) | ✅ |
| Network | npm registry + GitHub releases (Electron prebuilt) | ✅ |

> ⚠️ **Beware [`/usr/bin/od`](https://man7.org/linux/man-pages/man1/od.1.html):** on
> Linux, `od` is the system octal-dump utility by default — NOT the OpenDesign CLI.
> When using the CLI outside the UI, always call it by absolute path
> `apps/daemon/dist/cli.js`.

## 4. Steps

### Step 1 — Enable pnpm via Corepack

The repo pins pnpm through `packageManager`, so Corepack is all you need:

```bash
corepack enable
corepack pnpm --version   # must print 10.33.2
```

- Corepack installs a `pnpm` shim into the active Node's bin directory (`…/node_modules/corepack/dist/pnpm.js`).
- ⚠️ **Hit on this machine:** before full access was granted, Corepack failed with
  `EROFS: read-only file system` when writing its cache to `~/.cache/node/corepack`
  (the environment's sandbox blocks writes outside the workspace). Workaround when
  sandboxed: point `COREPACK_HOME` (and pnpm's `--store-dir`/`--cache-dir`) at a
  directory inside the workspace. No longer needed once `~/.cache` is writable.

### Step 2 — Install workspace dependencies

```bash
pnpm install
```

Observed results:

- Finished in **~2 minutes** (`Done in 2m 1.8s using pnpm v10.33.2`).
- The repo's `postinstall` builds many internal packages (daemon, tools-dev,
  tools-pack, tools-serve, `packages/dsh-runtime`…) — meaning **no separate build
  step is needed for those packages**; `dist/` is already produced.
- The native `better-sqlite3` module (daemon) is compiled for Node 24; quick check:

  ```bash
  cd apps/daemon && node -e "const db=require('better-sqlite3'); const d=new db(':memory:'); d.exec('create table t(a)'); console.log('daemon sqlite ok')"
  ```

  (> Run it **from inside the package directory** — pnpm does not hoist everything
  to the root, so `require('better-sqlite3')` from the repo root reports "Cannot
  find module". That is expected.)

- ⚠️ **Ignored build scripts warning:** pnpm prints
  `Ignored build scripts: @google/genai@1.52.0, node-pty@1.1.0` (per the repo's
  `onlyBuiltDependencies` policy). If `node-pty` (terminal) or those scripts are
  needed later, run `pnpm approve-builds` to allow them — the integration here is
  unaffected.

### Step 3 — Start daemon + web

The only lifecycle entry point is `tools-dev`. Use fixed ports for predictability:

```bash
pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```

- `tools-dev` starts the daemon first, then passes its port to web; `apps/web/next.config.ts`
  rewrites `/api/*` to the daemon port.
- Check the daemon: `curl http://127.0.0.1:17456/api/health` →
  `{"ok":true,"version":"0.20.3"}`.
- The web (Next dev server) needs a **warm-up** on the first request (~60s
  compile), then responds fast (HTTP 200 in ~0.3s).

| Service | URL |
|---|---|
| Web UI | http://127.0.0.1:17573 |
| Daemon API | http://127.0.0.1:17456 |

Stop when needed: `pnpm tools-dev stop` · status: `pnpm tools-dev status` ·
logs: `pnpm tools-dev logs`.

### Step 4 — Integrate DeepSeek Harness (the main step)

One command, run with the built daemon CLI (never bare `od`):

```bash
node apps/daemon/dist/cli.js agent setup deepseek-harness \
  --daemon-url http://127.0.0.1:17456 --json
```

Result: `{"ok":true,"packageVersion":"0.1.0"}` and the agent is detected as
`available: true` (see Step 5).

**What happens under the hood** (read from `apps/daemon/src/agent-companion-setup.ts`):

1. **Detect** — the daemon runs `dsh --version` and probes `dsh --profile open-design --probe`;
   before install the profile does not exist, so it is "not yet compatible".
2. **Package the connection component** — the daemon builds `@open-design/dsh-runtime`
   (`pnpm --filter @open-design/dsh-runtime build`) then `pnpm pack` it into a
   tarball, computes SHA-256 and writes a `manifest.json` (integrity-verification mechanism).
3. **Stage** — the tarball is written into the profile directory
   `~/.dsh/profiles/open-design/.open-design/<sha256>.tgz`.
4. **Install the plugin into dsh** — the daemon runs
   `dsh plugin --profile open-design add .open-design/<sha>.tgz`; dsh creates the
   `open-design` profile (package.json, cordis.yml, node_modules, pnpm-lock…).
5. **Re-probe** — probes again; on success it reports `action: "installed"` (or
   `"repaired"` if the profile already existed, `"already-compatible"` if nothing was needed).

> This step writes into `~/.dsh/` (outside the repo), so a sandboxed environment
> blocks it and needs permission — on this machine it ran with full access.

### Step 5 — Verify the runtime

Standalone probe loop (no daemon needed):

```bash
dsh --profile open-design --probe
# {"v":1,"type":"probe","runtime":"open-design","protocol_version":1,
#  "plugin_version":"0.1.0","capabilities":{"session_resume":true,"session_cancel":true,"structured_events":true}}

dsh --profile open-design --models
# {"v":1,"type":"models","runtime":"open-design",
#  "models":[{"provider":"nine-router","provider_name":"nine-router","id":"deepseek-v4-flash",...}]}
```

Through the daemon (the same API the web UI uses to render the runtime list):

```bash
curl http://127.0.0.1:17456/api/agents
```

→ runtime `deepseek-harness`: `"available": true`, `"modelsSource": "live"`,
models `["default", "nine-router/deepseek-v4-flash"]`, path pointing at the real `dsh`.

⚠️ **"untested-version" warning:** the local dsh is `0.1.1-rc.2`, while the runtime
def declares `supportedVersions: ['0.1.0-rc.6']`
(`apps/daemon/src/runtimes/defs/deepseek-harness.ts`). After checking
`apps/daemon/src/runtimes/detection.ts`: a version outside the list only produces
a **diagnostic warning and does not gate availability** — proven by the Step 6 smoke test.

### Step 6 — End-to-end smoke test

Create a project, then send a minimal chat via `/api/chat`:

```bash
curl -X POST http://127.0.0.1:17456/api/projects \
  -H 'content-type: application/json' \
  -d '{"id":"smoke-test","name":"Smoke Test"}'

curl -N -X POST http://127.0.0.1:17456/api/chat \
  -H 'content-type: application/json' \
  -d '{"projectId":"smoke-test","agentId":"deepseek-harness",
       "model":"nine-router/deepseek-v4-flash",
       "message":"Reply with the single word: OK"}'
```

Observed SSE sequence (evidence of the full loop):

```
event: start     → runId, agentId "deepseek-harness", streamFormat "dsh-profile-jsonl",
                   model "nine-router/deepseek-v4-flash"
event: agent     → type "status" "working", sessionId "od-…"   (native session started)
event: agent     → type "thinking_start" / "thinking_delta" …  (structured thinking)
event: agent     → type "text_delta" "OK"                      (the result)
event: agent     → type "usage" provider "nine-router" model "deepseek-v4-flash"
                   input_tokens 26147, output_tokens 38
event: diagnostic→ runtime_close, rpc_close_reason "exit_0", status "succeeded"
event: end       → code 0, status "succeeded", artifactCount 0
```

You also see `native_session_recovery` events (first `no_recoverable_session`,
then `captured_not_resumed` after spawning) — i.e. the dsh runtime's **session
resume** mechanism is enabled (`resumesSessionViaProfileStdio`,
`capturesSessionIdFromStream` in the def).

## 5. Final state (verified)

| Component | State |
|---|---|
| pnpm 10.33.2 (Corepack) | ✅ |
| `pnpm install` full workspace | ✅ 2 min, with a non-blocking approve-builds warning |
| Daemon v0.20.3 @ `127.0.0.1:17456` | ✅ `/api/health` OK |
| Web @ `127.0.0.1:17573` | ✅ HTTP 200 |
| Profile `~/.dsh/profiles/open-design` | ✅ plugin `@open-design/dsh-runtime` 0.1.0 |
| `dsh --profile open-design --probe` | ✅ `plugin_version: "0.1.0"` |
| `dsh --profile open-design --models` | ✅ `nine-router/deepseek-v4-flash` |
| `GET /api/agents` | ✅ `deepseek-harness available: true` (untested-version warning only) |
| End-to-end chat | ✅ model replied "OK", `status: succeeded`, `exit 0` |
| AMR (`vela` 0.0.33) runtime | ✅ `available: true` after the symlink fix (see [section 8](#8-follow-up-fix-amr--opendesign-cloud-sign-in-vela-cli)) |

## 6. How to use

**Via the UI:** open http://127.0.0.1:17573 → pick/create a project (a `smoke-test`
demo project already exists) → select **DeepSeek Harness**, model
**deepseek-v4-flash · nine-router** → enter a brief (prototype / deck / image /
document…) and send.

**Via the CLI** (always use the absolute path):

```bash
node apps/daemon/dist/cli.js project list --daemon-url http://127.0.0.1:17456 --json
node apps/daemon/dist/cli.js skills list --daemon-url http://127.0.0.1:17456 --json
node apps/daemon/dist/cli.js agent setup deepseek-harness --daemon-url http://127.0.0.1:17456 --json
```

## 7. Pitfalls & lessons learned

1. **`/usr/bin/od` shadowing** — don't run bare `od`; use `node apps/daemon/dist/cli.js`.
2. **CLI default port** is `127.0.0.1:7456` — if the daemon runs on another port,
   pass `--daemon-url` (or set `OD_DAEMON_URL`). Resolution order:
   flag → `OD_DAEMON_URL` → `OD_SIDECAR_IPC_PATH` → `:7456` (`apps/daemon/src/daemon-url.ts`).
3. **Sandbox blocks writes outside the workspace** — Corepack (`~/.cache/node/corepack`),
   the pnpm store, and `~/.dsh` are all affected. Avoid by setting `COREPACK_HOME`
   plus `--store-dir`/`--cache-dir` inside the workspace, or grant full access.
4. **The untested-version warning does not block** — it is diagnostic only; the real
   loop runs fine.
5. **Model key** — the smoke test ran **without** exporting `NINE_ROUTER_API_KEY`
   or `DEEPSEEK_API_KEY` to the shell. ⚠️ **Not verified:** the exact mechanism dsh
   uses to resolve the key at inference time (possibly from its settings/storage).
   If you hit an auth-style error ("no model API key"), the guidance in
   `apps/daemon/src/runtimes/auth.ts` is: run `dsh web` → Settings → Models to
   configure the key, or export `DEEPSEEK_API_KEY` into the daemon process environment.
6. **The web dev server needs a warm-up** on the first request (~60s) — this is not a bug.
7. **Restart after daemon code changes:** `pnpm --filter @open-design/daemon build`
   then `pnpm tools-dev restart --daemon-port 17456 --web-port 17573`.
8. **Daemon data:** all daemon-owned data lives under the **daemon data root**
   resolved at daemon startup — the path rules are mandatory reading in the root
   `AGENTS.md` → **Daemon data directory contract**; this document does not restate
   that contract. (Observation note: the `smoke-test` project's cwd in this run was
   under the data directory inside the workspace repo, visible in the run's `start` event.)

## 8. Follow-up fix: AMR / OpenDesign Cloud sign-in (vela CLI)

> Applied on 2026-08-24, same session as the install above. Verified end-to-end.

**Symptom.** Clicking **Sign in** in the web UI fails with a 500. The tools-dev log shows:

```
[browser] [amr-login] startVelaLogin failed { …
  error: 'vela binary not found; install vela or configure VELA_BIN', ok: false, status: 500 }
```

**Root cause.** The sign-in flow runs the AMR login (`handleCloudSignIn` →
`handleAmrSignInToContinue` in `apps/web/src/components/EntryShell.tsx`), which makes
the daemon spawn the **`vela`** CLI (`amr` runtime def, bin `vela`,
`apps/daemon/src/runtimes/defs/amr.ts`). The daemon resolves that binary via `VELA_BIN`
or a PATH scan (`apps/daemon/src/runtimes/executables.ts`). This machine had **no `vela`
on PATH and no `VELA_BIN`**, so both the direct and proxy spawn routes failed with
`vela binary not found` (`apps/daemon/src/integrations/vela.ts`, `vela-command.ts`).

**The fix applied.** No new install was needed — the `vela` CLI already ships in this
repo as a dependency of `tools/pack` (`@powerformer/vela-cli@0.0.33` plus the
`@powerformer/vela-cli-linux-x64` platform binary):

1. Verified the binary: `tools/pack/node_modules/.bin/vela --version` → `0.0.33`.
2. Linked it into `~/.local/bin/vela` (a directory already on the daemon's PATH),
   pointing at the **real package entry**:
   `node_modules/.pnpm/@powerformer+vela-cli@0.0.33/node_modules/@powerformer/vela-cli/bin/vela.cjs`.
   ⚠️ Gotcha: symlinking the pnpm `.bin/vela` shim itself fails, because the shim
   resolves paths relative to its own directory — target the real entry instead.
3. Restarted the runtime: `pnpm tools-dev stop`, then
   `pnpm tools-dev run web --daemon-port 17456 --web-port 17573`.

**Verification after the fix.**

```bash
curl http://127.0.0.1:17456/api/agents
# amr → { "available": true, "version": "0.0.33", "path": "/home/ubuntu/.local/bin/vela", "diagnostics": [] }

curl http://127.0.0.1:17456/api/integrations/vela/status
# {"loggedIn":false,"profile":"prod","configPath":"/home/ubuntu/.amr/config.json", ...}
```

**What to do now.** Open http://127.0.0.1:17573 (hard refresh), click **Sign in**
again — the daemon now spawns `vela`, and the UI shows a device-activation link;
complete it with an OpenDesign Cloud / AMR account (create one if needed). Sign-in is
optional for local usage: the DeepSeek Harness runtime and other local CLIs work
without it.

**Outcome (2026-08-24, same day):** the login flow now completes —
`/api/integrations/vela/status` returned `loggedIn: true`
(`sessionState: "authenticated"`, the configured AMR account). The earlier `{}`
failure shown in the UI was a stale browser session from before the daemon restart;
a hard refresh surfaces the signed-in state.

**Caveats.**
- The `~/.local/bin/vela` link points into the pnpm virtual store; a future
  `pnpm install` may prune or re-hash the package. If `vela` disappears, re-link it,
  or persist the config via **Settings → Execution mode → AMR agent CLI env**
  (`VELA_BIN`) — settings-backed config overrides the inherited environment.
- The AMR model list stays empty until the account is signed in (live catalog).

### 8.2 Console error: hydration mismatch (browser extension)

**Symptom.** The web console shows a React hydration-mismatch error pointing at
`apps/web/app/layout.tsx:41` (the theme-init inline `<script>`). The diff shows the
server-rendered node carrying `src="chrome-extension://lgblnfidahcdcjddiepkckcfdhpknnjh/content/popups-script.js"`
and an emptied `__html`, while the client render has the real theme script.

**Root cause.** A browser extension (id `lgblnfidahcdcjddiepkckcfdhpknnjh`, a
script-injecting "popups" helper) rewrites the theme-init `<script>` node before React
hydrates — exactly the "browser extension installed which messes with the HTML" case
React documents for hydration mismatches. The page itself works; the error is console
noise caused by extension DOM tampering.

**Fix applied.** Added `suppressHydrationWarning` to the `<script>` element in
`apps/web/app/layout.tsx` (consistent with the existing `suppressHydrationWarning` on
`<html>` and `<body>`). Verified: `pnpm --filter @open-design/web typecheck` passes and
the page still serves HTTP 200 with the theme script intact. The Next dev server picks
the change up via HMR; no restart needed.

**Permanent user-side fix (recommended).** Disable/remove that extension for this site
(chrome://extensions → find `lgblnfidahcdcjddiepkckcfdhpknnjh`). The app-side patch only
silences the warning; the extension keeps rewriting every page it runs on.

## 9. Reference documents & source

- [QUICKSTART.md](../QUICKSTART.md) — official quickstart (one-shot dev, scripts, troubleshooting).
- [docs/architecture.md](architecture.md) · [docs/agent-adapters.md](agent-adapters.md) — architecture & adapter contract.
- `apps/daemon/src/runtimes/defs/deepseek-harness.ts` — dsh runtime definition (bin, args, probe, models, versionPolicy).
- `apps/daemon/src/agent-companion-setup.ts` — the connection-component install/repair flow.
- `apps/daemon/src/runtimes/detection.ts` — detection logic (version gate, compatibility probe, version warning).
- `apps/daemon/src/runtimes/auth.ts` — per-runtime auth guidance/failure handling.
- `packages/dsh-runtime/` — source of the `@open-design/dsh-runtime` plugin (packaged into the dsh profile).
- [docs/deepseek-harness-one-click-install.zh-CN.md](deepseek-harness-one-click-install.zh-CN.md) — one-click install path for end users.