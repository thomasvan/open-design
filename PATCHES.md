# PATCHES.md — fork-local changes vs upstream

**Fork:** `thomasvan/open-design` (parent repo: `nexu-io/open-design`)
**Upstream base at patch time:** `a8d94dfa6` (`fix(updater): make payload
runtime handoff atomic (#8348)`) — re-synced on 2026-08-25, +291 upstream commits
**Date:** 2026-08-24 (last updated 2026-08-25)

This file records every change this fork carries on top of upstream, and is
the reference for what is being contributed back. It is maintained by the
fork owner; keep it in sync whenever the fork diverges further.

Remotes:

- `origin` → `git@github.com:thomasvan/open-design.git` (this fork)
- `upstream` → `git@github.com:nexu-io/open-design.git` (parent)

## 1. Upstream-bound (contributed via PR)

| ID | Change | Files | Branch → PR |
|---|---|---|---|
| P1 | `fix(web)`: suppress hydration-mismatch warning on the theme-init script | `apps/web/app/layout.tsx` | `fix/web-hydration-script-suppress` → PR to `nexu-io/open-design` |

**P1 — why.** A script-injecting browser extension (observed id
`lgblnfidahcdcjddiepkckcfdhpknnjh`, `content/popups-script.js`) rewrites the
theme-init `<script>` node before React hydrates: it injects its own `src`
attribute and empties `__html`, which makes React raise a spurious
hydration-mismatch console error pointed at `app/layout.tsx`. The patch adds
`suppressHydrationWarning` to that script element, consistent with the
existing `suppressHydrationWarning` on `<html>`/`<body>`. Product behavior is
unchanged; the console error goes away. This is the only change in the PR.

The same change is also carried on this fork's `main` as a cherry-pick
(`ad38cd7c4`), so the local dev runtime serves the fixed code while PR #7348 is
still open. When upstream merges the PR, upstream's identical hunk merges
cleanly; drop the cherry-pick then if a linear history is preferred.

## 2. Local-only (fork, not submitted upstream)

| ID | Change | Files |
|---|---|---|
| L1 | Runbook: installing OpenDesign from source + DeepSeek Harness integration (EN + VI) | `docs/deepseek-harness-setup.md`, `docs/deepseek-harness-setup.vi.md` |

**L1 — why local-only.** These runbooks document a specific workstation's
install (2026-08-24, Ubuntu, Node v24, dsh 0.1.1-rc.2 at the time of writing,
now 0.1.5-rc.2) for learning purposes: exact paths, ports, observed outputs, and
machine-specific troubleshooting (AMR/vela sign-in fix, hydration-mismatch fix).
They are not product documentation for the general audience, so they stay in the
fork.

## 3. Machine-level (no repo change — environment only)

These fixes live outside the repository and are recorded here for
reproducibility on this machine:

- **AMR / OpenDesign Cloud sign-in** — linked `~/.local/bin/vela` to the real
  `@powerformer/vela-cli` entry
  (`node_modules/.pnpm/@powerformer+vela-cli@0.0.33/node_modules/@powerformer/vela-cli/bin/vela.cjs`).
  The pnpm `.bin/vela` shim cannot be symlinked (it resolves paths relative to
  its own directory). Daemon then reports `amr.available = true`.
- **DeepSeek Harness runtime** — `od agent setup deepseek-harness` installed
  `@open-design/dsh-runtime` v0.1.0 into `~/.dsh/profiles/open-design/`; daemon
  spawns `dsh --profile open-design --stdio` (stream `dsh-profile-jsonl`).
  Current `dsh` is `0.1.5-rc.2`, which is outside the runtime def's tested list
  (`0.1.0-rc.8`, `0.1.1-rc.2`) — a non-blocking `untested-version` warning.
- **Local runtime** — daemon on `0.0.0.0:7456` (default port; non-loopback
  binding requires `OD_API_TOKEN`) and web on `127.0.0.1:17573`, started with
  `pnpm tools-dev run web --daemon-port 7456 --web-port 17573`. `OD_BIND_HOST`
  and `OD_API_TOKEN` live in the gitignored `.env.development.local`, which
  tools-dev loads automatically.

## Suggesting rules for this fork

- New upstream-suitable fixes go on a `fix/…` branch off `upstream/main`, get
  added to section 1, and are submitted as a PR with this file as context.
- Machine-specific notes never enter the PR; keep them in section 2/3.
- Sync `main` with `git fetch upstream && git rebase upstream/main`, then
  `git push --force-with-lease origin main`. `--ff-only` does not apply while
  `main` carries the fork-local commit.