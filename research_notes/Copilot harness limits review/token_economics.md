# Token economics of long agent sessions (as of 2026-10-13)

Scope: prompt caching per provider, compaction cost, fresh-session handoff cost, polling cost. Figures are list prices in USD per million tokens (MTok) from the official pages fetched on 2026-10-13. Scenario token counts are my assumptions, not measurements.

## 1. Prompt caching rules

### Anthropic
- Explicit markers: `cache_control: {type: "ephemeral"}` on blocks; optional `"ttl": "1h"`. Up to 4 breakpoints per request; the system checks at most 20 positions per breakpoint looking backward. Cache order is `tools`, then `system`, then `messages`. Source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Minimum cacheable length (per the same page): 512 tokens for Sonnet 5.5, Opus 5.5, Haiku 5.5, Fable 5/5.1, Mythos 5/5.1; 1,024 for Sonnet 5, Opus 4.8, Sonnet 4.6; 2,048 for Opus 4.7; 4,096 for Opus 4.6/4.5, Haiku 4.5. Shorter prompts are silently not cached. Source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Multipliers: 5-minute write 1.25x base; 1-hour write 2x base; hit 0.1x base (0.05x on Sonnet 5.5 and Opus 5.5; 0.025x on Fable/Mythos 5.1). The 5-minute TTL refreshes free on each hit. Source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching); [Pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- Keying: cumulative hash of the prefix ending at the breakpoint; "Cache hits require 100% identical prompt segments". Isolation is per organization, and per workspace on the first-party API. The page does not mention a session id. Source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- Invalidation table (same page): changing tool definitions invalidates all three levels; web-search or citations toggles invalidate system and messages; `tool_choice` and image add/remove invalidate messages only; thinking parameters and `output_config.effort` invalidate messages.

### OpenAI
- Automatic on prompts of 1,024+ tokens. For GPT-5.6 and later, a breakpoint sits at the end of the latest eligible message; earlier models use implicit intervals (2,048 tokens on GPT-5.5). Matching requires the entire rendered prefix to match. Source: [Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- `prompt_cache_key`: on earlier models it "influence[s] routing; [it does] not pin requests to a machine or guarantee a cache hit". On GPT-5.6+ it is optional. Source: same guide.
- Retention: GPT-5.6+ uses `prompt_cache_options.ttl = "30m"` (default and only value). Earlier models use `prompt_cache_retention` `in_memory` (about 5-10 min, max 1 h) or `24h`. Source: same guide; [Azure prompt caching](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching)
- Discount: GPT-5.6+ cached reads are 0.1x the uncached rate. Writes on GPT-5.6+ cost 1.25x and are not an additive fee. Earlier models have no write charge. Source: [Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- Prices (standard, short context, per MTok): gpt-5.6-sol input $4.00, cached $0.40, output $20.00; gpt-6.1-sol input $2.00, cached $0.10, output $10.00. Source: [OpenAI pricing](https://developers.openai.com/api/docs/pricing). Note the page says gpt-5.6-sol promotional pricing runs at least through 2026-11-21.

### Azure OpenAI
- Same 1,024-token minimum and identical-prefix rule. On models up to GPT-5.5, hits come in 128-token increments. GPT-5.6+ charges for cache writes; `prompt_cache_key` is supported; more than about 15 requests per minute on one key may miss. In-memory retention is 5-10 minutes (max 1 h); extended retention up to 24 h on listed models. Caches are not shared across Azure subscriptions. Source: [Azure prompt caching](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching)
- Azure per-token cache discount: not found (the Azure pricing page was not fetched).

### Google Gemini
- Implicit caching is on by default for Gemini 2.5+; minimum 2,048 tokens (2.5 Pro/Flash) or 4,096 (Gemini 3.x Pro/Flash preview). Hits appear in `usage.total_cached_tokens`. The page gives no discount percentage. Source: [Gemini caching](https://ai.google.dev/gemini-api/docs/caching) (page updated 2026-10-09)
- Explicit caching: minimum size not stated on the fetched page; TTL and storage pricing not on the page. Secondary sources claim a 90% discount on 2.5+ and storage fees; these are unverified. Source (secondary, unverified): [Gemini API optimization](https://ai.google.dev/gemini-api/docs/optimization); Google pricing page not fetched.

### DeepSeek
- Context caching is on by default. Each request builds a disk cache; a hit requires a full match with a persisted prefix unit; usage fields are `prompt_cache_hit_tokens` and `prompt_cache_miss_tokens`. Unused caches are cleared, usually within hours to days. Source: [DeepSeek KV cache](https://api-docs.deepseek.com/guides/kv_cache)
- Hit/miss prices, granularity (I could not confirm the 64-token unit), and cross-conversation behavior: not found on the fetched page.

### Does a new conversation with a byte-identical prefix hit an old cache?
- Anthropic: Yes, when the prefix hash matches within the same org (and workspace on the first-party API) and the TTL has not lapsed. Nothing depends on a session id. Source: [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)
- OpenAI: Keyed on exact prefix content; `prompt_cache_key` only affects routing, so a hit is likely but not guaranteed. Source: [Prompt caching guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- Azure: Yes, within one subscription and deployment (the page says caches are not shared across subscriptions). Source: [Azure prompt caching](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/prompt-caching)
- Gemini: Implicit caching matches common prefixes and says it works with stateful and stateless conversations; cross-conversation hits are implied, not stated. Source: [Gemini caching](https://ai.google.dev/gemini-api/docs/caching)
- DeepSeek: not stated. Gap.

## 2. What breaks a prefix hit
- Any change before the breakpoint: timestamps at the top of the system prompt, tool-list reordering or edits, model switch, `reasoning.effort` or `text.verbosity` changes, `context_management` (compaction). Sources: [Anthropic prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching); [OpenAI guide](https://developers.openai.com/api/docs/guides/prompt-caching)
- Editing earlier messages (append-only is required). Source: [Manus, Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus), which says "even a single-token difference can invalidate the cache from that token onward" and warns that JSON key order is not stable in many libraries.
- Compaction rewrites history, so the post-compaction prefix is new and must be written once.
- Idle time over the TTL (5 min Anthropic default; 30 min minimum on OpenAI GPT-5.6+).

## 3. Compaction cost and context rot

Formula (in-place compaction):
- Summary call: `K_c = C * p_hit + S * p_out` (if the compaction call shares the cached prefix; `C * p_base` if the cache has lapsed)
- Next turn: the new suffix S is written: `S * p_write`
- Saving per later turn: `(C - P - S) * p_hit`

Published numbers:
- Anthropic: "often 1,000-2,000 tokens" for sub-agent condensed returns; after Claude Code compaction the agent keeps "the five most recently accessed files". Compaction "can result in the loss of subtle but critical context". No compaction cost or quality-loss figures. Source: [Effective context engineering](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- Chroma Context Rot: 18 LLMs, 194,480 calls; degradation grows with input length; "Even a single distractor reduces performance relative to the baseline"; the text gives no exact accuracy numbers (they are in the charts). Source: [Chroma, Context Rot](https://www.trychroma.com/research/context-rot)
- Manus: KV-cache hit rate is "the single most important metric"; cached input $0.30 vs uncached $3 per MTok (10x), average input:output ratio about 100:1. Source: [Manus](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
- Amp: moved from handoff to automatic compaction at 90% full; the newer post argues compaction is good enough that handoff is no longer needed. Sources: [Amp Handoff](https://ampcode.com/news/handoff); [Amp, Rebuilt](https://ampcode.com/news/neo)
- Summary size and quality-loss measurements for compaction: not found.

## 4. Fresh-session handoff: re-discovery cost
- Formula: `K_h = P * p_hit + H * p_write + R * p_write + G`, where P = system and tools prefix, H = handoff summary, R = re-read files, G = cost of generating the handoff (0 if a deterministic script).
- Measurements (practitioner, not peer-reviewed): one developer's analysis of 722 sessions / 34.56B tokens found re-reading context was about 55.9% of list-rate cost and cache writes about 31.9%. Source: [DEV, "97% of what my coding agent billed for was re-reading its own context"](https://dev.to/arsentev/97-of-what-my-coding-agent-billed-for-was-re-reading-its-own-context-4o2). A GitHub issue logged a $2.53 / 7-minute run for a docs change, with 2-4 same-file re-reads targeted down to 0-1. Source: [fullsend-ai/agents #1369](https://github.com/fullsend-ai/agents/issues/1369)
- A 2026 arXiv paper argues token-count reductions overstate savings because cache traffic is priced at 10-125% of nominal input, and that compression which drops needed context causes re-reads and extra turns. Source: [arXiv 2607.12161](https://arxiv.org/pdf/2607.12161)
- Controlled re-discovery measurement for handoff: not found.

## 5. Polling vs notification
- Polling cost per hour (warm cache): `(3600 / T) * C * p_hit` (valid while T is less than the TTL, since each hit refreshes it).
- Notification with no polling: one cold resume if the TTL lapsed, `C * p_base` (or a 1-hour write of `C * 2 * p_base` beforehand on Anthropic).
- Claude Code docs: a background subagent's completion "notification in a later turn"; the harness re-invokes the parent. Source: [Claude Code subagents](https://code.claude.com/docs/en/sub-agents)
- Anthropic/OpenAI guidance specifically on polling vs notification: not found. Claude Code GitHub issues report agents still polling with sleep loops despite notifications (e.g., [#95202](https://github.com/anthropics/claude-code/issues/95202), [#94178](https://github.com/anthropics/claude-code/issues/94178)). These are user reports, not official guidance.

## 6. Worked example (assumptions: P = 10k tokens; S = 4k summary output; H = 2k handoff; R = 30k re-read tokens; C = 0.9 x window; G = 0; compaction assumed to need no re-reads, which is optimistic)

Prices:
- Claude Sonnet 5.5: base $2.00; 5m write $2.50; 1h write $4.00; hit $0.10; output $10.00 per MTok. Source: [Anthropic pricing](https://platform.claude.com/docs/en/about-claude/pricing)
- GPT-5.6-sol: input $4.00; cached $0.40; write $5.00 (1.25x); output $20.00 per MTok. Source: [OpenAI pricing](https://developers.openai.com/api/docs/pricing)

128k window (C = 115k):
- Sonnet 5.5 compaction: summary read $0.0115 + output $0.040 + new suffix write $0.010 = **$0.062**. Saving per later turn: 101k x $0.10/M = $0.010. Handoff: $0.001 + $0.005 + $0.075 = **$0.081**.
- GPT-5.6-sol compaction: $0.046 + $0.080 + $0.020 = **$0.146**. Saving per later turn: $0.040. Handoff: $0.004 + $0.010 + $0.150 = **$0.164**.

200k window (C = 180k):
- Sonnet 5.5 compaction: $0.018 + $0.040 + $0.010 = **$0.068**. Saving per later turn: $0.017. Handoff: **$0.081** (independent of window size).
- GPT-5.6-sol compaction: $0.072 + $0.080 + $0.020 = **$0.172**. Saving per later turn: $0.066. Handoff: **$0.164**.

Polling (60 s interval, cache kept warm):
- Sonnet 5.5, 128k: $0.0115 per poll, **$0.69 per hour**. 200k: $0.018 per poll, **$1.08 per hour**.
- GPT-5.6-sol, 128k: $0.046 per poll, **$2.76 per hour**. 200k: $0.072 per poll, **$4.32 per hour**.
- Notification path with one cold resume: Sonnet 5.5 128k about $0.23; GPT-5.6-sol 128k about $0.46 (using base input prices).

Conclusion from these numbers: at these assumptions, compaction and handoff cost about the same in dollars (both well under $0.20 per event). Polling dominates over an hour, so use notification. Re-discovery volume R is the biggest unknown; the one-time event cost is small relative to a single extra long-context turn with a cold cache.

## Gaps
- Azure per-token cache prices.
- Gemini explicit cache TTL, minimum size, and storage price (Google pricing page not fetched).
- DeepSeek hit/miss prices.
- Quantitative compaction summary size and quality-loss numbers from any vendor.
- Controlled measurement of post-handoff re-discovery tokens.
- Official Anthropic/OpenAI guidance on polling vs notification for background agents.
