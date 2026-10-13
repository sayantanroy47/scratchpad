# Copilot CLI / SDK long-session failures: issue evidence

Collected 2026-10-13 with `gh issue view --comments` (JSON with body and comments) and `gh search issues`. No posts, comments or reactions were made. Numbers are copied as written in the issues; "reporter's estimate" marks values the reporter did not measure.

## Bottom line for the harness spec

- None of the 17 assigned issues reports a failure at the thresholds the spec cites (32 MiB of events.jsonl, 3,000 events, a second compaction, 4 h). The thresholds in the spec do not appear in any issue body or comment I read.
- Reported failures are far above 32 MiB. Sizes at which failures were reported: ~120 MiB / ~37k lines with mutex timeouts (#2609); 381,627,178 bytes (#4506, memory watchdog loop); 559,508,763 bytes failed to load and 429,424,287 bytes loaded (#4325); ~532 MB (#4251, resume grind/OOM); 934,639,321 bytes and 282,912 events (#4639); 1,664,072,585 bytes failed extension startup, largest successful 97,568,545 bytes (#4717). #4805 reports 38.6 MB / ~13k events with 42 compaction cycles and resume stalls.
- Small sessions also failed: a 1.6 MB, 437-event session OOMed during compaction (#4780), which points at heap use in compaction, not file size.
- Duration: failures after 37 min (#4686), 3 to 4 h of CPU (#4699), 72 h of uptime (#4251 comment, 1.0.79-5), 77.5 h (#4251 comment, Windows 1.0.79), 24 h (#4686 comment, 1.0.93).
- Compaction: 131 compactions with 110 failed in one session (#5054); 825 compactions in a 24 h run (#4686 comment); 33 compactions in a session that froze (#5035); 38 failed out of 38 starts (#4663).

## Status summary

- Closed by automated bot comments that say "resolved as of v1.0.x" (not a maintainer verification): #4664 (v1.0.88), #4717 (v1.0.88), #4098 (v1.0.86), #3366 (v1.0.89), #2649 (v1.0.88), #4807 (v1.0.89), #5045 (v1.0.96).
- Closed by maintainers or reporters: #4325 (closed 2026-08-10, no version in the last comment I read), #662 (closed 2026-02-25, comment: "We've improved our handling of large files"), #1457 (closed 2026-04-07, comment: "I believe this has been addressed in recent releases").
- Conflict: #4664 was closed against v1.0.88, but #4725 (open) reports "Still happening on **1.0.88**" with the same heap failure.
- Open, reproduced after claimed fix: #5100 (still on 1.0.96-2 per comment), #4725 (1.0.88), #4686 (1.0.93 per comment on 2026-10-09).
- No open issue in the set asks for or reports a size cap or rotation with a maintainer response. #2261 and #4063 are open, with only bot or reporter text.

---

## A. Assigned issues

### github/copilot-sdk

**#2261: Ensure `events.jsonl` honors inherited file-size limits** (OPEN, created 2026-08-04)
- Numbers: SDK `github-copilot-sdk==1.0.7`, bundled CLI 1.0.71, host file-size limit 50 MB, Linux container, error `I/O error: File too large (os error 27)` (EFBIG). Size at failure is not reported beyond the 50 MB limit.
- Root cause: stated by reporter that the CLI inherits RLIMIT_FSIZE and events.jsonl is unbounded. Bot investigation (github-actions, label `bug`) agrees that compaction "does not truncate or rotate `events.jsonl` on disk".
- Workaround (bot comment): `ulimit -f unlimited`, or periodic `deleteSession` plus recreate. Maintainer statement: none.
- Link: https://github.com/github/copilot-sdk/issues/2261

**#2510: Rust: bound JSON-RPC frames and transport queues** (OPEN, created 2026-09-03)
- Not a session-size issue. No session numbers. Transport frame and queue bounds (Rust only). No comments.
- Link: https://github.com/github/copilot-sdk/issues/2510

**#2469: Resume replays a text-less assistant turn as `content: null`** (OPEN, created 2026-09-02)
- Not size-related. CLI 1.0.83-1, `GitHub.Copilot.SDK` 1.0.0, BYOK Azure. Reporter scanned "252 local session directories" and found "15 occurrences" of empty `assistant.message` followed by `assistant.turn_end`. Also "~31,800 empty `assistant.message` events followed by `tool.execution_start`".
- Root cause: runtime serializes empty content as null on replay; Azure rejects with HTTP 400. No comments.
- Link: https://github.com/github/copilot-sdk/issues/2469

**#2781: Rust: subagent lifecycle hooks are logged as unknown** (OPEN, created 2026-09-27)
- Not size-related. No numbers. Hook names `subagentStart`/`subagentStop` not dispatched. No comments.
- Link: https://github.com/github/copilot-sdk/issues/2781

**#2794: Rust: `errorOccurred` / `sessionEnd` hook `error` field type mismatch** (OPEN, created 2026-09-29)
- Not size-related. CLI 1.0.84-5 and 1.0.85 (observed); SDK rust v1.0.14 and v1.0.15. No numbers. No comments.
- Link: https://github.com/github/copilot-sdk/issues/2794

### github/copilot-cli

**#4063: Persistent append handle for `events.jsonl`** (OPEN, created 2026-07-08, no comments)
- Numbers: Windows 10/11 with Defender real-time protection. Events "tens–hundreds of times per second" during active turns (reporter's estimate). No file size reported.
- Root cause (reporter): open/append/close per event triggers Defender rescans of the growing file. Workaround: batch events or keep a persistent handle (proposed). No maintainer statement.
- Link: https://github.com/github/copilot-cli/issues/4063

**#5035: CLI Updates Stop (events.jsonl keeps growing)** (OPEN, created 2026-10-01, no comments)
- Affected 1.0.91, Windows x64 in Windows Terminal. Session "compacted 33 times". Waited "not more than 30m" with no recovery. No size reported.
- Root cause: not identified by reporter ("I am not sure what causes it"). Possible causes suggested: large history, expanded "Thinking" on Opus 5.5 Max.
- Link: https://github.com/github/copilot-cli/issues/5035

**#4251: Resume of a large session OOMs / ~70 min CPU grind in 1.0.74** (OPEN, created 2026-07-24, 4 comments)
- Numbers (reporter): session events.jsonl ~532 MB, 184 checkpoints, created ~4 months earlier, resumed daily. Peak RSS: 1.0.73 ~2.8 GB (steady), 1.0.74 ~8.5 GB (climbing). Reconstruction ~70 min of one CPU core. Node heap default ~2716 MB on 12 GB host. "~16× expansion: a 532 MB log parses to ~8.5 GB RSS". Remote export uploads and reads full log.
- Root cause (suspected by reporter): 1.0.74 subagent-timeline rework; `_timelineReplayBuffer` appears in 1.0.74 only.
- Comments:
  - MattPD (2026-08-01): session loading stopped at 559,508,763 bytes and loaded again at 429,424,287 bytes. V8 max string 536,870,888 bytes (512 MiB) cited; reporter did not verify decimal vs binary. Reproduces on 1.0.77 with no version change. Workaround in #4325: truncate oversized strings under `.data.result` with jq.
  - rjmurillo (2026-08-06, 1.0.79-5): crash at uptime 259,262 s (72 h), events.jsonl 544 MB at death, JS heap 3,751 MB used / 3,845 MB total against 4,099 MB limit, RSS 15.0 GB, host 125 GB RAM. 2,090 libuv handles (2,059 async). Notes `NODE_OPTIONS` ignored by the SEA.
  - AndreiTkachyov (2026-08-11, Windows 1.0.79): fatal "Committing semi space failed" after ~77.5 h; JS heap 117,006,584 bytes used of 4,298,113,024 limit; host 32 GB, ~6.56 GB free.
  - simukka (2026-09-15): stopgap fork script in #4780.
- Maintainer statement: none.
- Link: https://github.com/github/copilot-cli/issues/4251

**#4699: OOM crash on long `--resume` sessions; crash dumps written into cwd** (OPEN, created 2026-09-02, 6 comments)
- Numbers (1.0.82, Linux x64, Node v24.18.1): 3 crashes on 2 Sep 2026: 00:51:37 heap 3.98/4.00 GiB, RSS 4.63 GiB, CPU 7833 s; 03:08:30 heap 3.99/4.00 GiB, RSS 4.75 GiB, CPU 4065 s; 14:39:00 heap 3.99/4.00 GiB, RSS 4.69 GiB, CPU 2785 s. "CPU 46m–2h11m" in resumed sessions. Host 30 GiB RAM, ~9 GiB free.
- Root cause: reporter says `--optimize-for-size` sets the 4.00 GiB cap and "retention growth looks like the real bug". Confirmed by MattPD (comment, 1.0.83) with a heap of 3.20 to 3.22 GiB of 4.00 GiB, RSS 10.6 to 15.2 GiB, user CPU 3.4 to 10.6 h before abort, and events.jsonl of 2.8 GiB (session A) and 1.7 GiB (session B). Session A died 5 times in a row, B 3 times.
- Other comments: kondv (2026-09-19, Windows 1.0.79-9, 1.0.81-0, 1.0.84-5, 4,099 MiB limit, four crashes). kondv (2026-09-21): TUI reproduction aborts at 21 turns with 515 MiB cap; headless ACP completes 50 turns with no heap growth. Raising `--max-old-space-size` gives roughly 99 percent heap at abort.
- Workaround: simukka's fork script (#4780); headless driver per kondv (not a fix).
- Link: https://github.com/github/copilot-cli/issues/4699

**#4805: Sessions become unrevivable: stale `inuse.<pid>.lock`** (OPEN, created 2026-09-10, 2 comments)
- Numbers (1.0.83, macOS): 14 of 184 sessions had a fault signature, 8 with a dead-PID lock. Worst local session: events.jsonl 38.6 MB, ~13k events, 42 compaction cycles (173 re-injected `system.message` events ≈ 10.7 MB). MCP OAuth timeout stalls resume ~5 min.
- Comment daveroama (2026-09-29): 7 live session hosts, 43 resumes in ~3 h, logs ~274 lines/s, MCP host processes at 35% and 20% CPU, 113+ ENOENT spawn failures.
- Root cause: stale lock not reclaimed (confirmed by deleting lock). Defect 4 (unbounded events.jsonl) suspected as same root as #4251.
- Link: https://github.com/github/copilot-cli/issues/4805

**#5045: /compact repeatedly fails with empty model response (gpt-6.1-sol)** (CLOSED COMPLETED, created 2026-10-02, closed 2026-10-13)
- CLI 1.0.84-3, Linux. No size numbers.
- Closure: bot comment says resolved in v1.0.96 (automated). CLI 1.0.84-5 release note: "/compact no longer reports an empty model response when a valid summary was returned".
- Link: https://github.com/github/copilot-cli/issues/5045

**#4506: Memory-pressure watchdog force-compacts at 23% context usage** (OPEN, created 2026-08-16, 1 comment)
- Numbers (1.0.80, Linux, 15 GB RAM, 48 GB swap): context 93,493 of 400,000 tokens (23%); compaction recovered 3 tokens (0.003%). events.jsonl 381,627,178 bytes. RSS 1.3 GB, swap 10.5 GB, committed ~11.8 GB, V8 heap limit 2144 MB, conversation ~38 KB, uptime 1d 13h. Logs ~5.8 GB in `~/.copilot/logs`; log sizes 2.4 GB, 1.8 GB, 1.1 GB for sessions A, B, C. Log counts in one 200 MB slice: 54 "Memory pressure detected", 42 "compaction already in progress". 128 KB floor is the only guard.
- Root cause (reporter): memory watchdog triggers compaction that does not reduce memory; events.jsonl held parsed in memory; `logLevel: "all"` forwards html5ever tracing.
- Link: https://github.com/github/copilot-cli/issues/4506

**#4639: event-storage exhaustion retry storm drives session into GC/compaction loop and OOM** (OPEN, created 2026-08-27, 2 comments)
- Numbers (1.0.80, Linux, 125 GiB RAM, no swap, Node v24.18.1): events.jsonl 934,639,321 bytes (~892 MiB), 282,912 event records, 32 checkpoints. Session created 2026-08-10; process started 2026-08-18; failure 2026-08-27 (~8.8 days of process, ~16.9 days of session). Counts: `session_event_storage_exhausted` 18,909; "Memory pressure detected" 21,352; "emergency compaction" 473; "timed out waiting for bridge event ack" 3,001 (timeout_ms 30000). Batch flush 500 events, HTTP 409.
- Root cause (reporter's inference): remote event storage exhaustion with retry backlog and memory pressure. Reporter did not capture `/proc` state before crash.
- Comments: vicondoa (2026-08-27): slow over time, segfault OOM with free host memory.
- Link: https://github.com/github/copilot-cli/issues/4639

**#4686: Node.js OOM after ~37 min: 31,965 leaked async libuv handles** (OPEN, created 2026-09-01, 4 comments)
- Numbers (1.0.82, Linux, Node v24.20.0): crash at 2,203 s (~37 min) uptime. old_space 3,899 / 3,899 MB; memoryLimit 4,298 MB; RSS 4,634 MB; libuv handles 31,996 (31,965 async unreferenced). Growth ~100 MB/min. Second crash (Marcus-Lindbloom 2026-09-01): 29,966 handles, userCpuSeconds 674 (~11 min of work).
- Root cause: reporter suspects per-turn or per-tool async handle leak (not confirmed).
- Comments: jeberger (2026-09-02): idle-session OOM with 4,089.9 MB heap, no libuv entries. edouardsence (2026-09-10): different profile (6 active async handles), heap raised to 6000 MB and still OOMed at 5.90/6.00 GiB after 276 s (see #4725). AntonPo456 (2026-10-09, 1.0.93, Windows): 24 h run, 825 compactions, heap 3.48 GB used of 4,298,113,024; 52 libuv handles (37 async); cost 99k AIC; session lost.
- Workaround (reporter): run node directly with `--max-old-space-size=12288` (buys ~2 h by reporter's estimate).
- Link: https://github.com/github/copilot-cli/issues/4686

**#4780: Session compaction OOMs and never completes** (OPEN, created 2026-09-09, 2 comments)
- Numbers (1.0.83, Linux, 67 GB RAM): four OOM reports, heap 4.06/4.30 GB, 4.03/4.30 GB, 4.15/4.30 GB, 4.01/4.30 GB; RSS 4.94 to 6.46 GB; CPU 466 to 964 s. Session: 437 events, 1.6 MB events.jsonl, ~161k tokens against a 200k limit. Four `session.compaction_start` events, none completed. Skipped counts 592 in event stream lag.
- Root cause (reporter): compaction materializes the whole conversation; failed compaction does not halt turn. Not confirmed by maintainer.
- Comments: edouardsence (2026-09-10): retry at 6000 MB heap OOMed at 5.90/6.00 GiB after 276 s; GC reclaimed 5896.1 to 5891.7 MB. simukka (2026-09-15): fork/trim gist script.
- Link: https://github.com/github/copilot-cli/issues/4780

**#4506** covered above.

**#5054: Automatic compaction keeps timing out ("summarizer did not settle within 300s")** (OPEN, created 2026-10-05, no comments)
- Numbers (Windows, CLI 1.0.89, model gpt-6-astra, token limit 1,050,000, reasoning xhigh): 131 compactions, 110 failed: 100 "did not settle within 300s", 5 "RetriesExhausted { retries: 5 }", 3 "Compaction Cancelled", 2 background-summarizer timeouts. Trigger about 840K tokens. Last successful compaction took 209 s. 15 failed in a row from 05:34 to 09:15 (context 842K to 924K).
- Root cause (reporter): summarizer at xhigh effort over 840K to 920K input tokens near the 300 s cap.
- Link: https://github.com/github/copilot-cli/issues/5054

**#4063 and #5035 and #4251** covered above.

---

## B. Other issues found by search (any date, 2025–2026)

Items marked "title only" were listed by search and their body was not read. Their states are from search.

### Events.jsonl size or growth
- #5035 (open): covered above.
- #4063 (open): covered above.
- #2609 (open, created 2026-04-09; CLI 1.0.21, Linux): events.jsonl ~120 MiB / ~37k lines (reporter later measured ~123 MiB); "timeout while waiting for mutex" on append; stale `inuse.*.lock` files. Read in full.
- #4805 (open): covered above.
- #4506 (open): covered above.
- #4639 (open): covered above.
- #3551 (open, title only): feature request to formalize events.jsonl as an API. No size numbers.
- #4141 (open, title only): Xcode DerivedData written into `~/.copilot/session-state` accumulates indefinitely.
- #4098 (closed COMPLETED, 2026-09-20; bot: resolved in v1.0.86): truncated and concatenated events in events.jsonl on resume, CLI 1.0.71-0, three malformed physical lines.
- #2649 (closed COMPLETED, 2026-09-27; bot: resolved in v1.0.88): resume fails with "Session file is corrupted" from raw newlines in `tool.execution_complete`; CLI 1.0.24; lines 1086–1088 and 1577–1579.
- #3366 (closed COMPLETED, 2026-09-30; bot: resolved in v1.0.89): orphan `tool_use` in events.jsonl wedges sessions.

### Slow or failing session resume
- #4251 (open): covered above.
- #4805 (open): covered above.
- #4664 (closed COMPLETED, 2026-09-26; bot: resolved in v1.0.88): "Copilot CLI crashes with JavaScript heap out of memory when resuming a long-standing session", CLI 1.0.81; heap ~4.06 GB of 4.10 GB. Conflicts with #4725 comment (still on 1.0.88).
- #4717 (closed COMPLETED, 2026-09-23; bot: resolved in v1.0.88): "Extension startup fails on large session histories"; `Invalid string length` from `session.resume`; failing events.jsonl 1,664,072,585 bytes; largest successful 97,568,545 bytes; CLI 1.0.82.
- #4325 (closed COMPLETED, 2026-08-10; no version in the last comment read): "Session becomes permanently unloadable once events.jsonl exceeds V8's max string length"; failing 559,508,763 bytes; loaded at 429,424,287 bytes.
- #4138 (open, body read): resume triggers background compaction; empty compaction response leaves the process hung indefinitely; "recurred 4x"; CLI 1.0.71-2.
- #5023 (open, title only): resume fails when masked code-change metrics make `session.shutdown` counters strings.
- #4259 (open, title only): `--resume` replays orphaned `permission.requested` events.
- #2800 (SDK, open, title only): `Client::resume_session` errors do not say which stage failed.

### Out-of-memory
- #4725 (open, created 2026-09-04, title "Frequent JavaScript heap out of memory"): crash "every few minutes"; heap 3918.2 MB used of 4097.4 MB (Mark-Compact line); last comment: "Still happening on 1.0.88", macOS.
- #4699, #4686, #4780, #4251, #4639, #4506: covered above.
- #4664 (closed): covered above.
- #4499 (open, created 2026-08-14, v1.0.79 Windows, Node v24.18.1): "Committing semi space failed"; heap ~607 MB of 4.30 GB limit; host 32 GB RAM nearly exhausted; autopilot.
- #1457 (closed COMPLETED, 2026-04-07; CLI 0.0.409): heap OOM at ~4 GB during typecheck workflow. Comment: "I believe this has been addressed in recent releases."
- #2132 (open, created 2026-03-18, Windows, CLI 1.0.x): "crashes during parallel background agent execution (OOM/TypeScript error in long sessions)". No numbers beyond 30–60 min sessions, 4–7 background agents.
- #3976 (open, title only): native tgrep indexer OOM-kills host on large monorepos.
- #3687 (open, title only): copilot.exe fatal-aborts under load (0xc0000409).
- #4891 (open, title only): copilot.exe hard-crashes on Windows (0xc0000005).

### Invalid string length
- #662 (closed COMPLETED, 2026-02-25; CLI 0.0.365): `RangeError: Invalid string length` scanning a ~500+ MB JSON file. Comment: "We've improved our handling of large files".
- #4717 and #4325: covered above.

### Memory leak / runaway in long sessions
- #4612 (open, created 2026-08-26): runaway FileWatch loop; debug log 13 GB; growth ~460 KB/s; RSS ~1.1 GB; CPU ~200%. Last comment: reproduced on 1.0.84-0.
- #4807 (closed COMPLETED, 2026-09-29; bot: resolved in v1.0.89): idle CLI FileWatch event storm; log 33 GB then over 37 GB; ~216 KB/s; CPU ~221% for 35 h; RSS ~1.4 GiB; CLI 1.0.82.
- #4306 (open, title only): subtasks freeze and stop responding.
- #841 (closed COMPLETED, 2026-02-25): title says "CRITICAL: Memory leak - @ symbol triggers unbounded heap allocation causing OOM crash" but the body describes unrelated `kill`/`pkill` command interception. Title and body do not match; low evidentiary value.

### Compaction failures
- #4663 (open, created 2026-08-30): failed compaction retried every turn; CLI 1.0.81-3; session A 38 starts, 0 success, 38 failed; session B 16 starts, 0 success, 16 failed. Comment (reporter): not a maintainer.
- #5054 (open): covered above.
- #5045 (closed): covered above.
- #4138 (open): covered above.
- #4646 (open, title only): "Compaction fails with CAPIError 400 Tool choice must be auto on custom models".
- #4914 (open, title only): Gemini continuation and /compact fail with "Tool name is required".
- #3542 (open, title only): enterprise MCP allowlist tool schemas exceed token limit, "persistent compaction loop".
- #2553 (open, title only): CAPIError 400 reasoning_effort when compacting after model switch.
- #2861 (closed, title only): compaction failed: empty response (3x retry).
- #4698 (closed, title only): cannot compact session.
- #4351 (open, title only): session cost total loses spend the first time compaction succeeds in a process lifetime.
- #4733 (open, title only): when max_output_tokens is hit, some events are not emitted or logged.

### Infinite sessions
- No CLI issue matched "infinite sessions" in search.
- SDK #2355 (open, title only, search in github/copilot-sdk): "Custom 'anthropic' provider does not enforce provider.max_prompt_tokens — sessions grow past the model context window until a hard…"

### Event delivery stopping
- #5100 (open, created 2026-10-09, 1 comment, read in full): session event delivery permanently fails after one 120 s host-ack timeout. CLI 1.0.94 (observed), also 1.0.90–1.0.92, still on 1.0.96-2 per comment. Counts: one process logged the failure 1,645 times; ~14,600 warnings and ~40 MB of logs in 10 minutes; 20 `session.error`s across two sessions; 62 and 39 "Node memory snapshot timed out" lines in two processes. Two sessions ~100 MB events.jsonl, ~4,000 tool calls, 7–8 resumes. Comment (ElliotChong-MS, 2026-10-12): new warning "Native session event flush exceeded 10000ms" before stalls; stalls of 41.7 s and 187.2 s with no JS logs. Root cause (reporter): Node main thread unresponsive. Maintainer statement: none.
- #5018 (open, title and first lines read): "Permission prompts auto-cancelled with 'Session host did not acknowledge 'permission.requested' delivery within 5 seconds'". Same family as #5100.
- #4755 (open, title only): session wedges when a queued message lands at turn end.
- #4990 (open, title only): busy Copilot CLI swallowed messages.
- #4907 (open, title only): periodic MCP reconnect notifications flood history.
- #4946 (open, title only): HTTP 400 `content[].thinking` after a background shell completion.
- SDK #2008 (open, title only): .NET runtime logs forwarding `session.idle` but SDK callback never delivers it.

### Host ack timeout
- #5100 and #5018: covered above.
- #4598 (open, title only): MCP startup connects only 3 of 18 servers and reloads destroy live handles.

---

## C. CLI release notes, 1.0.80 to newest

Source: GitHub releases for github/copilot-cli (https://github.com/github/copilot-cli/releases/tag/v<version>). The repo has no CHANGELOG.md (API returns 404). Newest release is 1.0.97-2 (2026-10-13). Dates below are release dates from the releases list.

Matching entries (text quoted from release bodies):
- 1.0.81 (2026-08-27) and 1.0.81-14: "Resume large sessions faster by showing recent history first while older messages load."
- 1.0.83 (2026-09-04) and 1.0.83-4: "Long-running sessions on Linux return freed memory to the system instead of holding gigabytes of it" (memory).
- 1.0.83 and 1.0.83-4: "Resuming large sessions keeps the input prompt responsive sooner."
- 1.0.84-2 (2026-09-08): "Speed up startup when resuming an existing local session by its exact UUID with `--resume`."
- 1.0.84-2 and 1.0.85: "Large sessions resume without freezing the interface during context token counting."
- 1.0.84-5 (2026-09-11) and 1.0.85: "/compact no longer reports an empty model response when a valid summary was returned."
- 1.0.84-6 (2026-09-14) and 1.0.85: "Automatic compaction now saves a checkpoint that appears in /session checkpoints."
- 1.0.84-6 and 1.0.85: "When session history is truncated or compacted just as a turn finishes, the 'Working' indicator now clears instead of staying up for the rest of the session."
- 1.0.84-9 (2026-09-15) and 1.0.85: "Reduce metadata scanning time for large local session histories, with increased thread and memory use."
- 1.0.86-0 (2026-09-16) and 1.0.86 (2026-09-17): "Resume sessions even when transcript files contain recoverable corruption."
- 1.0.87-0 (2026-09-18) and 1.0.87 (2026-09-21): "Resume very large local sessions and continue with new prompts reliably."
- 1.0.87-0 and 1.0.87: "Session resume no longer hangs while reconnecting MCP servers."
- 1.0.87-0 and 1.0.87: "Tear down the processes an internal git command started when it times out, so a slow repository status check no longer leaves them running and consuming memory."
- 1.0.88-0 (2026-09-22) and 1.0.88: "Resuming large local sessions keeps transcript memory bounded for smoother CLI performance."
- 1.0.88-0 and 1.0.88: "Session resume preserves pending conversation events when saving fails and explains that retrying is safe."
- 1.0.90 (2026-09-30) and 1.0.90-6: "Compaction now returns a summary even when instructions ask for tool use."
- 1.0.92-4 (2026-10-04) and 1.0.92 (2026-10-05): "Compaction keeps your latest prompt when requests exceed context limits."
- 1.0.92-4 and 1.0.92: "Sessions no longer slow to a crawl for minutes after the agent writes a very large file in one step."
- 1.0.92-4 and 1.0.92: "Legacy HTTP+SSE MCP connections no longer hang indefinitely when a message POST is never acknowledged; the acknowledgement is bounded by the server's configured timeout." (MCP only, not host event delivery.)
- 1.0.97-0 (2026-10-12): "Emergency memory relief in long sessions can now bypass its cooldown before V8 aborts, so a session that would have been killed by a JavaScript heap out of memory crash may survive with compacted history instead."
- 1.0.97-0: "Resuming suspended sessions now reports interrupted tool calls and avoids replaying canceled writes." (resume, not speed)

Not found in any release note from 1.0.80 to 1.0.97-2: events.jsonl size, EFBIG, a session size cap or rotation, a session-count or event-count threshold, host event-ack timeout fixes, or event delivery fixes. 1.0.80 body is only "Update model configurations".

## D. SDK changelog (github/copilot-sdk CHANGELOG.md)

- v1.0.13 (2026-09-04), section "Feature: early session-event subscription (Rust)": `prepare_session` / `prepare_resume_session` let a subscriber see events emitted during `session.create` / `session.resume`. Previously such events "were broadcast with no receiver installed and dropped". Default `event_buffer_capacity` 512; "slow subscribers observe `Lagged` rather than applying backpressure". Relevant to event delivery loss.
- No SDK changelog entry mentions events.jsonl size, EFBIG, a size cap, or the host ack timeout.

## Gaps

- The spec's thresholds (32 MiB, 3,000 events, second compaction, 4 h) have no source in these issues. Any threshold claim should cite a different source.
- #2261 has no reporter-measured file size beyond the 50 MB host limit.
- Closure version claims are bot-generated ("resolved as of vX") and not maintainer-verified; #4664 conflicts with #4725.
- Title-only items above were not opened.
- Two Windows-related reports (#4251 comment, 1.0.79-5 and Windows 1.0.79 crash) were read from comments, not from the reporter's logs.
