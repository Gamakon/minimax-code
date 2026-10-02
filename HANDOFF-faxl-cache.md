# Handoff: mcode prefix-cache fix for faxl-proxy

Working branch: `fix/stable-agent-context-prefix` (already created, one commit of
changes staged but not yet committed as of this file's writing — check
`git status` / `git diff` first).

## Context: why this exists

We're running `mcode` (this repo, MiniMax Code CLI) against a local model
server (`faxl-proxy`, a Gamakon caching proxy for Apple-silicon LLM inference,
`~/Dev/gamakon/faxl-mac-community`) instead of MiniMax's cloud API. faxl's
whole value is prefix-caching: it saves the model's KV/recurrent state keyed
on an exact byte-for-byte prefix match, so a repeated prompt prefix skips
re-computation. See `faxl-mac-community/AGENTS.md` for faxl's own operating
instructions.

`mcode` is configured as a custom BYOK provider in
`~/.minimax/config.yaml` (`custom_provider.faxl`, `api: openai-completions`,
`baseURL: http://127.0.0.1:8767/v1`).

## Two bugs found and fixed so far

### 1. Tool-call argument double-encoding (FIXED — on faxl's side, not here)

faxl's `<tool_call>` parser was re-`json.dumps()`-ing an already-correct
`arguments` string from Granite, producing a JSON string wrapping a JSON
string. `mcode` stored the malformed tool_call and replayed it as history on
the next turn, which surfaced as `mcode`'s generic error:

> Conversation history could not be safely updated. Please retry.

This was **not a mcode bug** — confirmed via `FAXL_SNIFF` raw traffic capture
(`~/.faxl/logs/sniff.jsonl`). Fixed upstream in faxl-core `750c2cd`, synced to
faxl-mac-dev `b13691d`. Verified fixed against a live rebuilt proxy: ran
single-tool-call, two-chained-tool-call, and three-chained-tool-call `mcode
exec` sessions, all passing, no regressions. **Nothing to do here** — this
item is closed, documented for context only.

### 2. Volatile `SESSION ID` / `date` fields break prefix caching (FIXED in this branch, NOT YET TESTED)

**The bug:** `mcode` injects a `<system-reminder><agent-context>...` block as
a prefix on every user message (every turn, every session). That block used
to put `YOUR SESSION ID: mvs_<random-hex>` and `date: <current-timestamp>`
near the *top* of the block — ahead of stable fields like
`projectInstructions`. Since faxl's cache match is byte-for-byte on the
prompt prefix, and the session ID is fresh per `mcode` session while the date
changes every turn, **this volatile content sitting near the front of the
very first user message invalidated prefix-cache reuse constantly** — even
between two back-to-back, nearly-identical `mcode exec` invocations. Verified
via `FAXL_SNIFF` capture + faxl's `/metrics`: two near-identical `exec` calls
both came back 100% cache miss.

**The fix (this branch):** moved `YOUR SESSION ID` and `date` to the very end
of both `buildAgentContextBlock` (full, turn 1) and `buildSlimAgentContextBlock`
(slim, turn 2+) in
`packages/agent-modules/system-reminder/src/blocks.ts`, right before the
closing `</agent-context>` tag, leaving every other field's position
unchanged. This maximizes the stable, cacheable byte-prefix length before the
first volatile byte. See the inline comments added at both call sites.

Full trace of the construction/injection path (for anyone who wants to verify
or extend this), in case the fix needs revisiting:

- Block content built in `packages/agent-modules/system-reminder/src/blocks.ts`
  — `buildAgentContextBlock` (turn 1) and `buildSlimAgentContextBlock` (turn 2+).
- Which one runs: `packages/agent-modules/system-reminder/src/providers.ts:193-199`
  (`agentContextProvider`), registered **first** in the provider chain
  (`providers.ts:826`), so its output is always the first sub-block inside
  `<system-reminder>`.
- Outer `<system-reminder>` wrapper assembled in
  `packages/agent-modules/system-reminder/src/service.ts:210-213`.
- Prepend-vs-append decision (reminders go BEFORE the user's actual prompt
  text) is in `packages/local-runtime-v2/src/service/turn-system/agent-host/execution/prompt.ts`
  (`joinUserPrompt`) and invoked in
  `packages/local-runtime-v2/src/service/turn-system/agent-host/execution/executor.ts:506-525`.
- Session ID generated once per session (not per turn) in
  `packages/local-runtime-v2/src/service/session-system/sessions/lifecycle/record-service.ts:261`
  (`mvs_${randomUUID()...}`) — but if `mcode exec` creates a new session per
  CLI invocation (plausible, not fully confirmed), the ID still changes every
  invocation in practice.
- `date` is regenerated every turn in
  `packages/local-runtime/src/memory/local-data-collector.ts:230`
  (`this.input.formatDate()`), called from
  `SystemReminderService.buildReminder()` → `collector.collect()` on every turn.

**What's NOT yet verified / possible follow-up:**

- This fix only reorders fields *within* the `agent-context` block. A prior
  investigation (not yet re-checked after this fix) flagged that other
  turn-1-only or conditional providers registered *after* `agentContextProvider`
  in the chain (`peersUpdateProvider`, `memoryTopicsProvider`,
  `cliSunsetMemoryNoticeProvider`, `skillEvolutionChannelsProvider` — see
  `providers.ts:826-853`) could also vary per turn and sit ahead of the user's
  real prompt text. Since `agent-context` is first in the chain, this fix is
  necessary but might not be fully sufficient if any of those later blocks
  also carries per-turn volatility. Worth re-running the `FAXL_SNIFF` capture
  (see below) after this fix to check the full assembled prefix, not just the
  `agent-context` sub-block.
- **Not yet built or tested.** Needs `pnpm install && pnpm build`, Node
  22.19+/24.2+/25/26 (not 23.x — see below), then a real multi-session
  `FAXL_SNIFF` capture to confirm cache hits actually improve across separate
  `mcode exec` invocations (not just within one session, which already
  worked before this fix).

## How to test this fix

1. Build (see Environment notes below for the Node version gotcha):
   ```bash
   export PATH="/opt/homebrew/opt/node@24/bin:$PATH"
   cd /Users/andrewmorgan/Dev/gamakon/minimax-code-gamakon
   pnpm install --frozen-lockfile
   pnpm build
   ```
2. Make sure `faxl-proxy` is running (`curl -s localhost:8767/health`). To
   capture raw traffic for verification, restart it with sniff logging:
   ```bash
   FAXL_SNIFF=~/.faxl/logs/sniff.jsonl faxl-proxy
   ```
   (needs an actual restart — `FAXL_SNIFF` is read once at import.)
3. Run two **separate** `mcode exec` invocations (new process each time, not
   `--continue`) with near-identical prompts, e.g.:
   ```bash
   node dist/cli.js exec "What is 2+2? Reply with just the number."
   node dist/cli.js exec "What is 3+3? Reply with just the number."
   ```
4. Check `curl -s localhost:8767/metrics` and/or the sniff log — the second
   call should now show a much longer `prefilled_tokens`-from-cache vs.
   `prefill_paid`, where before this fix it was 100% miss both times.
5. Also re-run the known-good multi-tool-call regression cases to make sure
   nothing broke:
   ```bash
   node dist/cli.js exec "List the files in the current directory and tell me what this project is."
   ```

## Environment notes (things that cost time to discover)

- System Node is v23.10.0, which is **outside** this project's supported
  range (`>=22.19 <23 || >=24.2 <27` in root `package.json`). Installed
  `node@24` via Homebrew as a **keg-only** formula (doesn't touch the global
  `node` symlink) — use `export PATH="/opt/homebrew/opt/node@24/bin:$PATH"`
  before building/running from source.
- `mcode` alias in `~/.zshrc` currently points at the OLD checkout
  (`/Users/andrewmorgan/Dev/gamakon/minimax-code`, being deleted). Update it
  to point here once this build is verified working:
  ```bash
  alias mcode='PATH="/opt/homebrew/opt/node@24/bin:$PATH" node /Users/andrewmorgan/Dev/gamakon/minimax-code-gamakon/dist/cli.js'
  ```
- `~/.minimax/config.yaml` already has the `faxl` custom provider configured
  and selected as default. It's been hand-edited a couple of times during
  this investigation (model id generalized to `faxl`/`faxl` since faxl
  ignores the client's requested model name and just serves whatever it's
  running; output token limit raised to 131072; `compat.thinkingFormat:
  qwen-chat-template` set for proper thinking-toggle support). Don't revert
  these — they're deliberate, documented inline in the YAML's own comments.
- This repo (`Gamakon/minimax-code`) is a fork of `MiniMax-AI/minimax-code`,
  currently at v0.6.1 (ahead of the old checkout's 0.5.10). `upstream` remote
  is wired to the public MiniMax repo if you need to diff/sync.
- Per this repo's `AGENTS.md`: use a feature branch + PR, don't push to
  `main`; branch naming is `fix/<short-description>` etc. (already followed
  here); run `pnpm verify` before opening a PR.

## Suggested next steps

1. Build and run the test plan above to confirm the fix actually improves
   cross-invocation cache hit rates.
2. If other turn-varying providers are found in the prefix (see "not yet
   verified" above), apply the same tail-ordering treatment to them.
3. Run `pnpm verify` (or at least `pnpm typecheck` + the relevant vitest
   suite for `packages/agent-modules/system-reminder`) before opening a PR.
4. Open a PR against `Gamakon/minimax-code` `main` from this branch.

## Status log (what's actually been done, kept current)

**Done:**
- [x] Fix applied: `packages/agent-modules/system-reminder/src/blocks.ts`,
  both `buildAgentContextBlock` and `buildSlimAgentContextBlock` — `YOUR
  SESSION ID` / `date` moved to the tail of the block, right before
  `</agent-context>`. No other field position changed.
- [x] `pnpm install --frozen-lockfile` — clean.
- [x] `pnpm build` — clean, v0.6.1, 6277 source files, no errors.
- [x] `pnpm typecheck` — clean, no errors.
- [x] Searched for any test or other source file that depends on
  `agent-context` field order/position — none found (only `blocks.ts` itself
  references `YOUR SESSION ID` anywhere in the repo). So this is a safe,
  self-contained reorder with no known blast radius, but also **no existing
  regression test protects this behavior** — worth adding one (see Test list
  below).
- [x] Patch file generated: `0001-stable-agent-context-cache-prefix.patch`
  (in this directory) — pure diff of the fix, applyable elsewhere with
  `git apply 0001-stable-agent-context-cache-prefix.patch`.
- [x] Confirmed this bug is NOT faxl/Granite-specific. `mcode` injects the
  same volatile-prefix `<agent-context>` block ahead of the user's prompt
  text on every turn, for every provider/model. Any prefix-caching backend
  behind `mcode` — faxl, vLLM, SGLang, Anthropic prompt caching, anything
  matching on a byte-identical shared prefix — would hit the same cache-busting
  problem. It just wasn't visible with providers where hit-rate wasn't being
  separately measured.

**Blocked / not yet done:**
- [ ] Live verification against faxl-proxy — **currently blocked**: the
  proxy is intentionally down on another session, being updated for the
  latest Qwen vision model. Do not start/restart `faxl-proxy` until that
  session confirms it's clear. Check with the user or
  `curl -s localhost:8767/health` first.

## Test list (run once the proxy is back up)

1. **Primary regression test — the actual bug this fix targets.** Two
   separate `mcode exec` invocations (fresh process each, not `--continue`),
   near-identical prompts:
   ```bash
   FAXL_SNIFF=~/.faxl/logs/sniff.jsonl faxl-proxy   # restart needed for sniff
   node dist/cli.js exec "What is 2+2? Reply with just the number."
   node dist/cli.js exec "What is 3+3? Reply with just the number."
   ```
   Check `curl -s localhost:8767/metrics` and the sniff log: the second call
   should show a much longer cached/prefilled-from-cache prefix than before
   this fix (previously: 100% miss on both, confirmed broken pre-fix).

2. **No-regression check on the two previously-fixed tool-call scenarios**
   (faxl-side fix, item 1 above — confirm this mcode-side change doesn't
   reintroduce or interact badly with it):
   ```bash
   node dist/cli.js exec "List the files in the current directory and tell me what this project is."
   node dist/cli.js exec "Use the glob tool to find all files matching 'docs/*.md', then read the first one you find and summarize it in one sentence."
   ```
   Both should succeed cleanly, no "Conversation history could not be safely
   updated" error.

3. **Check the rest of the `<system-reminder>` prefix for other turn-varying
   content**, not just `agent-context`. Capture the full assembled first-user-message
   prefix (via `FAXL_SNIFF`) for two separate sessions and diff them
   byte-for-byte up to where the real user prompt starts — confirm the ONLY
   difference is the tail `YOUR SESSION ID`/`date` lines, not something in
   `peersUpdateProvider` / `memoryTopicsProvider` /
   `cliSunsetMemoryNoticeProvider` / `skillEvolutionChannelsProvider` (see
   `providers.ts:826-853`) that was not addressed by this fix.

4. **Within-session cache behavior still works** (this worked before the fix
   and must keep working): multi-turn tool-call chains within one continuous
   session/TUI run should still hit cache on repeated turns, same as the
   earlier verification pass.

5. **Add an actual regression test** for the field-ordering behavior itself
   (currently nothing in the repo tests this) — e.g. a unit test on
   `buildAgentContextBlock`/`buildSlimAgentContextBlock` asserting that
   `YOUR SESSION ID` and `date` are the last two non-closing-tag lines in the
   output, so a future edit can't silently reintroduce the bug.

6. **`pnpm verify`** (full profile) on a clean, committed working tree before
   opening the PR, per this repo's `AGENTS.md`.
