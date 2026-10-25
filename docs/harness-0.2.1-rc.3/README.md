# Harness 0.2.1-rc.3 compatibility (fork build)

Upstream `0.1.26` enumerates host targets up to `0.2.1-rc.2`. The `0.2.1-rc.3` core rejects that package before activation — the failure mode reported in
[upstream #277](https://github.com/NanmiCoder/dsh-agent-teams/issues/277) and auto-filed by the plugin hub for every user on an rc.3 desktop build. This fork declares rc.3 as the recommended target.

## Change surface

| Layer | Change |
| --- | --- |
| `compatibility.json` | `0.2.1-rc.3` added as `recommended`; `0.2.1-rc.2` demoted to `legacy`; `recommendedHost` moved to rc.3. |
| `package.json` peers | `0.2.1-rc.3` prepended to all 21 DSH ranges; every previously declared target is retained. |
| `package.json` dev cohort | `@deepseek-ai/dsh` and 24 DSH packages pinned to `0.2.1-rc.3`. |
| `pnpm-workspace.yaml` | 305 overrides pinned to `0.2.1-rc.3`, so one exact cohort resolves instead of a mix. |
| Product source | Unchanged: `src/` is byte-identical to `0.1.26`; `lib/` is rebuilt from it. |
| Verification tooling | Windows lane fixes only (see below). |

## Verification — 2026-10-25

Environment: Windows, Node.js 24.20.0, pnpm 12.15.0, target host `@deepseek-ai/dsh@0.2.1-rc.3`.

| Check | Result |
| --- | --- |
| Dependency resolution | 296-package exact rc.3 cohort; `pnpm-lock.yaml` carries 6,178 rc.3 pins and 0 rc.2 pins |
| `pnpm typecheck` (server + client projects) | pass with no source edits |
| `pnpm build` | pass; `scripts/git-artifacts.mjs --check` reports the committed `lib/` matches source |
| `node scripts/verify.mjs` | 187/187 checks pass |
| `node scripts/harness-compat-tdd.mjs` | 19/19 pass, including the managed-activation path rc.3 uses |
| `node --test scripts/*.test.mjs` | 251 pass, 7 fail, 4 skipped — every failure is a fixture that creates a Windows symlink (EPERM without Developer Mode), none touches host compatibility |
| `harness-runtime-verify.mjs --host-version 0.2.1-rc.3` | 10/10 scenarios pass: `lifecycle`, `lifecycle-cold-restore`, `fallback`, `fallback-cold-restore`, `failure`, `captain-idle-wakeup`, `progressive-entry`, `web-approval`, `protocol-compatibility`, `stability` |
| Installation gate | `dsh plugin --profile web add --save-exact <artifact>` in an isolated `DSH_HOME` on the rc.3 CLI installs `0.1.27` and exits 0 |

Runtime scenarios use real Harness sessions, tools and sub-agents with a deterministic model fixture. The plugin's 14 tools, member tool restriction, durable member wake-up, mailbox ordering, task state machine, web approval route and cold restore are all exercised against the rc.3 contracts.

### Host contract spot check

The live rc.3 host still resolves every service the plugin injects — `tools`, `llm`, `subagents`, `systemPrompt`, `agents`, `webServer`/`httpServer` and `workspaceRegistry`/`workspace` — and `ctx.subagents` exposes `startActivation`, which the plugin prefers over the legacy `startContinuable` path. `ctx.subagents`, `ctx.tools.restrict`, `ctx.systemPrompt.section` and the agent `steer`/`inject`/`cancel` surface survive the rc.2 → rc.3 bump, which is why the typecheck passes without edits.

## Windows lane fixes (verification tooling only)

- `scripts/harness-runtime-verify.mjs`: `spawn('npm')` cannot resolve the `npm.cmd` shim (`CreateProcess` appends `.exe` only, and Node refuses to launch batch files without a shell), so package-manager calls route through `%ComSpec%`; the install timeout grows to 45 minutes for a 296-package cohort on a slow link; the fixture environment forwards `LOCALAPPDATA`/`APPDATA`/`USERPROFILE`/`PATHEXT` so native install scripts can run, and pins the pnpm shim directory at the front of `PATH` so the CLI's plugin-install gate resolves the package manager.
- `scripts/harness-runtime-scenarios.mjs`: fixture profile links use a directory junction on Windows instead of a symlink, which needs no privilege, and re-linking replaces a stale junction instead of failing with `EEXIST`.

Run the runner against a fresh report directory: the scenarios assert against their own empty `home/` and `workspace/`, and a reused directory replays previous team state.

## Artifact

- `nanmicoder-dsh-agent-teams-0.1.27.tgz`, 2,174,049 bytes
- sha256 `94d5e4fff0652a24e35fddbe7105a38c8923049114ffac04f5b303778490909e`

## Limits

Real provider credentials, browser interaction with the activity panel, native desktop acceptance and migration of existing user data are outside this coverage. The matrix proves the plugin mounts, installs and runs on rc.3; it does not replace a manual desktop acceptance run.

## Rollback

The change is metadata-only: revert the branch to `main` to restore the rc.2 cohort. Existing team state under `<workspace>/.agent-teams/` is unaffected by the host bump.
