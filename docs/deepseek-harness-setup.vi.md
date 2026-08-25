# Runbook: Cài đặt OpenDesign từ source và tích hợp DeepSeek Harness (tiếng Việt)

> **Phạm vi:** Đây là runbook ghi lại chính xác những gì đã làm trên máy workstation
> Ubuntu (`cm-workstation`, x86_64) vào ngày 2026-08-24, từ bản `main @ e34d82316`
> của repo này, để người đọc **học được quy trình** và có thể tái lập.
> Khác với [`install-guide.md`](install-guide.md) (bản Docker one-click) và
> [`deepseek-harness-one-click-install.zh-CN.md`](deepseek-harness-one-click-install.zh-CN.md)
> (installer phát hành), tài liệu này đi theo đường **chạy từ source** và tích hợp
> **DeepSeek Harness làm native runtime** của OpenDesign.
>
> Mọi lệnh, đường dẫn, output trong tài liệu này đều đã được chạy và xác minh thực tế
> ở phiên cài đặt đó. Phần nào **chưa xác minh** được đánh dấu rõ ràng bằng ⚠️.

---

## 1. Mục tiêu

- Cài đặt OpenDesign (web + daemon + `od` CLI) chạy từ source bằng `pnpm tools-dev`.
- Tích hợp **DeepSeek Harness (`dsh`)** làm runtime agent: daemon spawn
  `dsh --profile open-design --stdio`, stream theo giao thức `dsh-profile-jsonl`,
  có model discovery, cancellation, session resume.
- Xác minh end-to-end: tạo project → gửi chat → model `deepseek-v4-flash` trả lời.

## 2. Các khái niệm cần hiểu trước

| Khái niệm | Vai trò |
|---|---|
| **apps/web** | Next.js 16 App Router — giao diện người dùng (chat, preview iframe, settings). |
| **apps/daemon** | Express + SQLite daemon; sở hữu `/api/*`, spawn agent CLI, skills, design systems. Đây là "máy chủ local" duy nhất có quyền. |
| **`pnpm tools-dev`** | Điểm vào lifecycle duy nhất (start/stop/status/logs). Không dùng các alias cũ (`pnpm dev`, `pnpm start`…). |
| **Runtime registry** | `apps/daemon/src/runtimes/registry.ts` — danh sách agent CLI được hỗ trợ; mỗi runtime có một def trong `runtimes/defs/`. |
| **dsh profile** | Cài đặt dưới `~/.dsh/profiles/<name>/`. OpenDesign dùng profile tên `open-design`, chứa plugin `@open-design/dsh-runtime` — đây là "component kết nối" giúp OpenDesign nói chuyện với dsh. |
| **`dsh-profile-jsonl`** | Định dạng stream JSONL mà runtime def quy ước để parse các sự kiện (thinking, text, usage…) từ tiến trình dsh. |

Chi tiết kiến trúc: [docs/architecture.md](architecture.md) · contract adapter:
[docs/agent-adapters.md](agent-adapters.md).

## 3. Điều kiện tiên quyết (đã kiểm tra trên máy này)

| Hạng mục | Yêu cầu | Máy này |
|---|---|---|
| Node.js | `~24` (bắt buộc, do `engines` trong `package.json`) | ✅ **v24.19.0** |
| pnpm | `10.33.x` (pin qua `packageManager: pnpm@10.33.2`; dùng Corepack) | ✅ **10.33.2** |
| `dsh` CLI | Bản chính thức của DeepSeek Harness; phải có trên PATH | ✅ **0.1.1-rc.2** (tại `~/.npm/_npx/…/node_modules/.bin/dsh`) |
| Đĩa trống | ~50 GB (node_modules + pnpm store + Electron) | ✅ |
| Network | npm registry + GitHub releases (Electron prebuilt) | ✅ |

> ⚠️ **Lưu ý [`/usr/bin/od`](https://man7.org/linux/man-pages/man1/od.1.html):** trên
> Linux, `od` mặc định là lệnh octal-dump của hệ thống, KHÔNG phải OpenDesign CLI.
> Khi dùng CLI ngoài UI, luôn gọi bằng đường dẫn tuyệt đối `apps/daemon/dist/cli.js`.

## 4. Các bước thực hiện

### Bước 1 — Kích hoạt pnpm qua Corepack

Repo pin pnpm qua `packageManager`, nên chỉ cần Corepack:

```bash
corepack enable
corepack pnpm --version   # phải in ra 10.33.2
```

- Corepack tạo shim `pnpm` vào thư mục bin của Node đang dùng (`…/node_modules/corepack/dist/pnpm.js`).
- ⚠️ **Đã gặp trên máy này:** trước khi cấp quyền đầy đủ, Corepack báo
  `EROFS: read-only file system` khi ghi cache vào `~/.cache/node/corepack`
  (sandbox của môi trường chặn ghi ngoài workspace). Cách né khi gặp sandbox:
  trỏ `COREPACK_HOME` (và `--store-dir`/`--cache-dir` của pnpm) vào một thư mục
  trong workspace. Không cần nữa khi đã có quyền ghi `~/.cache`.

### Bước 2 — Cài dependencies toàn workspace

```bash
pnpm install
```

Kết quả quan sát được:

- Hoàn tất trong **~2 phút** (`Done in 2m 1.8s using pnpm v10.33.2`).
- `postinstall` của repo tự build nhiều package nội bộ (daemon, tools-dev,
  tools-pack, tools-serve, `packages/dsh-runtime`…) — tức là **không cần chạy
  build riêng cho các package đó**, `dist/` đã có sẵn.
- Native module `better-sqlite3` (daemon) được build cho Node 24; kiểm tra nhanh:

  ```bash
  cd apps/daemon && node -e "const db=require('better-sqlite3'); const d=new db(':memory:'); d.exec('create table t(a)'); console.log('daemon sqlite ok')"
  ```

  (> Chạy **từ trong thư mục package** — pnpm không hoisted mọi thứ lên root, nên
  `require('better-sqlite3')` từ thư mục gốc sẽ báo "Cannot find module". Đó là bình thường.)

- ⚠️ **Cảnh báo ignored build scripts:** pnpm báo
  `Ignored build scripts: @google/genai@1.52.0, node-pty@1.1.0` (chính sách
  `onlyBuiltDependencies` của repo). Nếu sau này cần `node-pty` (terminal) hoặc
  các script đó, chạy `pnpm approve-builds` để cho phép — chưa ảnh hưởng tới tích hợp này.

### Bước 3 — Khởi động daemon + web

Điểm vào lifecycle duy nhất là `tools-dev`. Dùng port cố định để dễ trỏ tới:

```bash
pnpm tools-dev run web --daemon-port 17456 --web-port 17573
```

- `tools-dev` khởi động daemon trước rồi truyền port cho web; `apps/web/next.config.ts`
  rewrite `/api/*` sang port daemon.
- Kiểm tra daemon: `curl http://127.0.0.1:17456/api/health` →
  `{"ok":true,"version":"0.20.3"}`.
- Web (Next dev server) cần **warm-up** ở request đầu (~60s compile), sau đó nhanh
  (HTTP 200 trong ~0.3s).

| Service | URL |
|---|---|
| Web UI | http://127.0.0.1:17573 |
| Daemon API | http://127.0.0.1:17456 |

Dừng khi cần: `pnpm tools-dev stop` · trạng thái: `pnpm tools-dev status` ·
log: `pnpm tools-dev logs`.

### Bước 4 — Tích hợp DeepSeek Harness (bước chính)

Lệnh một dòng, chạy bằng CLI daemon đã build (không dùng trần `od`):

```bash
node apps/daemon/dist/cli.js agent setup deepseek-harness \
  --daemon-url http://127.0.0.1:17456 --json
```

Kết quả: `{"ok":true,"packageVersion":"0.1.0"}` và agent được phát hiện
`available: true` (xem Bước 5).

**Điều gì xảy ra bên trong** (đọc từ `apps/daemon/src/agent-companion-setup.ts`):

1. **Detect** — daemon chạy `dsh --version` và probe `dsh --profile open-design --probe`;
   trước khi cài, profile chưa tồn tại nên "chưa compatible".
2. **Đóng gói component kết nối** — daemon build `@open-design/dsh-runtime`
   (`pnpm --filter @open-design/dsh-runtime build`) rồi `pnpm pack` ra tarball,
   tính SHA-256 và ghi `manifest.json` (cơ chế verify integrity).
3. **Stage** — tarball được ghi vào thư mục profile
   `~/.dsh/profiles/open-design/.open-design/<sha256>.tgz`.
4. **Cài plugin vào dsh** — daemon chạy `dsh plugin --profile open-design add .open-design/<sha>.tgz`;
   dsh tạo profile `open-design` (package.json, cordis.yml, node_modules, pnpm-lock…).
5. **Re-probe** — chạy lại probe; nếu pass thì báo `action: "installed"` (hoặc `"repaired"`
   nếu profile đã có sẵn, `"already-compatible"` nếu không cần làm gì).

> Vì bước này ghi vào `~/.dsh/` (ngoài repo), một môi trường sandbox sẽ chặn
> và cần cấp quyền — trên máy này đã chạy với full access.

### Bước 5 — Xác minh runtime

Vòng probe độc lập (không cần daemon):

```bash
dsh --profile open-design --probe
# {"v":1,"type":"probe","runtime":"open-design","protocol_version":1,
#  "plugin_version":"0.1.0","capabilities":{"session_resume":true,"session_cancel":true,"structured_events":true}}

dsh --profile open-design --models
# {"v":1,"type":"models","runtime":"open-design",
#  "models":[{"provider":"nine-router","provider_name":"nine-router","id":"deepseek-v4-flash",...}]}
```

Qua daemon (API mà chính web UI dùng để hiển thị danh sách runtime):

```bash
curl http://127.0.0.1:17456/api/agents
```

→ runtime `deepseek-harness`: `"available": true`, `"modelsSource": "live"`,
models `["default", "nine-router/deepseek-v4-flash"]`, path trỏ tới `dsh` thật.

⚠️ **Cảnh báo "untested-version":** bản dsh ở máy này là `0.1.1-rc.2`, trong khi
runtime def khai báo `supportedVersions: ['0.1.0-rc.6']`
(`apps/daemon/src/runtimes/defs/deepseek-harness.ts`). Sau khi kiểm tra
`apps/daemon/src/runtimes/detection.ts`: version ngoài danh sách chỉ tạo
**diagnostic cảnh báo, không chặn availability** — đã chứng minh bằng smoke test ở Bước 6.

### Bước 6 — Smoke test end-to-end

Tạo project rồi gửi một chat tối thiểu qua `/api/chat`:

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

Chuỗi SSE quan sát được (bằng chứng vòng lặp hoàn chỉnh):

```
event: start     → runId, agentId "deepseek-harness", streamFormat "dsh-profile-jsonl",
                   model "nine-router/deepseek-v4-flash"
event: agent     → type "status" "working", sessionId "od-…"   (native session bắt đầu)
event: agent     → type "thinking_start" / "thinking_delta" …  (structured thinking)
event: agent     → type "text_delta" "OK"                      (kết quả)
event: agent     → type "usage" provider "nine-router" model "deepseek-v4-flash"
                   input_tokens 26147, output_tokens 38
event: diagnostic→ runtime_close, rpc_close_reason "exit_0", status "succeeded"
event: end       → code 0, status "succeeded", artifactCount 0
```

Ngoài ra còn thấy sự kiện `native_session_recovery` (đầu tiên là
`no_recoverable_session`, sau khi spawn là `captured_not_resumed`) — tức cơ chế
**session resume** của runtime dsh được bật (`resumesSessionViaProfileStdio`,
`capturesSessionIdFromStream` trong def).

## 5. Trạng thái cuối cùng (verified)

| Thành phần | Trạng thái |
|---|---|
| pnpm 10.33.2 (Corepack) | ✅ |
| `pnpm install` toàn workspace | ✅ 2 phút, kèm cảnh báo approve-builds không chặn |
| Daemon v0.20.3 @ `127.0.0.1:17456` | ✅ `/api/health` OK |
| Web @ `127.0.0.1:17573` | ✅ HTTP 200 |
| Profile `~/.dsh/profiles/open-design` | ✅ plugin `@open-design/dsh-runtime` 0.1.0 |
| `dsh --profile open-design --probe` | ✅ `plugin_version: "0.1.0"` |
| `dsh --profile open-design --models` | ✅ `nine-router/deepseek-v4-flash` |
| `GET /api/agents` | ✅ `deepseek-harness available: true` (chỉ cảnh báo untested-version) |
| Chat end-to-end | ✅ model trả lời "OK", `status: succeeded`, `exit 0` |
| Runtime AMR (`vela` 0.0.33) | ✅ `available: true` sau khi sửa symlink (xem [mục 8](#8-sửa-lỗi-follow-up-đăng-nhập-amr--opendesign-cloud-cli-vela)) |

## 6. Cách sử dụng

**Qua UI:** mở http://127.0.0.1:17573 → chọn/ tạo project (đã có project demo
`smoke-test`) → chọn runtime **DeepSeek Harness**, model **deepseek-v4-flash · nine-router**
→ nhập brief (prototype / deck / image / document…) và gửi.

**Qua CLI** (ví dụ, luôn dùng đường dẫn tuyệt đối):

```bash
node apps/daemon/dist/cli.js project list --daemon-url http://127.0.0.1:17456 --json
node apps/daemon/dist/cli.js skills list --daemon-url http://127.0.0.1:17456 --json
node apps/daemon/dist/cli.js agent setup deepseek-harness --daemon-url http://127.0.0.1:17456 --json
```

## 7. Cạm bẫy & mẹo đã học

1. **`/usr/bin/od` shadowing** — đừng gõ trần `od`; dùng `node apps/daemon/dist/cli.js`.
2. **Port mặc định của CLI** là `127.0.0.1:7456` — nếu daemon chạy port khác, phải
   truyền `--daemon-url` (hoặc set `OD_DAEMON_URL`). Thứ tự resolve:
   flag → `OD_DAEMON_URL` → `OD_SIDECAR_IPC_PATH` → `:7456` (`apps/daemon/src/daemon-url.ts`).
3. **Sandbox chặn ghi ngoài workspace** — Corepack (`~/.cache/node/corepack`), pnpm
   store, `~/.dsh` đều dính. Né bằng `COREPACK_HOME` + `--store-dir`/`--cache-dir`
   trong workspace, hoặc cấp quyền đầy đủ.
4. **Cảnh báo untested-version không chặn** — chỉ diagnostic; vòng lặp thực tế chạy được.
5. **Model key** — smoke test chạy được **không cần** export `NINE_ROUTER_API_KEY`
   hay `DEEPSEEK_API_KEY` vào shell. ⚠️ **Chưa xác minh** cơ chế dsh resolve key ở
   bước inference (có thể từ settings/storage của dsh). Nếu gặp lỗi auth kiểu
   "no model API key", hướng dẫn trong `apps/daemon/src/runtimes/auth.ts`: chạy
   `dsh web` → Settings → Models để cấu hình key, hoặc export `DEEPSEEK_API_KEY`
   vào env của tiến trình daemon.
6. **Web dev server cần warm-up** ở request đầu (~60s) — không phải lỗi.
7. **Restart sau khi đổi code daemon**: `pnpm --filter @open-design/daemon build`
   rồi `pnpm tools-dev restart --daemon-port 17456 --web-port 17573`.
8. **Daemon data**: mọi dữ liệu daemon nằm dưới **daemon data root** do daemon resolve
   khi khởi động — quy tắc đường dẫn là bắt buộc đọc ở root `AGENTS.md`
   → mục **Daemon data directory contract**; tài liệu này không quy định lại path đó.
   (Ghi chú quan sát: project `smoke-test` trong lần chạy này có cwd dưới thư mục
   data trong workspace repo, thấy qua event `start` của run.)

## 8. Sửa lỗi follow-up: đăng nhập AMR / OpenDesign Cloud (CLI `vela`)

> Áp dụng ngày 2026-08-24, cùng phiên với phần cài đặt ở trên. Đã xác minh end-to-end.

**Triệu chứng.** Bấm **Sign in** trên web UI bị lỗi 500. Log của tools-dev hiện:

```
[browser] [amr-login] startVelaLogin failed { …
  error: 'vela binary not found; install vela or configure VELA_BIN', ok: false, status: 500 }
```

**Nguyên nhân gốc.** Luồng đăng nhập chạy AMR login (`handleCloudSignIn` →
`handleAmrSignInToContinue` trong `apps/web/src/components/EntryShell.tsx`), khiến
daemon spawn CLI **`vela`** (runtime def `amr`, bin `vela`,
`apps/daemon/src/runtimes/defs/amr.ts`). Daemon resolve binary đó qua `VELA_BIN`
hoặc quét PATH (`apps/daemon/src/runtimes/executables.ts`). Máy này **không có `vela`
trên PATH và không có `VELA_BIN`**, nên cả hai nhánh spawn `direct` và `proxy` đều
fail với `vela binary not found` (`apps/daemon/src/integrations/vela.ts`, `vela-command.ts`).

**Sửa như thế nào.** Không cần cài gì mới — CLI `vela` vốn đã có sẵn trong repo như
dependency của `tools/pack` (`@powerformer/vela-cli@0.0.33` cùng binary
`@powerformer/vela-cli-linux-x64`):

1. Xác minh binary: `tools/pack/node_modules/.bin/vela --version` → `0.0.33`.
2. Link vào `~/.local/bin/vela` (thư mục đã nằm trong PATH của daemon), trỏ thẳng vào
   **entry thật của package**:
   `node_modules/.pnpm/@powerformer+vela-cli@0.0.33/node_modules/@powerformer/vela-cli/bin/vela.cjs`.
   ⚠️ Lưu ý: symlink vào shim `.bin/vela` của pnpm sẽ **fail**, vì shim resolve đường
   dẫn theo chính thư mục của nó — phải trỏ vào entry thật.
3. Restart runtime: `pnpm tools-dev stop`, rồi
   `pnpm tools-dev run web --daemon-port 17456 --web-port 17573`.

**Xác minh sau khi sửa.**

```bash
curl http://127.0.0.1:17456/api/agents
# amr → { "available": true, "version": "0.0.33", "path": "/home/ubuntu/.local/bin/vela", "diagnostics": [] }

curl http://127.0.0.1:17456/api/integrations/vela/status
# {"loggedIn":false,"profile":"prod","configPath":"/home/ubuntu/.amr/config.json", ...}
```

**Việc cần làm bây giờ.** Mở http://127.0.0.1:17573 (hard refresh), bấm **Sign in**
lần nữa — daemon giờ spawn được `vela`, UI sẽ hiện device-activation link; hoàn tất
bằng tài khoản OpenDesign Cloud / AMR (tạo mới nếu cần). Đăng nhập là **tùy chọn** cho
việc dùng local: runtime DeepSeek Harness và các CLI local khác chạy không cần nó.

**Kết quả (2026-08-24, cùng ngày):** luồng đăng nhập giờ hoàn tất —
`/api/integrations/vela/status` trả về `loggedIn: true`
(`sessionState: "authenticated"`, tài khoản AMR đã cấu hình). Lỗi `{}` trước đó hiện
trong UI là do phiên browser cũ (trước khi restart daemon); hard refresh sẽ hiện
trạng thái đã đăng nhập.

**Lưu ý.**
- Link `~/.local/bin/vela` trỏ vào pnpm virtual store; một lần `pnpm install` sau này
  có thể prune/re-hash package. Nếu `vela` "mất", link lại hoặc cấu hình bền hơn ở
  **Settings → Execution mode → AMR agent CLI env** (`VELA_BIN`) — config qua Settings
  có độ ưu tiên cao hơn env kế thừa.
- Danh sách model AMR sẽ rỗng cho tới khi tài khoản đăng nhập xong (live catalog).

### 8.2 Console error: hydration mismatch (browser extension)

**Triệu chứng.** Console web báo lỗi React hydration-mismatch tại
`apps/web/app/layout.tsx:41` (thẻ `<script>` inline khởi tạo theme). Diff cho thấy
node render server mang `src="chrome-extension://lgblnfidahcdcjddiepkckcfdhpknnjh/content/popups-script.js"`
và `__html` bị rỗng, trong khi render client có script theme thật.

**Nguyên nhân gốc.** Một extension trình duyệt (id `lgblnfidahcdcjddiepkckcfdhpknnjh`,
tiện ích chèn script "popups") viết lại node `<script>` khởi tạo theme trước khi React
hydrate — đúng trường hợp "browser extension installed which messes with the HTML" mà
React liệt kê. Trang web vẫn chạy bình thường; lỗi chỉ là nhiễu console do extension
sửa DOM.

**Cách sửa đã áp dụng.** Thêm `suppressHydrationWarning` vào thẻ `<script>` trong
`apps/web/app/layout.tsx` (nhất quán với `suppressHydrationWarning` sẵn có trên
`<html>` và `<body>`). Đã xác minh: `pnpm --filter @open-design/web typecheck` pass và
trang vẫn trả HTTP 200 với script theme còn nguyên. Next dev server tự nạp thay đổi
qua HMR; không cần restart.

**Sửa dứt điểm phía người dùng (khuyến nghị).** Vô hiệu hóa/gỡ extension đó cho site
này (chrome://extensions → tìm `lgblnfidahcdcjddiepkckcfdhpknnjh`). Patch phía app chỉ
che đi cảnh báo; extension vẫn sẽ sửa DOM trên mọi trang nó chạy.

## 9. Tài liệu & mã nguồn tham khảo

- [QUICKSTART.md](../QUICKSTART.md) — quickstart chính thức (one-shot dev, scripts, troubleshooting).
- [docs/architecture.md](architecture.md) · [docs/agent-adapters.md](agent-adapters.md) — kiến trúc & contract adapter.
- `apps/daemon/src/runtimes/defs/deepseek-harness.ts` — định nghĩa runtime dsh (bin, args, probe, models, versionPolicy).
- `apps/daemon/src/agent-companion-setup.ts` — luồng cài/repair component kết nối.
- `apps/daemon/src/runtimes/detection.ts` — logic phát hiện (version gate, compatibility probe, version warning).
- `apps/daemon/src/runtimes/auth.ts` — hướng dẫn/auth failure của từng runtime.
- `packages/dsh-runtime/` — nguồn plugin `@open-design/dsh-runtime` (bị đóng gói vào profile dsh).
- [docs/deepseek-harness-one-click-install.zh-CN.md](deepseek-harness-one-click-install.zh-CN.md) — đường one-click install cho người dùng cuối.