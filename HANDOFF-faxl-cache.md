# Handoff: mcode fixes for faxl-proxy integration

Working branch: `fix/stable-agent-context-prefix`, three commits, all pushed
to `origin` (`Gamakon/minimax-code`). Working tree should be clean — check
`git status` on reopening in case anything changed since this was written.

Old checkout `/Users/andrewmorgan/Dev/gamakon/minimax-code` is retired —
this directory (`minimax-code-gamakon`) is the live one, cloned fresh from
the actual `Gamakon/minimax-code` fork (the old one was pointed at a
personal `andrew-gamakon/minimax-code` mirror by mistake). Nothing in the
old directory is needed; it can be deleted.

## Context: why this exists

Running `mcode` (this repo, MiniMax Code CLI) against a local model server
(`faxl-proxy`, a Gamakon caching proxy for Apple-silicon LLM inference,
`~/Dev/gamakon/faxl-mac-community`) instead of MiniMax's cloud API. faxl's
whole value is prefix-caching: it saves the model's KV/recurrent state keyed
on an exact byte-for-byte prefix match, so a repeated prompt prefix skips
re-computation. See `faxl-mac-community/AGENTS.md` for faxl's own operating
instructions.

`mcode` is configured as a custom BYOK provider in `~/.minimax/config.yaml`
(`custom_provider.faxl`, `api: openai-completions`, currently pointed at
`http://127.0.0.1:8767/v1`). The model entry is named `faxl`/`faxl` rather
than a real model id, because faxl serves one model at a time and ignores
whatever model name the client sends — it's just a label for mcode's picker.

## Four bugs found this investigation, across both faxl and mcode

### 1. Tool-call argument double-encoding — FIXED, on faxl's side, not here

faxl's `<tool_call>` parser was re-`json.dumps()`-ing an already-correct
`arguments` string, producing a JSON string wrapping a JSON string. mcode
stored the malformed tool_call and replayed it as history on the next turn,
surfacing as mcode's generic "Conversation history could not be safely
updated. Please retry." error. Confirmed via `FAXL_SNIFF` raw traffic
capture that this was faxl's bug, not mcode's. Fixed upstream in faxl-core
`750c2cd`, synced to faxl-mac-dev `b13691d`. Verified against a live
rebuilt proxy: single/two/three-chained tool-call `mcode exec` sessions all
passed. Closed.

### 2. Volatile SESSION ID / date break prefix caching — FIXED here, commit `2bd6ff1`

mcode injects a `<system-reminder><agent-context>...` block ahead of every
user message, every turn. `YOUR SESSION ID` (fresh per mcode session) and
`date` (regenerated every turn) sat near the *top* of that block, ahead of
stable fields — so the cacheable prefix diverged at nearly the first byte on
every fresh `mcode exec` invocation, even for near-identical prompts.
Confirmed via `FAXL_SNIFF` + faxl `/metrics`: two near-identical `exec`
calls both came back 100% cache miss, pre-fix.

Fix: moved `YOUR SESSION ID` and `date` to the very end of both
`buildAgentContextBlock` (turn 1) and `buildSlimAgentContextBlock` (turn
2+) in `packages/agent-modules/system-reminder/src/blocks.ts`, right before
the closing `</agent-context>` tag. No other field's position changed.
Confirmed via repo-wide search that nothing else depends on field order/
position in this block.

**Not faxl-specific** — this bug affects any prefix-caching backend behind
mcode (vLLM, SGLang, Anthropic prompt caching, anything matching on a
byte-identical shared prefix), not just faxl/Granite.

**Known gap, not yet investigated:** this only reorders fields *within* the
`agent-context` sub-block. Other turn-1-only/conditional providers
registered after it in the chain (`peersUpdateProvider`,
`memoryTopicsProvider`, `cliSunsetMemoryNoticeProvider`,
`skillEvolutionChannelsProvider` — see
`packages/agent-modules/system-reminder/src/providers.ts:826-853`) could
also carry per-turn volatility ahead of the user's real prompt text. Worth
a full `FAXL_SNIFF` byte-diff of two separate sessions' first-user-message
prefix to confirm the *only* difference left is the tail SESSION ID/date.

### 3. "Thinking Off" label lied — actual request kept thinking on — FIXED here, commit `4638e7c`

mcode's TUI status bar said "Thinking Off," but for BYOK formats whose
upstream chat template defaults to thinking ON unless told otherwise
(`qwen-chat-template`, `qwen`, `zai`, `deepseek`, `together`), mcode sent
*no* thinking-related field at all when the toggle was off — because
`resolveConfiguredThinkingProtocol` treated "toggle off, no effort
selected" as "nothing to compute" and returned `undefined` unconditionally.

Confirmed on a live proxy (faxl + Qwen3.8-Flash-Next,
`compat.thinkingFormat: qwen-chat-template`): `enable_thinking=true` on the
wire reproduces the exact symptom (visible reasoning text + a bare,
opener-less `</think>` tag — this model's chat template puts the *opening*
`<think>` in the prompt, not the completion, so that's the normal shape of
a thinking-ON response, not truncation); `enable_thinking=false` fixes it
cleanly. faxl-core verified this with direct measurement against the live
model/template and ruled out `FAXL_MAX_TOKENS_DEFAULT` truncation as a
cause.

Fix: `resolveExplicitThinkingOffProtocol` in
`packages/local-runtime-v2/src/service/model-system/resolution/local-model-resolver.ts`
builds an explicit off-signal `requestPatch` (e.g.
`{ chat_template_kwargs: { enable_thinking: false, preserve_thinking: true } }`
for `qwen-chat-template`) for formats that need one, instead of returning
`undefined`. Routes through the existing `thinkingRequestPatch` →
`patchRequestPayload` mechanism, which already bypasses pi-ai's own
`model.reasoning` gate — same mechanism `resolveDisabledThinkingProtocol`
and the MiniMax M3 on/off control already use. 6 new regression tests
added in `local-model-resolver.test.ts`.

### 4. mcode unidentifiable to BYOK providers — FIXED here, commit `e2a9dca`

`buildLocalProviderHeaders` only set `User-Agent` for managed providers; a
custom/BYOK provider like faxl got whatever Node's default HTTP client UA
was, which faxl's console labels "other." faxl keys its client-label
allowlist off a `User-Agent` substring match (closed allowlist, privacy
boundary — never raw UA passthrough), so this client was never
identifiable in faxl's request log.

Fix: default `User-Agent` to `"faxleet"` for any provider when the config
didn't already set one explicitly (checked case-insensitively, so a
user-configured `User-Agent` in `custom_provider.*.options.headers` still
wins). **My first draft of this was buggy** — it unconditionally overwrote
any configured `User-Agent`, which a regression test caught immediately
(and which also broke the existing OpenCode Go identity tests, which
depend on `User-Agent` being absent/overridable on the way in). Fixed and
re-verified; 2 new regression tests added.

**Needs a matching change on faxl's side**, not done here: add `"faxleet"`
to `_CLIENT_LABELS` in `faxl-core/python/engine/faxl_proxy.py`, then vendor
to `faxl-mac-dev`. Andrew confirmed with faxl-core (session
`faxl-core [511fb7]` at the time) to go ahead — check whether that's landed
before expecting "faxleet" to actually show up in the console.

## faxl-side fixes relevant to retesting (not ours to do, but affect our test results)

- **Checkpoint-flooring bug** (faxl-core, branch `feat/qwen4-exp` at the
  time, commit `6afcb47` in faxl-core / vendored to faxl-mac-dev): short
  conversations (under ~256 tokens) never wrote a checkpoint at all due to
  a grid-flooring calculation, so every turn of a short chat re-prefilled
  from scratch. This explains some of the 40–118s cold request latencies
  seen during testing — not a thinking or caching-logic bug on mcode's
  side. Should already be fixed; confirm before re-testing caching.
- **Reasoning-content cache divergence** (faxl-core, branch
  `feat/chat-streaming-stop`, not confirmed committed as of this writing):
  faxl's settle was caching the conversation WITH the `<think>` reasoning
  block still in it, but reasoning templates drop `<think>` from replayed
  history — so the cached prefix diverged from every future turn at the
  first assistant reply whenever thinking was on. Relevant to us because
  faxl's `reasoning_config` is available by default even when the client
  toggles thinking off elsewhere. Re-run cache tests once this lands.

## What's NOT yet done — the actual remaining work

1. **No live end-to-end verification of any of the three mcode-side fixes
   together**, in a real `mcode` session against a live, correctly-configured
   faxl-proxy. Everything above is verified by direct code tracing,
   targeted live probes against faxl directly (not through mcode), and unit
   tests — but not by actually running the fixed `mcode` build against
   faxl-proxy end-to-end since all three fixes landed together.
2. **PR not opened.** Branch is pushed; PR has been deliberately held until
   live verification above is done.
3. **`pnpm verify` (full profile) not run.** Only `pnpm typecheck` and the
   `capability` vitest suite have been run directly; `AGENTS.md` asks for
   the full `pnpm verify` profile on a clean, committed tree before a PR.
4. The "other turn-varying providers in the prefix" gap noted under bug #2
   is still open.
5. Confirm faxl-core actually added `"faxleet"` to `_CLIENT_LABELS` (bug #4)
   before expecting it to show up.

## How to pick this up

```bash
export PATH="/opt/homebrew/opt/node@24/bin:$PATH"   # system Node is 23.x, outside the supported range
cd /Users/andrewmorgan/Dev/gamakon/minimax-code-gamakon
git status                                           # should be clean, on fix/stable-agent-context-prefix
curl -s localhost:8767/health                        # confirm faxl-proxy is up before touching it — it's shared
```

Then, with a genuinely fresh build (`pnpm install --frozen-lockfile && pnpm build` if `dist/` is stale):

1. Restart faxl-proxy with `FAXL_SNIFF=~/.faxl/logs/sniff.jsonl faxl-proxy` for raw traffic visibility (needs
   an actual restart; it's read once at import). Check with whoever's using the proxy first — it's shared
   across multiple sessions on this machine.
2. Run two **separate** `mcode exec` invocations (fresh process each) with near-identical prompts and confirm
   the second gets real cache reuse via `curl -s localhost:8767/metrics`.
3. Run a real agentic multi-tool-call `mcode exec` (e.g. "list the files here and tell me what this project
   is") and confirm no "Conversation history could not be safely updated" error.
4. Check the faxl console/request log for the client column — should read "faxleet" if their side landed too.
5. Toggle thinking off in the TUI (or rely on `thinking_config.default_value: 'false'` in `~/.minimax/config.yaml`)
   and confirm via `FAXL_SNIFF` that the response has no reasoning content / no dangling `</think>`.
6. If all four pass: `pnpm verify`, then open the PR against `Gamakon/minimax-code` `main`.

## Environment notes

- System Node is v23.10.0, outside this project's supported range
  (`>=22.19 <23 || >=24.2 <27`). `node@24` is installed via Homebrew as a
  keg-only formula (doesn't touch the global `node` symlink) — always
  prefix commands with `export PATH="/opt/homebrew/opt/node@24/bin:$PATH"`.
- `~/.zshrc` has an `mcode` alias pointing at this directory's `dist/cli.js`
  with the Node 24 path prefix already baked in — just run `mcode` in a new
  shell.
- `~/.minimax/config.yaml`'s `faxl` custom provider has been hand-tuned
  during this investigation (generalized model label, raised output token
  limit, `compat.thinkingFormat: qwen-chat-template`) — all deliberate,
  documented inline in the YAML's own comments. Don't revert.
- Per this repo's `AGENTS.md`: feature branch + PR, not direct pushes to
  `main`; `fix/<short-description>` naming (already followed); `pnpm verify`
  before opening a PR.
- Git identity for this repo (and any Gamakon-org repo): must be
  `Andrew Morgan <andrew@gamakon.ai>` via the `github.com-gamakon` SSH host
  alias, not plain `github.com` (which silently authenticates as a dead
  `kaito640` account) and not the `minkymorgan` personal account. Already
  set as repo-local config here (`git config user.name`/`user.email`), but
  verify with `git log -1 --format='%an <%ae>'` after any commit.
- No Claude/Anthropic attribution in commit messages or PR descriptions —
  user's standing instruction for this repo.
