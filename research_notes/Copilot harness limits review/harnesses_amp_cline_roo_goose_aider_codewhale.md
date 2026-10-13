# Long-session management in six coding-agent harnesses (Amp, Cline, Roo Code, Goose, Aider, Codewhale)

Research date: 2026-10-13. Sources are dated where known. GitHub source links marked "default branch" were read via `gh api` on 2026-10-13 and are not pinned to a commit; they should be re-pinned before publication. Codewhale links are pinned to commit 008d21d69fa14d93a36a749b6afedd17d706c8cb (local clone, read-only).

Headline for the spec review: none of the six harnesses was found to rotate sessions on event count, log size, or wall time. Every trigger found is a context-window token or percent threshold, or a provider overflow error. Handoff/new-session features exist (Amp, Cline, Codewhale) but are user- or model-initiated, not automatic.

## Q1. What triggers compaction / condensing / summarization?

### Takeaway
Five of six use a percentage of the context window (or a token soft limit). Amp publishes no threshold. Roo and Goose also add absolute caps or overflow fallbacks. No harness found uses event count, log size, or wall time.

### Cited Findings
- **Codewhale**: production default is `auto_compact_threshold_percent: 80.0` ([settings.rs:542](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/settings.rs#L542)), resolved per route by `compaction_threshold_for_route_at_percent` ([route_budget.rs:456-466](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/route_budget.rs#L456)), clamped to the input-budget ceiling. Call sites: [app.rs:6975](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/tui/app.rs#L6975), [turn_route_plan.rs:251](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/turn_route_plan.rs#L251). A separate constant `DEFAULT_COMPACTION_TRIGGER_PERCENT = 75.0` ([context_budget.rs:49](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/runtime/src/context_budget.rs#L49)) is used only as the subagent parent-window default ([subagent/mod.rs:404](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/tools/subagent/mod.rs#L404)). Fallback absolute threshold `DEFAULT_COMPACTION_TOKEN_THRESHOLD = 102_400` ([models/src/lib.rs:38](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/models/src/lib.rs#L38)). Trigger compares the larger of a local estimate and provider-billed input tokens against the threshold ([compaction.rs:732-769](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L732)).
- **Goose**: `DEFAULT_COMPACTION_THRESHOLD = 0.8` ([goose-context-management lib.rs:32](https://github.com/block/goose/blob/main/crates/goose-context-management/src/lib.rs), default branch); overridden by env `GOOSE_AUTO_COMPACT_THRESHOLD` ([context_mgmt/mod.rs:32-44](https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs#L32)). Absolute cap `DEFAULT_AUTO_COMPACT_TOKEN_LIMIT = 225_000` ([mod.rs:27](https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs#L27)), env `GOOSE_AUTO_COMPACT_TOKEN_LIMIT`. Docs: effective trigger = min(threshold x window, token limit); `0.0` disables ([Goose docs, Smart Context Management, goose-docs.ai](https://goose-docs.ai/docs/guides/sessions/smart-context-management), fetched via WebFetch). Gate: `check_if_compaction_needed` at [mod.rs:243-262](https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs#L243); skipped when `provider.manages_own_context()`.
- **Roo Code**: `autoCondenseContext = true` and `autoCondenseContextPercent = 100` defaults ([Task.ts:3887-3888](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/task/Task.ts#L3887), default branch). Slider bounds `MIN_CONDENSE_THRESHOLD = 5`, `MAX_CONDENSE_THRESHOLD = 100` ([condense/index.ts:109-110](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/condense/index.ts#L109)). Docs mirror: "enabled by default", threshold "default 100%" ([Roo docs, intelligent context condensing](https://roocodeinc.github.io/Roo-Code/features/intelligent-context-condensing)). An older experimental page says disabled by default; source and current docs say enabled, so the older page is treated as stale.
- **Roo overflow fallback**: on a context-window error Roo forces condensing to 75% (`FORCED_CONTEXT_REDUCTION_PERCENT = 75`, [Task.ts:134](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/task/Task.ts#L134); used at [Task.ts:3739](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/task/Task.ts#L3739) and [3786-3787](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/task/Task.ts#L3786)).
- **Aider**: `max_chat_history_tokens = min(max(max_input_tokens / 16, 1024), 8192)` ([aider/models.py:356-358](https://github.com/Aider-AI/aider/blob/main/aider/models.py#L356), default branch); fallback 1024 at [models.py:339](https://github.com/Aider-AI/aider/blob/main/aider/models.py#L339). Trigger is `too_big(done_messages)` ([base_coder.py:1002-1003](https://github.com/Aider-AI/aider/blob/main/aider/coders/base_coder.py#L1002)). Docs: "Soft limit on tokens for chat history, after which summarization begins" ([aider.chat options](https://aider.chat/docs/config/options.html), fetched via WebFetch). Aider does not use a percent of window for history.
- **Cline**: docs say auto-compact runs "when your conversation approaches the model's context window limit"; no percentage or token number is stated ([docs.cline.bot/features/auto-compact](https://docs.cline.bot/features/auto-compact), fetched via WebFetch). Source in the cline repo: `docs/features/auto-compact.mdx` exists (default branch); the trigger constant was not located.
- **Amp**: no threshold found in the manual or the Handoff announcement ([ampcode.com/manual](https://ampcode.com/manual), fetched; "not found").

### Inferences
- The effective default trigger in Codewhale is 80% of the route's input window (settings default), not 75%; the 75% constant is a separate diagnostic/subagent default.
- Roo's 100% default means the slider is effectively a context-limit trigger unless the user lowers it.

### Gaps
- Cline's concrete trigger value (percent or tokens) in source: not found.
- Amp thresholds: not found in any public source.

## Q2. How is compaction performed?

### Takeaway
Four harnesses summarize with an LLM call (Codewhale, Goose, Roo, Cline). Aider summarizes with a separate weak model in a background thread and keeps a verbatim tail. Codewhale and Goose add non-LLM pruning or tool-output truncation before or alongside the summary.

### Cited Findings
- **Codewhale**: before an LLM pass it mechanically prunes old verbose tool results; if projected size drops below 80% of the threshold, it compacts without an LLM call ([compaction.rs:846-872](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L846)). Otherwise it calls the summarizer with a built-in prompt ([compaction.rs:303](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L303)). Keeps last `KEEP_RECENT_MESSAGES = 4` ([compaction.rs:209](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L209)); refuses if fewer than `MIN_SUMMARIZE_MESSAGES = 6` ([compaction.rs:210](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L210)) or if the retained floor cannot get below the trigger ([compaction.rs:886-891](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/compaction.rs#L886)). Summary is stored as a `Compaction` journal entry ([session_tree.rs:26](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/runtime/src/session_tree.rs#L26)).
- **Goose**: `compact_messages` sends the conversation to the provider for summarization ([mod.rs:89](https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs#L89)). Tool-call pairs are summarized separately with a cutoff from `compute_tool_call_cutoff` ([mod.rs:385](https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs#L385)). Docs: two tiers; summarize / truncate / clear / prompt strategies as backup; truncate and clear are CLI-only ([Goose docs](https://goose-docs.ai/docs/guides/sessions/smart-context-management)).
- **Roo Code**: `summarizeConversation` ([condense/index.ts:254](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/condense/index.ts#L254)) uses the active conversation model (docs: "Condensing always uses your active conversation provider/model"). Messages after the last summary are sent on later turns ([index.ts:512](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/condense/index.ts#L512)). Slash-command blocks from the first message are preserved ([index.ts:390-391](https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/condense/index.ts#L390)); docs say so too.
- **Aider**: `ChatSummary` (default `max_tokens=1024`, [history.py:8](https://github.com/Aider-AI/aider/blob/main/aider/history.py#L8)) recursively splits history and keeps a verbatim tail capped at half of `max_tokens` ([history.py:46-67](https://github.com/Aider-AI/aider/blob/main/aider/history.py#L46)). Summarization uses the weak model via `summarize_all` ([history.py:98-123](https://github.com/Aider-AI/aider/blob/main/aider/history.py#L98)). It runs in a thread (`summarize_start` / `summarize_worker`, [base_coder.py:1002-1014](https://github.com/Aider-AI/aider/blob/main/aider/coders/base_coder.py#L1002)). Docs: `--weak-model` "for commit messages and chat history summarization" ([aider.chat options](https://aider.chat/docs/config/options.html)).
- **Cline**: summary keeps "all technical details, code changes, and decisions" and replaces history; reuses prompt cache; falls back to rule-based truncation for unsupported models ([Cline auto-compact docs](https://docs.cline.bot/features/auto-compact)).
- **Amp**: Amp's Handoff announcement says compaction "replaces the context window contents with a summary" and calls it lossy ([ampcode.com/news/handoff](https://ampcode.com/news/handoff), dated 2025-10-23, fetched via WebFetch).

### Inferences
- Goose's and Codewhale's pruning-before-summary pattern means a harness can avoid an LLM call entirely when pruning clears pressure.

### Gaps
- What exactly Goose keeps verbatim beyond the tool-pair cutoff: not verified in source.
- Cline summarizer model: not named in docs.

## Q3. Handoff to a fresh session? Automatic or manual? Event/log/time triggers?

### Takeaway
Amp and Codewhale have explicit handoff features; both are user- or model-invoked. Cline has a `new_task` tool that is approval-gated. No harness found rotates on event count, log size, or wall time.

### Cited Findings
- **Amp**: Handoff replaces compaction. "Amp analyzes the current thread and generates a starting prompt, along with a list of relevant files"; the draft is editable; "Is it automatic: No. You invoke it, for example with `/handoff`" ([ampcode.com/news/handoff](https://ampcode.com/news/handoff), 2025-10-23). Stated reasons: compaction is "lossy"; it "encourages long, meandering threads" and "stacking summary on top of summary". No thresholds, no wall-time or event triggers mentioned.
- **Cline**: `NEW_TASK = "new_task"` is in the tool enum ([apps/vscode/src/shared/tools.ts:22](https://github.com/cline/cline/blob/main/apps/vscode/src/shared/tools.ts#L22), default branch). Cline's blog and docs (search-result snippets, not fully fetched) describe it as ending the session after user approval and starting a new one with a `<context>` block. The fetched `new-task-tool` docs page does not document it, so the sources conflict and only the enum is verified. A 2025 blog says the model proposes it; a `.clinerules` threshold (e.g., 50%) is mentioned in a snippet, not verified.
- **Codewhale**: `/relay` writes `.codewhale/handoff.md` and asks the model to write it ([commands/groups/session/relay.rs:60-80](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/commands/groups/session/relay.rs#L60)). The handoff skill is `invocation: model+user` ([handoff/SKILL.md](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/assets/skills/handoff/SKILL.md)). Auto-compact warning text says `/compact` "saves the original conversation and its model-written handoff before replacing context" ([core/engine/compaction.rs:217](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/core/engine/compaction.rs#L217)). Handoff is written by the model, not deterministic. No automatic rotation found.
- **Roo, Goose, Aider**: no handoff-to-new-session feature found in docs or the sources checked.
- **Event/log/time triggers**: none found in any harness. Codewhale's byte cap is a protocol limit, not a rotation trigger: `MAX_CANONICAL_HISTORY_BYTES = 8 MiB` and `MAX_CANONICAL_HISTORY_ENTRIES = 16_384` ([protocol/src/lib.rs:77-78](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/protocol/src/lib.rs#L77)), used in app-server thread control ([thread_control.rs:63-101](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/app-server/src/thread_control.rs#L63)). Codewhale changelog: "A saved session over 8 MiB can be resumed and forked again" ([CHANGELOG.md:295](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/CHANGELOG.md#L295)).

### Inferences
- The draft spec's "3,000 log events / 32 MiB / 4 h" triggers have no precedent in the six harnesses. Codewhale's closest analogue is a byte cap that refuses or resumes a session, not one that rotates it.

### Gaps
- Goose and Roo handoff/subagent-as-handoff: not checked in depth.
- Amp: whether Handoff's behavior changed after 2025-10: not verified.

## Q4. Storage growth, resume with large histories, crash/slowdown reports

### Takeaway
Codewhale documents journal growth and archiving; the others were not investigated for storage growth.

### Cited Findings
- **Codewhale**: "Long sessions keep less in memory: superseded journal entries beyond a retained window move out of the live session into a per-session archive under `sessions/.journal-archive/`" ([CHANGELOG.md:495](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/CHANGELOG.md#L495)). Same changelog notes unbounded growth remains: "usage fingerprints in session metadata still grow without a bound" and "resume still reads the session file twice". Archive code path was not located in `.rs` files; treat as changelog-only.
- **Codewhale**: compaction appends rather than rewrites (`append_compaction`, [session_tree.rs:175](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/runtime/src/session_tree.rs#L175)); dead entries are pruned via `prune_plan` above a threshold ([session_tree.rs:381](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/runtime/src/session_tree.rs#L381)).
- **Amp, Cline, Roo, Goose, Aider**: no storage-growth or crash reports checked in this pass. Gap.

### Gaps
- No issue-tracker reports with sizes were searched for the five other harnesses. `gh` search for Goose/Roo code returned no hits; not pursued further.

## Q5. Subagents: fresh context? Result caps?

### Takeaway
Codewhale subagents run with their own compaction threshold and cap returned tool results at 1 MiB. Other harnesses were not checked in this pass.

### Cited Findings
- **Codewhale**: subagent has its own threshold derived from parent window ([subagent/mod.rs:404](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/tools/subagent/mod.rs#L404)). `SUBAGENT_TOOL_RESULT_BYTE_CAP = 1_048_576` (1 MiB) ([subagent/mod.rs:478](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/tools/subagent/mod.rs#L478)). `SUBAGENT_SUMMARY_CHAR_BUDGET = 12_000` at [mod.rs:18605](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/tui/src/tools/subagent/mod.rs#L18605); that line sits near a test module, so its production use is unconfirmed.
- **Amp**: manual has no subagent section ("Subagent (Task tool): Not found", ampcode.com/manual).
- **Cline, Roo, Goose, Aider**: not checked.

### Gaps
- Cline `new_task` context carry and Roo `new_task` result return: not verified in source.

## Q6. Provider prompt-cache preservation

### Takeaway
Codewhale tracks cache misses and labels compaction as an expected cache reset. Cline says its summarization reuses the prompt cache. Others not checked.

### Cited Findings
- **Codewhale**: a history reset such as compaction is recorded as an expected miss reason `reset:compaction` ([prefix_cache.rs:1005](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/core/src/prefix_cache.rs#L1005); doc at [prefix_cache.rs:196](https://github.com/Hmbown/CodeWhale/blob/008d21d69fa14d93a36a749b6afedd17d706c8cb/crates/core/src/prefix_cache.rs#L196)). Compaction is therefore a known cache-breaking event.
- **Cline**: summarization "reuses your existing prompt cache" ([docs.cline.bot/features/auto-compact](https://docs.cline.bot/features/auto-compact)).
- **Codewhale** prunes tool results before summarizing; this is a cache-preserving step only if the prefix is unchanged, not verified.

### Gaps
- Goose, Roo, Aider, Amp cache handling: not checked.

## Overall inferences for the draft spec
- No precedent found for event-count, log-size, or wall-time rotation in any of the six harnesses.
- Harnesses that handoff do so on user or model request (Amp `/handoff`, Codewhale `/relay`, Cline `new_task`), and carry a model-written prompt or file, not a deterministic summary.
- Compaction-in-place with a percent-of-window trigger is the dominant pattern; handoff is an alternative to it, not a supplement.
- Codewhale is the closest prior art for the spec's size-based concerns: it caps history bytes (8 MiB) and entries (16,384), but these are protocol limits, not rotation triggers.

## Sources
- Amp: https://ampcode.com/news/handoff ; https://ampcode.com/manual
- Cline: https://docs.cline.bot/features/auto-compact ; https://docs.cline.bot/exploring-clines-tools/new-task-tool ; https://github.com/cline/cline/blob/main/apps/vscode/src/shared/tools.ts
- Roo Code: https://roocodeinc.github.io/Roo-Code/features/intelligent-context-condensing ; https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/task/Task.ts ; https://github.com/RooCodeInc/Roo-Code/blob/main/src/core/condense/index.ts ; https://github.com/RooCodeInc/Roo-Code/blob/main/packages/types/src/global-settings.ts
- Goose: https://goose-docs.ai/docs/guides/sessions/smart-context-management ; https://github.com/block/goose/blob/main/crates/goose/src/context_mgmt/mod.rs ; https://github.com/block/goose/blob/main/crates/goose-context-management/src/lib.rs
- Aider: https://aider.chat/docs/config/options.html ; https://github.com/Aider-AI/aider/blob/main/aider/models.py ; https://github.com/Aider-AI/aider/blob/main/aider/history.py ; https://github.com/Aider-AI/aider/blob/main/aider/coders/base_coder.py
- Codewhale (commit 008d21d69fa14d93a36a749b6afedd17d706c8cb): links inline above.
