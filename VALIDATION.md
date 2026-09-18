# Validation — bidirectional loop

Development date: 2026-09-06. Tests run in disposable fixtures; production repositories were not built or modified by live smoke tests.

## Automated checks

- `python scripts/validate.py`: active skill frontmatter, local references, both provider manifests and shared-runner presence.
- `python -m unittest discover -s tests -v`: **23 passing tests** with fake CLI executables and real temporary Git repositories, without model calls.
- Codex Skill Creator validator: all three active skills.
- Codex Plugin Creator validator: `.codex-plugin/plugin.json`.
- `git diff --check`.

The contract suite covers both host directions, explicit model selection, read-only reviewer argument construction, success/failure parsing, malformed/empty/incomplete output, a failed turn following successful output, session identity on resume, timeout handling, plan hash invalidation, staged/untracked/deleted change coverage, inspection invalidation and preservation of unrelated work during build resumption.

GitHub Actions is configured for Windows, macOS and Linux. Local results establish Windows behavior; cross-platform CI results must be checked on the PR before merge.

## Live model checks

| Check | Result |
|---|---|
| Fable 5.1 reviews a deliberately broken backup plan through the Codex-host route | REVISE; identified deletion-before-read data loss; valid structured output, coverage and session UUID |
| Same Fable session reviews the revised plan | APPROVED with low-priority advice; exact UUID preserved and new plan hash recorded |
| Astra through npm Codex CLI 0.144.5 | Correctly failed, preserving the server error that this model needs a newer CLI |
| Astra through app-bundled Codex CLI 0.153.4 | REVISE; independently identified the seeded data-loss defect; valid structured output |
| Same Astra session reviews the revised plan | APPROVED with zero findings; exact UUID preserved and new plan hash recorded |
| Approval checks on both revised plans | Passed against the actual plan path and current content |
| Fable and Astra separately implement a tiny addition work order | Each created only addition.py; existing acceptance-check file unchanged |
| Host independently runs `python -B check.py` on both implementations | All three acceptance checks passed for each implementation |
| Fresh Astra inspects Fable's code | APPROVED; new untracked addition.py included in the inspected snapshot |
| Fresh Fable inspects Astra's code | APPROVED; new untracked addition.py included in the inspected snapshot |

Fable runs used Claude Code 2.1.261. The Astra test used an explicit CLI executable path rather than changing the user's global installation. The model selection was explicit in both adapters.

Both delegated builders reported that their proof commands were blocked locally: Claude needed approval in headless mode; Codex's Windows sandbox could not access the Python executable. Neither denial was bypassed. The coordinating host ran the proof independently and observed passing results. A completed build turn is not a verified build; the mandatory host proof step resolved these gaps before final inspection.

The fixture checks exercise transport and obvious-defect detection, not comparative model quality. No claim is made that one pairing is better or that an APPROVED response proves exhaustive correctness. A future benchmark should compare defect recall, false positives, proof results, time and usage on the same tasks, including sound plans.

## Limits

- Live review tests were run on Windows. Automated fake-CLI coverage is configured for all three operating systems.
- CLI versions, account access and permission behavior can change; diagnostics identify the selected executable and requested model.
- Codex's shell sandbox does not constrain external MCP side effects; review existing tool configuration as described in the runtime reference. Claude's adapter instead removes non-reading tools and MCP from the reviewer.
- Structured-output validation can reject broken transport and inconsistent verdicts, but cannot prove a model's findings or claimed coverage.

## Linux (Omarchy G3) — live pass 2026-09-18

Run once on 2026-09-18 (results table at the end; all checks passed, one skipped). Runner invoked from this checkout, Arch-based Omarchy, kernel 7.2. The runbook below is kept as written so the pass can be repeated; one deviation is noted in the table (the Claude reviewer ran on `claude-opus-5`). Fold any further corrections back into this file and `references/runtime.md`.

### Prerequisites

- **Claude Code ≥ 2.1.261** — the live-tested baseline (`skills/claudex-loop/references/runtime.md`). Check:
  ```bash
  claude --version
  claude auth status
  ```
- **Codex CLI ≥ 0.153.4**, app-bundled build — the npm build 0.144.5 exposed the required flags but Astra rejected it outright ("requires a newer version of Codex"; see the runtime reference). Check:
  ```bash
  codex --version
  codex login status
  ```
- **Python ≥ 3.10** (the runner is stdlib-only; `requirements-dev.txt` is only for the test/validate tooling below):
  ```bash
  python3 --version
  ```
- If `codex --version` on PATH resolves to something older than 0.153.4 — plausible on Arch, where an AUR or npm-global package can shadow an app-bundled install — resolve the newer binary's absolute path, verify its version directly, and pass it explicitly on every runner call instead of changing PATH or guessing an install location:
  ```bash
  /absolute/path/to/newer/codex --version
  # then on every review/build/inspect call:
  --cli /absolute/path/to/newer/codex
  ```
  The same `--cli` override works for Claude.

Resolve the runner once and reuse it:
```bash
export RUNNER=/absolute/path/to/skills/claudex-loop/scripts/runner.py   # wherever it's actually installed on the G3 — plugin install or manual copy per README's Install section, never a copy inside the target repo
```

### Disposable fixture

Live smoke tests must use a throwaway repo, never a production checkout (see the runtime reference's compatibility note). Build one with a plan and a proof command in one shot:

```bash
FIXTURE=$(mktemp -d)
git -C "$FIXTURE" init -q
git -C "$FIXTURE" commit -q --allow-empty -m baseline
cat > "$FIXTURE/PLAN.md" <<'EOF'
# Plan: add a tiny greeting function

Goal: add `greet(name)` to greet.py, returning "Hello, {name}!".
Acceptance: `python -B check.py` exits 0.
Proof: python -B check.py
EOF
cat > "$FIXTURE/check.py" <<'EOF'
from greet import greet
assert greet("Linux") == "Hello, Linux!"
print("ok")
EOF
git -C "$FIXTURE" add PLAN.md check.py
git -C "$FIXTURE" commit -q -m "seed fixture"
BASE=$(git -C "$FIXTURE" rev-parse HEAD)
export ARTIFACTS=$(mktemp -d)
```

### Smoke direction 1 — Claude host, Codex reviewer

```bash
python "$RUNNER" roles --host claude
python "$RUNNER" review --host claude --repo "$FIXTURE" --plan PLAN.md \
  --model gpt-6-astra --effort high --artifacts "$ARTIFACTS"
```
Note the printed artifacts directory; read `result.json` from it and keep its path as `RESULT1`. Confirm the approval binds to this exact plan:
```bash
python "$RUNNER" check --host claude --repo "$FIXTURE" --plan PLAN.md --approval "$RESULT1"
```

### Smoke direction 2 — Codex host, Claude reviewer

```bash
python "$RUNNER" roles --host codex
python "$RUNNER" review --host codex --repo "$FIXTURE" --plan PLAN.md \
  --model claude-fable-5-1 --artifacts "$ARTIFACTS"
```
Keep the resulting `result.json` path as `RESULT2`, then:
```bash
python "$RUNNER" check --host codex --repo "$FIXTURE" --plan PLAN.md --approval "$RESULT2"
```

### One delegated build (+ inspect)

```bash
python "$RUNNER" build --host claude --builder codex --repo "$FIXTURE" --plan PLAN.md \
  --approval "$RESULT1" --proof "python -B check.py" --artifacts "$ARTIFACTS"
```
Run the proof independently — never trust the builder's own report:
```bash
(cd "$FIXTURE" && python -B check.py)
```
Inspect with a fresh session from the provider opposite the builder (Claude, since Codex built):
```bash
python "$RUNNER" inspect --host claude --builder codex --repo "$FIXTURE" --plan PLAN.md \
  --base "$BASE" --artifacts "$ARTIFACTS"
```

### Contract tests and the shared validator

Run from the actual repository checkout, not the fixture:
```bash
python -m pip install -r requirements-dev.txt
python -m unittest discover -s tests -v      # expect 28 passing, no model calls
python scripts/validate.py                    # skill frontmatter, references, both provider manifests
git diff --check
```
The Codex Skill Creator and Plugin Creator validators listed under "Automated checks" above have no captured shell command in this repo — they were run through Codex's own skill/plugin tooling for the Windows/macOS passes. Record on the G3 exactly how they were invoked (or that they were skipped) rather than assuming parity.

### Platform notes for `runner.py` (read-only — do not edit here)

- `cli_prefix()` only special-cases `os.name == "nt"`: on Windows, an npm-installed CLI can resolve to a `.cmd`/`.ps1` shim that has to be relaunched through `node` directly to dodge `cmd.exe`'s argument-quoting corruption of the JSON / `-c key="value"` arguments. Linux does not need an equivalent branch — an npm-global or nvm-installed `codex`/`claude` on PATH is a POSIX shebang script (`#!/usr/bin/env node`) that the kernel execs directly, and `subprocess.Popen` handles that natively. Expect no code change here; the runner's own `--version` probe before each real call already fails loud (`CLI version probe failed`) rather than misbehaving silently if exec ever does fail.
- Two things are untested on Linux and worth confirming live rather than assuming parity with macOS:
  - `shutil.which(provider)` needs a literal `codex`/`claude` on PATH. If either CLI on the G3 is installed as an AppImage (as Siril was, per the 2026-09-18 Omarchy as-built note) rather than a conventional package or npm install, it must be symlinked/renamed onto PATH as `codex`/`claude` first, or invoked via `--cli /absolute/path/to/*.AppImage`. An nvm install that relies on a shell function rather than a real PATH entry will also make `shutil.which` return nothing even though the CLI works in an interactive shell — check with `command -v codex` / `command -v claude` in a plain non-login shell, since the runner never sources `.bashrc`/`.zshrc`.
  - The timeout/interrupt path in `execute()` kills the process group with `os.killpg(proc.pid, signal.SIGKILL)`, relying on `start_new_session=True` (POSIX `setsid`) to make the child its own process-group leader. This is standard POSIX behavior and should match macOS, but it has not actually run on Linux. During the G3 pass, deliberately let one review run past a short `--timeout` (or Ctrl-C mid-run) and confirm with `ps` that no orphaned `codex`/`claude` process survives.

### Results (fill in after the live pass)

| Date | Check | Claude Code version | Codex CLI version | Model(s) | Verdict | Latency (elapsed_seconds) | Notes |
|---|---|---|---|---|---|---|---|
| 2026-09-18 | Direction 1 review (Claude host → Codex reviewer) | 2.1.276 | 0.155.0 | gpt-6-astra, effort high | APPROVED, 0 findings | 29.69 | `observed_models` empty (Codex does not return it) |
| 2026-09-18 | Direction 1 `check` (approval) | 2.1.276 | 0.155.0 | — | Approval matches the current plan | — | |
| 2026-09-18 | Direction 2 review (Codex host → Claude reviewer) | 2.1.276 | 0.155.0 | claude-opus-5 requested; observed claude-opus-5 + claude-haiku-4-5 | APPROVED, 2 non-material findings | 16.62 | Opus, not the runbook's claude-fable-5-1: the G3 handoff forbids delegated sessions on the top-tier interactive model. Plumbing check only, so the model is not what is being validated |
| 2026-09-18 | Direction 2 `check` (approval) | 2.1.276 | 0.155.0 | — | Approval matches the current plan | — | |
| 2026-09-18 | Delegated build (host claude, builder codex) | 2.1.276 | 0.155.0 | Codex CLI default (build keeps normal config) | exit 0; added `greet.py`; no commit | 33.41 | |
| 2026-09-18 | Host proof run (`python -B check.py`) | — | — | — | `ok`, exit 0 | — | Run by the host, not taken from the builder's report |
| 2026-09-18 | Inspection (host claude, builder codex) | 2.1.276 | 0.155.0 | claude-opus-5 | APPROVED, 0 findings | 12.79 | Fresh session |
| 2026-09-18 | `python -m unittest discover -s tests -v` | — | — | — | 28 tests OK | 3.5 | Python 3.14.7 |
| 2026-09-18 | `python scripts/validate.py` | — | — | — | Passed | — | `git diff --check` clean |
| 2026-09-18 | Codex Skill Creator / Plugin Creator validators | — | — | — | **Skipped** | — | No captured shell command exists for them; not run on the G3 |
| 2026-09-18 | Linux: CLI discovery from a bare environment | 2.1.276 | 0.155.0 | — | **Fails closed, as designed** | 0.0 | With `env -i PATH=/usr/bin:/bin` the runner stops with "codex is not on PATH" before any model call. Both CLIs are mise installs (`~/.local/share/mise/installs/…`), native ELF binaries reachable only through the interactive shell's PATH. `--cli /absolute/path` works from the same bare environment. Any systemd unit, cron job or hook that calls the runner on this machine must pass `--cli` or set PATH itself |
| 2026-09-18 | Linux: forced timeout leaves no orphan | 2.1.276 | 0.155.0 | gpt-6-astra; claude-opus-5 | **Pass, both providers** | 6.1 / 5.09 | `--timeout 6` and `--timeout 5`: "Run timed out or was interrupted; no approval recorded", and `ps` two seconds later shows no surviving `codex` or `claude` child. `start_new_session=True` + `os.killpg` behaves on Linux as on macOS |
