# Long-running session management in four coding-agent harnesses

Snapshot date: 2026-10-13. Sources pinned to commits: Codex `90f00bd` (2026-10-13), Gemini CLI `51617c1` (2026-10-12), OpenCode `044d8b7` (2026-10-13, local clone), Claude Code latest release v2.1.296 (2026-10-09; Claude Code source is closed, so its evidence is official docs and issues only).

Evidence tags: **[SRC]** verified in source at the pinned commit; **[DOC]** stated in official docs (fetched 2026-10-13); **[ISSUE]** reported in a GitHub issue (not verified by me); **[GAP]** not found.

## Bottom line for the draft spec

- **No harness I checked rotates to a fresh session on event count, log-file size, or wall-clock time.** Claude Code: none in docs. Codex: rotation is token-based only, and only in an opt-in mode. Gemini: none found. OpenCode: none found; its only time-based reset is a 55-minute WebSocket reconnect at the transport layer, not a session handoff.
- Every harness compacts **in place** when a token threshold is crossed. Codex's opt-in `token_budget` mode is the one exception: it installs a fresh context window without summarizing.
- The thresholds that exist are token-based and derived from the model's window (Claude Code: model limit; Codex: 90% of window; Gemini: 50% of token limit; OpenCode: window minus 10% or 16k reserve).
- Session/transcript size problems are real and reported (Codex and Claude Code issues, with sizes of GB to hundreds of GiB), but the fixes are about storage and resume, not automatic rotation.

---

## 1. Claude Code

### Q1 Compaction trigger
- **Default: compact when the conversation reaches the model's context limit**, with exceptions: Sonnet 4.6 and Opus 4.6 without extended context compact at the 200K boundary; 1M-window models with `CLAUDE_CODE_DISABLE_1M_CONTEXT=1` compact at 200K; models with a native 1M window compact at about 967K tokens by default. [DOC] https://code.claude.com/docs/en/model-config (section "Default auto-compact thresholds")
- Expressed in tokens. Override: `CLAUDE_CODE_AUTO_COMPACT_WINDOW` (plain integer, 100000 to 1000000, capped at model window; takes precedence over `/autocompact`, `--autocompact`, and `autoCompactWindow`). [DOC] https://code.claude.com/docs/en/env-vars ; https://code.claude.com/docs/en/model-config (section "Set the auto-compact window")
- `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE`: sets 1-100 percent of the auto-compact window; can only lower the threshold; applies to main and subagents. The fetched doc text refers to a "default percentage" but does not state its value. [DOC] https://code.claude.com/docs/en/env-vars . **Default percentage: [GAP]**, not stated in the pages I could read.
- Manual: `/compact [instructions]`. [DOC] https://code.claude.com/docs/en/sessions (section "Manage context within a session")
- Provider overflow-error trigger: [GAP], not checked.

### Q2 How compaction is performed
- LLM summarization. `/compact` "reads the conversation it summarizes, so compacting a large context is itself a large request." [DOC] https://code.claude.com/docs/en/costs (section "Why usage climbs in a long session")
- Which model, and exactly what is sent: [GAP]; the "What survives compaction" section was not reachable in my fetches.
- Non-LLM pruning: docs mention "clearing old tool results from context" as a cache-rebuild cause. [DOC] https://code.claude.com/docs/en/costs (prompt cache statistics). Mechanics and thresholds: [GAP].
- Custom instructions: a `# Compact instructions` section in CLAUDE.md is honored. [DOC] https://code.claude.com/docs/en/costs (section "Manage context proactively")
- What is kept verbatim afterwards: [GAP].

### Q3 Automatic fresh session / rotation
- **No automatic handoff to a fresh session found in docs.** Refuted, pending source access (closed source).
- The only automatic session-level choice: on Pro/Max, resuming a session inactive for more than about an hour and over 100,000 tokens opens a dialog offering "Resume from summary" (runs `/compact`), "Resume full session as-is", or "Don't ask me again". The user picks. [DOC] https://code.claude.com/docs/en/sessions (section "Resume from a summary")
- `/clear` starts a new session, but only on user command. [DOC] https://code.claude.com/docs/en/sessions
- Event-count, log-size, or wall-clock rotation: **none found**.

### Q4 Transcript storage and resume
- Format and path: JSONL at `~/.claude/projects/<project>/<session-id>.jsonl`; entry format is internal and may change between versions. [DOC] https://code.claude.com/docs/en/sessions (section "Where transcripts are stored")
- Growth: sessions are saved continuously; retention sweep default **30 days** (`cleanupPeriodDays`). [DOC] https://code.claude.com/docs/en/sessions ; https://code.claude.com/docs/en/costs (section "Analyze your usage patterns")
- Resume loads: docs say resume restores "the full history, including tool calls and results." [DOC] https://code.claude.com/docs/en/sessions (section "What a resumed session restores"). So resume loads the whole transcript, not from the last compaction boundary, per the docs.
- Size-related issues (reported, not verified by me):
  - [ISSUE] #92089 (2026-09-04): second `/compact` in one process re-appends copies of prior history, so the transcript grows quadratically. https://github.com/anthropics/claude-code/issues/92089
  - [ISSUE] #70555: "goes dumb" after long sessions, compaction loses working state. https://github.com/anthropics/claude-code/issues/70555
  - No sizes at which resume crashes were found for Claude Code. [GAP]
- Mitigations shipped: resume-from-summary dialog (above); `CLAUDE_CODE_TRANSCRIPT_LOCAL_GC` limits transcript growth for `-p`/SDK sessions; `CLAUDE_CODE_SKIP_PROMPT_HISTORY`. [DOC] https://code.claude.com/docs/en/sessions (section "Where transcripts are stored")

### Q5 Subagents
- Each subagent has its own context window. [DOC] https://code.claude.com/docs/en/costs (section "Delegate verbose operations to subagents"); only a summary returns to the parent (same section).
- Size caps on returned results: [GAP].

### Q6 Prompt cache
- Docs cover caching behavior: cache lifetime 1 hour on subscription, 5 minutes on API key or cloud provider by default; compaction and tool-result clearing count as "expected rebuilds." [DOC] https://code.claude.com/docs/en/costs (section "Prompt cache statistics" and "Why usage climbs in a long session")
- Stable-prefix mechanics and "actions that invalidate the cache" are on https://code.claude.com/docs/en/prompt-caching, which I did not fetch. [GAP]

---

## 2. OpenAI Codex CLI

### Q1 Compaction trigger
- **Auto-compact limit = 90% of the resolved context window**, capped by any configured `model_auto_compact_token_limit`. [SRC] codex-rs/protocol/src/openai_models.rs L529-540 (`(context_window * 9) / 10`, `min(limit, context_limit)`). https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/protocol/src/openai_models.rs#L529-L540
- **Hard cap: `effective_context_window_percent` default 95** of the window. [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/models-manager/src/model_info.rs#L132 ; computed in codex-rs/core/src/session/context_window.rs L84-86.
- Trigger logic: `token_limit_reached` when scope tokens reach the limit plus `auto_compact_fallback_buffer_tokens`, or when active tokens reach the 95% full-window cap. [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/core/src/session/context_window.rs#L61-L116
- Post-turn compaction: `model_post_turn_compact_threshold_percent` (default value [GAP]). Same file L111-116.
- Scope: `model_auto_compact_token_limit_scope` (`Total` or `BodyAfterPrefix`). Same file L61-80.
- Provider overflow and manual `/compact`: manual compaction exists (`run_manual_compact_task`). [SRC] codex-rs/core/src/compact_token_budget.rs L21-27. Provider-overflow behavior: [GAP].

### Q2 How compaction is performed
- LLM summarization by default. Summarization prompt: `SUMMARIZATION_PROMPT` (from `codex_prompts`). [SRC] codex-rs/core/src/compact.rs L61, L125.
- Recent user messages are kept verbatim up to **`COMPACT_USER_MESSAGE_MAX_TOKENS = 20_000`**. [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/core/src/compact.rs#L63 ; used in `build_compacted_history` (L690-696).
- Summary stored as a summary-prefixed message; history rebuilt via `build_compacted_history` (L388-396). Remote compaction variants exist (`compact_remote_*.rs`), not read in detail.
- Non-LLM mode, `token_budget` (opt-in): "Token-budget compaction skips model/server summarization and installs a fresh context window instead." [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/core/src/compact_token_budget.rs#L16-L20
- Pruning of old tool outputs: a file `compact_retained_tool_outputs.rs` exists; contents [GAP].

### Q3 Automatic fresh session / rotation
- **Yes, but only as an opt-in mode, and token-triggered.** `features.token_budget` has `default_enabled: false` and stage `UnderDevelopment`. [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/features/src/lib.rs#L1903-L1906
- In that mode, compaction calls `start_new_context_window` (compact_token_budget.rs L58). Window IDs advance on each new window (codex-rs/core/src/state/auto_compact_window.rs `advance()`, L77-85).
- Triggers are token-based (above). No event-count, log-size, or wall-clock trigger found.

### Q4 Transcript storage and resume
- Rollout files under `~/.codex/sessions` (location per [ISSUE] #34337). Growth is unbounded by default. [ISSUE] https://github.com/openai/codex/issues/34337 (reports tens to hundreds of GiB, 2026-07-20).
- Resume loads from the **newest compaction that can bound replay**: "Only the newest compaction can bound replay." [SRC] https://github.com/openai/codex/blob/90f00bd89ed3263b4d5993849cccb795f4330449/codex-rs/core/src/session/rollout_reconstruction.rs#L60-L80 . So resume after compaction reads from the boundary, not necessarily the whole file (the rest of the file is still on disk).
- Reported issues:
  - [ISSUE] #38431 (open, 2026-08-13): one large task with a rollout of about **500 MB** on a remote host fails `thread/resume` with "Max payload size exceeded" over SSH WebSocket. https://github.com/openai/codex/issues/38431
  - [ISSUE] #38846: long CLI threads exceed the Responses WebSocket message limit (close 1009) before auto-compaction runs. https://github.com/openai/codex/issues/38846
  - [ISSUE] #29531 (2026-06-23): rollout files duplicate full historical context on resume, reaching **multi-GB** `.jsonl`. https://github.com/openai/codex/issues/29531
- Shipped mitigations: a rollout compression test file exists (`rollout_compress_tests.rs`), and #34337 mentions a zstd setting; I did not verify either. [GAP]

### Q5 Subagents
- [GAP]. Not checked in source at this commit.

### Q6 Prompt cache
- `prompt_cache_key` is passed in the request path: codex-rs/core/src/client.rs (`previous_prompt_cache_key` L405, `current_prompt_cache_key` L422) and codex-rs/core/src/session/session.rs. [SRC] partial read; exact keying scheme [GAP].
- Stable-prefix and `previous_response_id` behavior: [GAP].

---

## 3. Google Gemini CLI

### Q1 Compaction trigger
- **Default threshold: `DEFAULT_COMPRESSION_TOKEN_THRESHOLD = 0.5`**, a fraction of the model's token limit. Compress when last prompt token count >= 0.5 × `tokenLimit(model)`. [SRC] https://github.com/google-gemini/gemini-cli/blob/51617c1bcbcb038291125b092501b4342f4e7f77/packages/core/src/context/chatCompressionService.ts#L42-L45 and L503-507. Configurable via `config.getCompressionThreshold()`.
- Token limit default: **`DEFAULT_TOKEN_LIMIT = 1_048_576`**; model-specific values in `tokenLimit()`. [SRC] https://github.com/google-gemini/gemini-cli/blob/51617c1bcbcb038291125b092501b4342f4e7f77/packages/core/src/core/tokenLimits.ts#L20-L37
- Manual: `force` path (`/compress`). [SRC] chatCompressionService.ts (compress is skipped only when `!force`).
- Provider overflow handling: [GAP].

### Q2 How compaction is performed
- LLM summarization with the same model (`modelConfigKey: { model: modelStringToModelConfigAlias(model) }`). [SRC] chatCompressionService.ts L596, L621.
- **Recent verbatim kept: `COMPRESSION_PRESERVE_THRESHOLD = 0.3`** (the latest 30% of history). [SRC] https://github.com/google-gemini/gemini-cli/blob/51617c1bcbcb038291125b092501b4342f4e7f77/packages/core/src/context/chatCompressionService.ts#L266-L269 ; split at `1 - 0.3` (L559).
- Older part is summarized; "truncatedHistory" suggests a prior truncation pass, contents [GAP].

### Q3 Automatic fresh session / rotation
- **None found.** Compression is in-place. Docs for "checkpointing" describe git-based snapshots for `/restore`, not session rotation. [SRC] https://github.com/google-gemini/gemini-cli/blob/51617c1bcbcb038291125b092501b4342f4e7f77/docs/cli/checkpointing.md (lines 3-11)
- Event-count, size, or time rotation: none found in `chatCompressionService.ts`. Full search of other services: [GAP].

### Q4 Transcript storage and resume
- [GAP]. Not checked in this pass. Known issue reports: [GAP].

### Q5 Subagents
- [GAP].

### Q6 Prompt cache
- [GAP].

---

## 4. OpenCode (anomalyco/opencode)

**Path note:** at commit `044d8b7d2e` the session code is at `packages/core/src/session/`, not `packages/opencode/src/session/` (that path does not exist in this clone). All citations below use the actual path.

### Q1 Compaction trigger
- **Ceiling = window − max(floor(window × 0.10), RESERVE_MIN)**, where `RESERVE_MIN = 16_000` applies when window >= 32,000; otherwise 10%. Auto-compaction is due when `estimateContext >= ceiling`. [SRC] https://github.com/anomalyco/opencode/blob/044d8b7d2e/packages/core/src/session/compaction.ts#L93-L99 and L905-L915 (`calculateCeiling`); due check L242-251.
- Window = `limit.input || limit.context`. Unknown window: never auto-triggers (L222-223); `UNKNOWN_WINDOW = 200_000` is used only to size the compaction request.
- Config overrides: `buffer` (tokens kept free, replaces 10% rule) and `keep`. Initial settings: `{ auto: true, keep: 15_000 }` (L195-197). [SRC] https://github.com/anomalyco/opencode/blob/044d8b7d2e/packages/core/src/session/compaction.ts#L195-L197
- Trigger call sites: auto before each step (`packages/core/src/session/runner/llm.ts` L217); on provider overflow (L265, `reason: "overflow"`); manual `/compact` (`session.ts` L439).
- Overflow retry shrink steps: `SHRINK_STEPS = [0.7, 0.5, 0.35]` (compaction.ts L95).
- Token estimate: `chars / 4` (packages/core/src/util/token.ts L3-5), but overridden by provider-measured usage at the last assistant message when available (`estimatePrompt`, compaction.ts ~L880-900).

### Q2 How compaction is performed
- LLM summarization with the `compaction` agent's model if configured, else the session model (compaction.ts L213-220). Summary uses a fixed template (Objective, Requirements, Decisions, Work State, Next Move, Relevant Files, Important Context), L104-135.
- **Verbatim recent kept: `keep` = 15,000 tokens** of the newest whole user/assistant exchanges, stored as text beside the summary (`splitConversation`/`recentStart`, compaction.ts L699-740).
- Tool outputs in the summarization transcript are truncated to **1,250 characters** (`TOOL_OUTPUT_MAX_CHARS`, L100; used L808-814).
- Native compaction for providers that support it (`compaction.type === "native"`, L213-233).
- Separate prune pass for old tool outputs: **none found** in this commit (grep for `PRUNE_` returns nothing). [SRC]

### Q3 Automatic fresh session / rotation
- **No automatic handoff found.** `session/transfer.ts` exports/imports sessions explicitly (L1-80), not on a trigger. [SRC]
- Time-based: `ROTATE_AFTER_MS = 55 * 60 * 1000` rotates the **WebSocket connection** (not the session) by age. [SRC] https://github.com/anomalyco/opencode/blob/044d8b7d2e/packages/core/src/session/model-transport.ts#L22 and L318-332.
- Step limit: when `agent.steps` is reached the final step is text-only (`runner/max-steps.ts`; `runner/llm.ts` ~L224-230). Not a rotation.
- Event-count or log-size rotation: none found.

### Q4 Transcript storage and resume
- Session messages stored in a database (`packages/core/src/database/`, drizzle); transcript format and resume loading path: [GAP] (only the transfer export path was read).
- Known size issues: [GAP] (not searched).

### Q5 Subagents
- Subagent = child session with its own id (`packages/core/src/tool/plugin/subagent.ts`). Nested depth default **1** (`experimental.subagent_depth ?? 1`, L129).
- Result returned to parent as `<subagent sessionID="..." state="completed">` + the child's full output text (L255-260). **Size cap: none found** in this excerpt. [GAP for confirmation]

### Q6 Prompt cache
- `prompt_cache_key` sent when caching is enabled; clamped to **64 characters** (OpenAI limit). [SRC] https://github.com/anomalyco/opencode/blob/044d8b7d2e/packages/ai/src/protocols/shared.ts#L40-L56 ; used in open-responses.ts L782-798.
- Incremental `previous_response_id` continuation on WebSocket: [SRC] https://github.com/anomalyco/opencode/blob/044d8b7d2e/packages/ai/src/protocols/open-responses-continuation.ts#L18 and L135-149.
- Tool definitions kept on the final step "to preserve the provider's cached prefix" (runner/llm.ts, comment near `toolChoice: stepLimitReached ? "none"`). [SRC]

---

## Cross-harness summary table

| Harness | Trigger (default) | Summarizer | Verbatim tail kept | Auto fresh session | Transcript |
|---|---|---|---|---|---|
| Claude Code | Model context limit (200K / ~967K / window by model); env overrides | LLM (model/prompt: GAP) | GAP | None in docs (manual `/clear`; resume-from-summary is user choice) | JSONL, full load on resume, 30-day retention |
| Codex CLI | 90% of window; full-window hard cap 95% | LLM (`SUMMARIZATION_PROMPT`) | Recent user msgs ≤20,000 tokens | Opt-in `token_budget` (default off), token-triggered | Rollout files, resume from newest compaction; 500 MB+ issues |
| Gemini CLI | 50% of token limit (1,048,576 default) | LLM (same model) | Latest 30% of history | None found | GAP |
| OpenCode | Window − 10% (or 16k reserve) | LLM (`compaction` agent) | 15,000 tokens; tool output cut to 1,250 chars | None found (55-min WebSocket reconnect only) | SQLite (path/resume: GAP) |

## Gaps to close if the report needs them
- Claude Code: `CLAUDE_AUTOCOMPACT_PCT_OVERRIDE` default; overflow trigger; summarizer model and kept content; subagent result cap.
- Codex: post-turn threshold default; `token_budget` rollout; `compact_retained_tool_outputs` contents; subagent behavior; rollout compression setting.
- Gemini: transcript storage and resume; checkpointing details beyond `/restore`; subagents; prompt caching.
- OpenCode: transcript resume path; subagent output cap; known issues.
