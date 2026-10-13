# Re-review: revised "Copilot harness task orchestration.md"

Reviewed: the revised spec (1,204 lines, all read; line numbers below are from the current file, which already uses RT1–RT4 for the rotation triggers), `reports/ui-wiring.json` (88 entries), the earlier review `spec_review.md` (F1–F33), `copilot_cli_internals.md`, and the SDK clone at `b19d3ef043ebfeaeb794339173c39ad6c8eb29ca` (read-only; paths below are relative to `rust/src/` unless they say otherwise).

Severity uses the same scale as the first review: **critical** = burns tokens or loses work in normal operation; **major** = likely bug, race or large waste; **minor** = inconsistency or polish.

## 1. Status of F1–F33

Totals: **30 fixed, 3 partly fixed, 0 left open.**

| ID | Status | Spec line(s) | Note |
|---|---|---|---|
| F1 3,000-event trigger | Fixed | 149, 191 | Removed. The reasoning uses the issue sizes. |
| F2 rotate on 2nd compaction | Fixed | 193, 201 | Removed. The handoff is seeded with `summary_content`. |
| F3 unbounded handoff | Fixed | 200-209 | Capped at 8 KiB, with mail sent as summaries only. The parts can still add up to more than 8 KiB (N8). |
| F4 32 MiB has no live source | Fixed | 101, 188, 192, 224, 1134 (V16) | Uses `sessions().get_sizes()`, polled every 60 s. |
| F5 rotation wakes idle agents | Fixed | 194, 197 | Lazy rotation; the 4 h trigger is removed. The memory case of RT4 conflicts with lazy rotation (N6). |
| F6 disconnect while busy | Partly | 196-197, 210, 127 | Waits for a safe point and uses `sessions().close`. There is still no timeout or abort if the safe point never comes, and the drain is paused only after the safe point (N5). |
| F7 queued mail lost on rotation | Fixed | 198, 212 | Withdraw, re-hold, then one merged message. The ordering race is filed as N5. |
| F8 last events lost / expired replay | Fixed | 199, 237-238, 803 | Drains with `wait_ms: 0`; on `expired` it discards the batch, calls `tail()` and emits `Resync`. |
| F9 60 s polling tools | Fixed | 40, 596-602, 694, 699, 715 | One wait, then `wake: true`. `agent_control` has no wait. Repeat calls are refused and counted. |
| F10 nudge on every tool call | Fixed | 590, 610 | Edge-triggered, at most once per turn; also on the failure hook. |
| F11 digest size | Fixed | 608 | At most 10 entries with summaries of ≤ 200 chars. |
| F12 TaskReady herd | Fixed | 491 | Notifies one teammate, then another after 60 s. |
| F13 completion Notify contradiction | Fixed | 589, 592 | Completions are held and batched. The table and the text agree. |
| F14 lease races | Fixed | 321-322, 504 | Renewed from observed liveness and paused during waits on the user. On expiry the task returns only if the session is idle. |
| F15 restart makes new generations | Fixed | 225, 146 | Resumes sessions; held handlers are resolved; RT3/RT4 only on failure. Stale locks go to RT3; `releaseLock` only releases this process's own lock, so not using it is fine. Resume defaults are still unsafe (N3, N15). |
| F16 duplicate retry | Fixed | 553, 580, 225 | Adds an `Unknown` state that is reconciled before any retry. |
| F17 "8 sessions" ambiguous | Fixed | 39, 229, 730, 1131 | 8 running turns and 16 live sessions; spawns queue only on turns. |
| F18 0.70 threshold | Fixed | 42, 155 | `min(0.80, 250K/limit)`. This formula cannot be computed when the session is created (N4). |
| F19 missed SDK APIs | Fixed | 100-102, 128, 177 | Adds compact, getSizes and close. `clear_context` is rejected with a reason; `contextInfo` is not used, which is acceptable. |
| F20 UI routes on events with no envelope | Fixed | 842, 850, 921 | Both events now produce envelopes. Line 921 states that names marked (Copilot) are raw event types. |
| F21 dead blob reference | Fixed | 627, 666 | Results are truncated with a pointer to `task_get` / `read_mail`. `task_list` returns compact rows with a default of 20. |
| F22 summary cap contradiction | Fixed | 571, 580, 705 | 200 chars everywhere. |
| F23 C13 cited decision 2 | Fixed | 966 | Now cites decision 10. |
| F24 V3/V4 already answered | Fixed | 552, 580, 1121-1122 | `expected_prompt` is stored; V3 and V4 are confirmation tests only. |
| F25 V9 partly answered | Fixed | 235, 1127 | `include_ephemeral: true` is now the design. V9 is widened on purpose to cover whether ephemerals are complete. |
| F26 incomplete PermissionDecision list | Fixed | 110 | All variants are listed. |
| F27 ExitPlanModeResult default | Fixed | 211, 508 | Fields are built explicitly, and the handler and plan mode are re-applied on a new generation. |
| F28 which 0.90 source | Fixed | 186 | Uses `post_compaction_tokens / token_limit`. |
| F29 escalation vs busy parent | Partly | 588 vs 41, 965 | 7.2 is fixed, but line 41 and C12 (and JSON C12) still state other rules (N10). |
| F30 baseline unmeasured | Fixed | 627, 1135 (V17) | Per-role subsets and V17. `tool_search` is not considered (optional). |
| F31 LRU dedupe | Fixed | 235, 803 | Persisted cursor, with the LRU only at the seam; expiry handled. |
| F32 worktree / WIP policy | Partly | 883 | WIP is committed only on `agent_stop`. Lease expiry, forced recycle and crash paths do not commit it (N1). There is no worktree cleanup at run end, and the branch name still assumes one task per agent (730). |
| F33 dead variants | Fixed | 546, 703, 504 | `Urgency::Stop` and `note` are removed; `InputRequired` is set only by the harness. |

## 2. New problems introduced or exposed by the revision

| # | Sev | Line | Problem | Fix |
|---|---|---|---|---|
| N1 | major | 227 (with 225, 504, 883) | **A forced planned recycle loses or duplicates work.** After 10 minutes the supervisor recycles even if turns are still running. Resume defaults to `continue_pending_work = false`, so "work still pending on resume is treated as interrupted" (`types.rs:3941-3951`). The spec resumes idle agents lazily and never re-wakes an agent whose turn was cut off. That agent stays idle with an `InProgress` task. After 300 s the lease returns the task to `Pending`, and another teammate claims it without the WIP commit, which happens only on `agent_stop`. | Never kill a running turn for a *planned* recycle, or abort it explicitly and queue an "interrupted; resume T7" directive for the next wake. Decide `continue_pending_work` deliberately. Commit WIP to the branch whenever a lease expires or a task is returned. |
| N2 | major | 226 | **The stall detector recycles the whole CLI on long, quiet tool calls.** "Turn open and no event for 180 s" is true for a 6-minute build or test run that prints nothing, which the lease section (504) expects to happen. A false positive aborts every agent's turn and leads into N1. | Exclude turns with a `tool.execution_start` that has no matching `complete`, and turns with a held handler. Use a `session.error` ack timeout or a cheap RPC ping as the stall signal. |
| N3 | major | 225, 187 | **RT3 may never fire, and a resume can silently start an empty session.** `ResumeSessionConfig` documents that resume "silently start[s] a new session" when the session is not on disk, unless `suppress_resume_event` (wire `disableResume`) is set (`types.rs:3938-3940`). If that happens the agent continues with no context and no handoff, and the harness still records `AgentResumed`. | Set `suppress_resume_event: Some(true)` on every resume and treat the error as RT3. Check that the resumed `session_id` equals the requested one. |
| N4 | major | 155, 911 (D11), 95 | **The compaction threshold formula cannot be computed when the session is created.** D11 takes `token_limit` "from `session.usage_info`", which arrives only after the session exists. `InfiniteSessionConfig` is set on create and resume (`types.rs:2282`, `:3845`). An implementer falls back to 0.80, which is 800K tokens on a 1M window: exactly the #5054 timeout case, which then fires RT1 rotations. | Take the limit before creating the session, from `ProviderConfig.max_prompt_tokens` (`types.rs:1356`) or the model's `max_context_window_tokens` (`types.rs:1757`) or the models list. Re-supply it on resume. Say this in D11. |
| N5 | major | 196-199 | **Rotation ordering race, and no safe-point timeout.** The trigger only sets `rotation_pending`. `set_drain_paused(true)` and holding mail happen in step 2, after the safe point. While the turn runs, user steers and directives keep going into the old session's queue, and Copilot drains them at the turn boundary, so `session.idle` can be delayed again and again. For RT1/RT2 each extra turn runs on a session whose compaction fails and is retried every turn (#4663), which burns tokens. There is also no timeout or abort if the safe point never comes (F6). | When the trigger fires: hold new mail, call `set_drain_paused(true)` and withdraw harness-owned queue items at once. Then wait for idle, with a bounded timeout (for example 5 min) followed by `session.interruptMainTurn` / `abort()`. |
| N6 | major | 188, 225, 227 | **The memory case of RT4 conflicts with lazy resume and lazy rotation, and can cause a recycle loop.** "RSS still ≥ 3 GiB right after a planned recycle" is measured when idle agents have not been resumed yet (225), so it rarely fires at the right time. When it does fire for an idle largest session, rotation is lazy (197), so no memory is freed. RSS climbs again, and the next recycle costs every active agent a cold prefix (227). With the #4686 leak (OOM after 37 min), recycles could come roughly every 30–40 min. | Measure RSS once every live session is loaded. Add a minimum interval between recycles and alert when it is hit. For memory relief on an idle agent, *close* its session (as the 15-minute idle rule does) instead of rotating. |
| N7 | minor | 210, 1191 | `sessions().close` is a client-level RPC. The Rust `Session` object and its subscriptions stay registered; only `disconnect` unregisters them (`session.rs:1331-1332`). Over many generations and idle closes this leaks router entries and handlers. | After `close`, drop the `Session` and its subscription, or call `disconnect` after `close` once V-tested. |
| N8 | minor | 200-207 | The parts of the handoff can add up to more than 8 KiB: summary 4 KiB, last message 2 KiB, checklist up to 30 × 160 B, 40 diff-stat lines, unbounded edited paths, tasks at 300 chars each and 10 mail summaries. No truncation order is given. | Give each part a budget that sums to ≤ 8 KiB, plus a drop order (paths, then diff stat, then checklist). |
| N9 | minor | 201, 415 | After RT1 in a generation that never compacted successfully, `last_summary` is empty and the handoff carries no conversation summary. It is not stated whether `last_summary` carries over from the previous generation. | Carry `last_summary` across generations and say so. |
| N10 | minor | 41, 588, 965 (+ JSON C12) | Three versions of the escalation rule: "idle and not answered within 120 s" (41); "idle or InputRequired … within 120 s of its next wake" (588); "after 120 s without a parent answer" (C12). | Use the wording from 588 everywhere and regenerate C12. |
| N11 | minor | 586, 961 (+ JSON C8 refs) | Stale text: the Stop control "calls `abort()` then `disconnect()`", while 883 and C8 say `sessions().close` once idle. C8 still links `Session::disconnect`. | Fix 586 and replace the C8 link with `sessions.close`. |
| N12 | minor | 16 | The tag legend still says probes "V1–V15"; the probes now go to V18. | Change it to V1–V18. |
| N13 | minor | 788 vs 103, 969 (C16) | C16 promises compaction cost "tokens in, cached, out", but the `Compaction` EventKind has no `cost_cache_read_tokens` field. | Add `cost_cache_read_tokens` (and `cache_write_tokens`, `request_count`, which exist in `CompactionCompleteCompactionTokensUsed`). |
| N14 | minor | 140, 188, 1194, 225 | Two inferences are stated as facts. "The runtime keeps each live session's whole durable event log in memory": the notes say it "appears" so, and that whether durable payloads are freed "is not shown" (`copilot_cli_internals.md` L53, L63). "A resume makes no model call" has no source in the notes. The case for the memory branch of RT4 rests on both. | Tag both [INF] or [UNV] and fold them into V16 (measure RSS and `assistant.usage` events during a resume). |
| N15 | minor | 225 | Resume says only "with the same `ProviderConfig`". `ResumeSessionConfig` also needs tools, handlers, hooks, `infinite_sessions` and the plan-mode handler. Managed settings are "not persisted … must be re-supplied" (`types.rs:3927-3932`). Step 6 of rotation lists these; resume does not. | List the full config on resume, the same as rotation step 6. |
| N16 | minor | 770, ui-wiring.json | The new `AgentResumed` EventKind updates no UI element, so nothing re-renders S6, A2 or C8 on a resume. `Intent`, `AssistantMessage`, `ModelUsage`, `TurnEnded` and `ModelChanged` are still never routed. Line 921 makes that legitimate, but they are dead weight. | Route `agent_resumed` to S6, C8 and A2, or drop it. Optionally remove the unrouted normalized kinds. |

Spec and ui-wiring.json are consistent. A script compared every JSON entry's `element`, `shows`, `source`, `action` and `updates` with the table rows in 10.3 and 10.4. The only differences are the escaped `\|` in S2, H4 and C17. X1 exists only in JSON and in prose at 10.6, which is fine. Every non-Copilot `updates` name is an `EventKind`. The JSON-side stale items are those listed in N10, N11 and N16. I found no remaining 0.70, 32 MiB, 3,000-event, 4 h, two-reader or `include_ephemeral: false` text outside Appendix C's "Before" column and the "removed triggers" list (191-194). The numbers 0.80 / 250,000 / 256 MiB / 3 GiB / 8 turns / 16 sessions / 60 s / 30 s / 8 KiB / 10 entries / 200 chars are consistent across sections 1, 3, 4, 7, 8, D11 and V13.

## 3. Spot-checks of [SV] claims against the clone

| # | Claim (spec line) | Source | Result |
|---|---|---|---|
| 1 | `history().compact()` at rpc.rs L10777 (102) | `generated/rpc.rs:10777` `pub async fn compact` | OK (experimental) |
| 2 | `compact_with_params(HistoryCompactRequest)` at L10806 (102, 177) | `generated/rpc.rs:10806` | OK |
| 3 | `HistoryCompactRequest{custom_instructions, token_limit, trigger}` → `HistoryCompactResult{success, summary_content, tokens_removed, messages_removed, context_window}` (102) | `generated/api_types.rs:9562-9600` | OK. `token_limit` is attribution metadata only. |
| 4 | `sessions().get_sizes()` → `SessionSizes{sizes}`, the size of the workspace directory (101, 188) | `generated/rpc.rs:5825`; `api_types.rs:26980-26982` (`HashMap<String,i64>`) | OK |
| 5 | Event-file-path RPC is `pub(crate)` (192) | `generated/rpc.rs:5798` | OK |
| 6 | `sessions().close(SessionsCloseRequest{session_id})` emits shutdown, flushes and releases the lock (100, 210) | `generated/rpc.rs:5900-5918` doc; `api_types.rs:26269-26275` | OK. It is also idempotent and "disposes the active session". |
| 7 | `prepare_resume_session` L1655, `resume_session` L1725 (100, 225) | `session.rs:1655`, `:1725` | OK. Not mentioned by the spec: `continue_pending_work` and `suppress_resume_event` (N1, N3). |
| 8 | `compaction_tokens_used{input_tokens, output_tokens, cache_read_tokens, duration}` at L2527-2562 (103, 175) | `generated/session_events.rs:2527-2562` | OK. It also has `cache_write_tokens`, `model` and `request_count`. |
| 9 | `AssistantUsageData` `cache_read_tokens` L4041, `input_tokens` L4090, `is_byok` L4102, `output_tokens` L4123, `cache_write_tokens` (104, D7) | `generated/session_events.rs` at those lines | OK |
| 10 | `QueueWithdrawMessageRequest{message_id, expected_prompt}` L22160-22164 (580, K2) | `generated/api_types.rs:22160-22164` | OK ("Message identity returned by send, not the queue item id") |
| 11 | `PostToolUseFailureOutput.additional_context` hooks.rs L166-169 (610) | `hooks.rs:166-169` | OK ("appended as hidden guidance … alongside the failed tool result") |
| 12 | `ExitPlanModeResult::default()` sets `approved: true` (508) | `handler.rs:172-180` | OK |
| 13 | With no handler, plan exit auto-approves (508) | `session.rs:3903-3908` (`json!({"approved": true})`) | OK |
| 14 | Six events typed `ephemeral: true` (233) | `nodejs/src/generated/session-events.ts`: `session.idle`, `assistant.usage`, `session.usage_info`, `assistant.intent`, `pending_messages.modified` and `user_input.requested` all have `ephemeral: true` ("Always true for events that are transient and not persisted") | OK. About 60 types are always ephemeral, including `tool.execution_progress` and `tool.shell_output`, which matters for N2. |
| 15 | `EventLogReadRequest.include_ephemeral` (default true, interleaved) and `wait_ms` (≤ 30,000; ephemerals delivered during the wait are not replayable) (235) | `generated/api_types.rs:8108-8132`, `:8175-8185` | OK. The docs also mention an in-memory ephemeral ring that is pruned; "keep reading with a non-zero waitMs" supports the spec's paraphrase. |
| 16 | `summarize_for_handoff` result is "produced by an LLM" (116, 209) | `generated/api_types.rs:9777-9779` | OK |
| 17 | `disconnect()` doc requires an idle session (127, 210) | `session.rs:1329-1345` | OK |
| 18 | `PermissionDecision` variants (110) | `generated/api_types.rs:48500-48511` | OK. Legacy `Approved*` variants are also present. |
| 19 | `large_output` default 50 KB, written to a file (627) | `types.rs:806-821` | OK |
| 20 | `summary_content` at session_events L2637 (C17) | `generated/session_events.rs:2637` | OK |
| 21 | `Client::delete_session` lib.rs L2889 (213) | `lib.rs:2889` | OK |
| 22 | `InfiniteSessionConfig` defaults 0.80 / 0.95 (95, 153) | `types.rs:1001-1012` | OK |

No [SV] claim was contradicted. Every new line reference landed on the cited item.

## 4. Can the new rotation design still burn tokens or lose work?

In normal operation rotation should now be rare. No trigger fires on a schedule, idle agents are not woken, and the handoff is bounded and reuses a summary that was already paid for. The remaining risks are in the edges around it:

- **Token burn:**
  - (a) N4: if the threshold falls back to 0.80 on large windows, compactions time out and RT1 fires repeatedly, each time paying a cold prefix, a handoff and file re-reads.
  - (b) N5: between an RT1 or RT2 trigger and the safe point, queued traffic keeps the failing session busy, and each turn retries a failing compaction.
  - (c) N6: planned recycles driven by the leak, with no rate limit, cost every active agent a cold prefix each cycle.
  - (d) N2: false stall detections recycle the process, then the agents' re-wakes and re-reads add up.
- **Lost or duplicated work:**
  - (a) N1: a forced recycle after 10 minutes, or a false stall, interrupts running turns. Nothing re-wakes those agents, and lease expiry hands their tasks to others without a WIP commit.
  - (b) N3: a resume can silently start an empty session, losing the agent's context without any handoff.
  - (c) N9: after RT1, the handoff may carry no conversation summary.

Rotation itself keeps code on disk in the worktree. Context is the only thing that is lost.

## 5. Verdict

The revision fixes the core of the first review: 30 of 33 findings are fixed and the other 3 are partly fixed. The section 10 tables match ui-wiring.json, and every [SV] citation I checked is accurate. The design is not foolproof yet. Six major issues remain, all in recovery and lifecycle edges rather than in the rotation triggers:
- forced recycle and interrupted turns (N1)
- stall false positives (N2)
- silent new-session resume (N3)
- the threshold formula needs `token_limit` before it is available (N4)
- rotation ordering and no safe-point timeout (N5)
- the RT4 memory case combined with lazy resume can loop (N6)

Fix N3 and N4 before implementation, because each is a one-line config decision. N1, N2, N5 and N6 need small protocol changes in sections 4.3 and 4.4.
