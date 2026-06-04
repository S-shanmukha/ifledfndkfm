# OpenRouter × Clyro — Complete Findings

**For:** Bhaskar **From:** Shanmukha **Date:** 2026-06-04
Everything we explored: the goal, what clyro is, where it's weak, OpenRouter's
full technical surface, the analysis, what we built (with code + tests), and the
open recommendations.

## Contents
1. The task & how we approached it
2. What clyro is (architecture)
3. Where clyro is weak (with code citations)
4. OpenRouter — full technical surface
5. Cost exactness analysis (direct key vs OpenRouter)
6. Feature-by-feature verdict (everything)
7. What we built — how, code, and test results
8. Conditional items — how we'd do them
9. Open recommendations
10. Bottom line

---

## 1. The task & how we approached it

Bhaskar's ask (verbatim intent): *"what is OpenRouter / do we have to add it to
clyro / if yes, how — explore it."*

Approach: instead of bolting OpenRouter features on for their own sake (which
only adds latency and bloat), we found where **clyro is genuinely weak today**
and checked whether OpenRouter fixes exactly that. Guiding constraints:
- **Never edit clyro's source** — only use its public API.
- **No added runtime latency.**
- **Don't send customer data to a third party** (clyro is a privacy/governance tool).

---

## 2. What clyro is (architecture)

Clyro is an **AI-agent governance & observability layer**. You wrap an agent with
`clyro.wrap(agent, config=ClyroConfig(...))` and it provides:

- **Tracing/observability** — session/step/tool-call/LLM-call events → local
  SQLite or cloud API.
- **Execution controls** — `max_steps`, `max_cost_usd`, loop detection.
- **Policy enforcement** — rules evaluated against action parameters.

**Modes:** `local` (YAML policies, no network) or `cloud` (backend API).

**Policy operators (8):** `max_value`, `min_value`, `equals`, `not_equals`,
`in_list`, `not_in_list`, `contains`, `not_contains`.
**Policy actions:** `block`, `allow`, `require_approval`.
**Operator semantics (clyroprod):** `in_list` = denylist (block if IN list),
`not_in_list` = allowlist (block if NOT in list).

**Backend components:** `api` (FastAPI), `sdk`, `mcp-wrapper`, `claude-code-hooks`.
The live/importing package is **`clyroprod/clyro`** (not the `archive/` copy).

**Key fact:** clyro sits *above* the LLM call — it observes via framework
adapters (LangGraph/CrewAI/Anthropic). It does **no** routing, provider
selection, fallback, or caching. Those belong to the agent or a gateway.

---

## 3. Where clyro is weak (with code citations)

We tested several suspected weaknesses; most didn't survive scrutiny. The honest
list:

**REAL & always-present:**
- **Made-up costs.** `DEFAULT_PRICING` has only ~8 models
  (`clyroprod/clyro/config.py:86-97`). Unknown models silently get a fabricated
  `$0.01/$0.03` per-1K fallback (`get_model_pricing`, `config.py:437`). Cost is
  always an *estimate* computed as `tokens × price / 1000`
  (`cost.py` `CostCalculator.calculate:426-466`, lookup at `:448`). Token
  fallback for tools/hooks is a rough `len(text)/4` heuristic. → `max_cost_usd`
  can't be trusted.
- **Blind to model metadata.** No knowledge of context windows, max output,
  capabilities — only hardcoded guesses (`model_selector.py`).

**NOT real clyro weaknesses (we dropped these):**
- *Provider failure recovery* — clyro records the failure (e.g. CrewAI
  `on_llm_failed`) but doesn't retry. **That's the agent framework's job, not a
  governance layer's.**
- *Privacy = name-only* — clyro can block a model name but can't verify a
  provider's data handling. Only matters if you run many providers; if you lock
  to one provider at the agent level, it's already handled.
- *Can't prove which provider ran it* — with a fixed provider, clyro already
  knows. Only unknown when routing through a gateway.
- *Crashes on long prompts* — **not confirmed**; clyro doesn't make the call, so
  the provider returns the error, not clyro. Retracted.

**Useful public hooks for cost (no source edits needed):**
- `ClyroConfig(pricing=...)` field (`config.py:275`)
- `config.register_model_pricing(model, in_per_1k, out_per_1k)` (`config.py:439`)
- `session.cumulative_cost` (`session.py:139`), `session.add_cost(delta)`
  (`session.py:472`), `clyro.get_session()` (`wrapper.py:1571`)

---

## 4. OpenRouter — full technical surface

### 4.1 What it is
A gateway in front of ~690 models from all major providers. One OpenAI-compatible
API key; it handles model access, provider routing, failover, caching, billing.

### 4.2 Default routing behavior
1. Drop providers with outages in the last 30s.
2. Weight the rest by price using **inverse-square** weighting (a $1/M provider
   is 9× more likely than a $3/M one).
3. Remaining providers form a fallback chain.
Disabled if you set `sort` or `order`.

### 4.3 Provider routing parameters (inside a `provider` object)
| Param | Meaning |
|---|---|
| `order` | Try these providers in this exact sequence |
| `allow_fallbacks` | Turn the fallback chain on/off (default true) |
| `only` | Allowlist of providers |
| `ignore` | Denylist of providers |
| `sort` | `price` / `throughput` / `latency` (or object `{by, partition}`) |
| `require_parameters` | Only route to providers supporting all request params |
| `data_collection` | `allow` (default) / `deny` (no logging/training) |
| `zdr` | Zero data retention enforcement |
| `enforce_distillable_text` | Only authors permitting distillation |
| `quantizations` | int4, int8, fp4, fp6, fp8, fp16, bf16, fp32, unknown |
| `preferred_min_throughput` | by percentile, e.g. `{p90: 50}` tok/s |
| `preferred_max_latency` | by percentile p50/p75/p90/p99 |
| `max_price` | ceiling: `{prompt, completion}` $/M |

Shortcuts: model slug `:nitro` (throughput), `:floor` (price). Anthropic beta
headers (`x-anthropic-beta`) enable fine-grained streaming, interleaved
thinking, strict tool use.

### 4.4 Model fallbacks
`models: [...]` — array of models tried in order; failover triggers on
context-length, moderation, rate-limit, or downtime. Billed at the model that
actually ran (returned in `model`).

### 4.5 Request/API parameters
Sampling: `temperature`, `top_p`, `top_k`, `frequency_penalty`,
`presence_penalty`, `repetition_penalty`, `min_p`, `top_a`, `seed`.
Generation: `max_tokens`, `max_completion_tokens`, `stop`.
Output: `response_format` (JSON mode / strict schema), `structured_outputs`,
`logprobs`, `top_logprobs`.
Tools: `tools`, `tool_choice`, `parallel_tool_calls`.
Reasoning: `include_reasoning`, `reasoning`, `reasoning_effort`
(xhigh/high/medium/low/minimal/none).
Other: `logit_bias`, `web_search_options`, `verbosity` (low→max).

### 4.6 Model catalog schema (`GET /api/v1/models`)
Each entry: `id`, `canonical_slug`, `hugging_face_id`, `name`, `created`,
`description`, `context_length`,
`architecture {modality, input_modalities, output_modalities, tokenizer,
instruct_type}`,
`pricing {prompt, completion, input_cache_read, input_cache_write, web_search,
image, audio, internal_reasoning}` — **per-token** decimal strings (USD),
`top_provider {context_length, max_completion_tokens, is_moderated}`,
`per_request_limits`, `supported_parameters[]`, `default_parameters{}`,
`supported_voices`, `knowledge_cutoff`, `expiration_date`, `links`.

### 4.7 Usage accounting / actual cost
Every response includes a `usage` block:
- `prompt_tokens`, `completion_tokens` (native tokenizer)
- `cost` — **authoritative USD actually charged**
- `cost_details {upstream_inference_cost, cache_discount}`
Always included now (`usage:{include:true}` is deprecated/automatic). Streaming:
in the last SSE message. Async lookup: `GET /api/v1/generation?id=...`.

### 4.8 Provider token detail (direct, non-OpenRouter)
OpenAI: `usage {prompt_tokens, completion_tokens, total_tokens,
prompt_tokens_details{cached_tokens, audio_tokens},
completion_tokens_details{reasoning_tokens, ...}}`.
Anthropic: `usage {input_tokens, output_tokens, cache_creation_input_tokens,
cache_read_input_tokens}`.
**Per call**, not aggregate — you sum calls for a task/session. Providers return
**tokens, never a dollar amount.**

### 4.9 Billing / privacy / BYOK (FAQ)
Credits in USD (Stripe/crypto), pass-through provider pricing + platform fee,
credits expire after 1yr inactivity, 24h refund window. Prompts/completions
**not logged by default**; opt-in logging for a 1% discount. BYOK: first 100k
requests/mo free, then a small %. Variants: `:free`, `:extended`, `:thinking`,
`:nitro`, `:floor`.

### 4.10 Plugins & multimodal
Plugins: web search, PDF parsing (file-parser), response-healing (auto-fix JSON),
context-compression (middle-out). Multimodal: image/audio/video input, image
generation, text-to-speech. Presets: server-side named config bundles
(model+params+routing).

---

## 5. Cost exactness analysis (direct key vs OpenRouter)

| | Direct provider key | Through OpenRouter |
|---|---|---|
| Token counts in response | ✅ exact (native) | ✅ exact |
| Dollar cost in response | ❌ none (tokens only) | ✅ `usage.cost` |
| Computable `tokens × price` | ✅ exact-to-published-rate | ✅ |

**Conclusions:**
- A direct provider key **never returns a dollar figure** — only tokens. So
  `exact tokens × accurate price` is the best obtainable cost, and it matches the
  bill at standard published rates.
- It diverges from the literal invoice **only** by things no real-time field
  exposes: enterprise/volume discounts, prompt-cache discounts, batch discounts,
  free credits. (Even OpenRouter's `usage.cost` reflects *OpenRouter's* pricing,
  not your negotiated direct rate.)
- The cache-discount portion *can* be closed by using detailed token fields
  (`cached_tokens`, `reasoning_tokens`) × detailed price fields
  (`input_cache_read`, `internal_reasoning`) — clyro currently uses only
  input/output, ignoring these.
- A provider-issued exact cost is only available if requests route **through** a
  biller (OpenRouter, incl. via BYOK).

---

## 6. Feature-by-feature verdict (everything)

**Legend:** 🟢 helps clyro · 🟡 only if multi-provider · 🔴 not clyro's job · ✅ done & tested

### A. Models & access
| Feature | Verdict | Why |
|---|---|---|
| 690+ model catalog | 🟢 ✅ | our source of price + model data |
| One API key for all providers | 🔴 | convenience, not clyro's role |
| OpenAI-compatible endpoint | 🔴 | convenience |
| Model variants (`:free`/`:nitro`/`:floor`/…) | 🟡 | routing presets; only if multi-provider |
| Live model metadata | 🟢 ✅ | context limits + capabilities |

### B. Routing & reliability
| Feature | Verdict | Why |
|---|---|---|
| Provider routing (price/speed/latency) | 🟡 | needs multi-provider |
| Provider allow / deny (`only`/`ignore`) | 🟡 | maps to clyro `in_list`/`not_in_list` |
| Model fallback (`models:[...]`) | 🟡 | really the agent's job |
| Auto failover / load-balancing | 🟡 | gateway behavior |
| `max_price` ceiling | 🟡 | per-call cost cap |
| Throughput / latency filters | 🔴 | gateway plumbing |

### C. Cost & billing
| Feature | Verdict | Why |
|---|---|---|
| Live pricing | 🟢 ✅ | fixes clyro's made-up costs |
| Actual billed cost in response | 🟢 | only if routing *through* OpenRouter |
| Unified billing / credits | 🔴 | OpenRouter's billing |
| BYOK (bring your own key) | 🔴 | OpenRouter routing feature |
| Usage / activity dashboard | 🔴 | clyro has its own tracing |

### D. Privacy & governance
| Feature | Verdict | Why |
|---|---|---|
| `data_collection: deny` | 🟡 | only meaningful when routing through OR |
| `zdr: true` (zero data retention) | 🟡 | same |
| Reports actual provider used | 🟡 | audit value; only if multi-provider |
| Enterprise SSO/SAML, org controls | 🔴 | overlaps clyro itself |

### E. Plugins
| Feature | Verdict | Why |
|---|---|---|
| Web search | 🔴 | agent capability; clyro would *govern* it |
| PDF parsing | 🔴 | agent capability |
| Response healing | 🔴 | gateway feature |
| Context compression (middle-out) | 🟡 | minor |

### F. Multimodal
| Feature | Verdict | Why |
|---|---|---|
| Image input (vision/OCR) | 🔴 | agent capability |
| Audio / video input | 🔴 | agent capability |
| Image generation, text-to-speech | 🔴 | agent capability |

### G. Request features
| Feature | Verdict | Why |
|---|---|---|
| Structured outputs (JSON schema) | 🔴 | passthrough (could be policy-enforced 🟡) |
| Tool / function calling | 🔴 | passthrough |
| Reasoning controls | 🔴 | passthrough |
| Streaming | 🔴 | passthrough |
| Presets | 🟡 ⚠️ | overlaps what clyro itself does — watch |
| App attribution headers | 🔴 | convenience |

---

## 7. What we built — how, code, and test results

### 7.1 Live pricing — `clyro_openrouter_pricing.py` ✅
- `build_pricing()` fetches `GET /api/v1/models`, converts per-token → per-1K
  (×1000), registers both full id (`openai/gpt-4o`) and bare name (`gpt-4o`),
  merges over clyro's defaults, caches to `~/.clyro/openrouter_pricing.json`
  (24h TTL). Degrades gracefully (stale cache → empty → clyro defaults).
- Fed in via clyro's public `ClyroConfig(pricing=build_pricing())`. **No source
  edits, no latency, no customer data sent.**
- Result: **691 priced entries** vs clyro's 8.

**Cost recorded per refund turn (1500 in / 200 out), stock vs live:**
| Model | stock clyro | + live | note |
|---|---|---|---|
| gpt-4o-mini | $0.000345 | $0.000345 | known → identical (no regression) |
| gpt-4o | $0.010500 | $0.005750 | clyro's hardcoded price was stale |
| deepseek/deepseek-chat | $0.021000 | $0.000460 | was ~45× too high |
| mistralai/mistral-large | $0.021000 | $0.004200 | was ~5× too high |

**Live end-to-end test** on `AI_agents/refund_agent.py` (LangGraph, gpt-4o-mini):
2 LLM calls, refund approved, clyro session cost = **$0.00006885**. OpenAI
returned `gpt-4o-mini-2024-07-18`; clyro's partial-match resolved it to the
correct `gpt-4o-mini` price.

**Integration:** 2 lines added to `refund_agent.py` (`import build_pricing` +
`pricing=build_pricing()`).

**Import gotcha (documented):** adding `/home/shanmukha` to `sys.path` breaks
`from clyro import ClyroConfig`, because the `/home/shanmukha/clyro` repo dir
shadows the live `clyroprod` package. Fix: the helper was **copied next to the
agent** in `AI_agents/` rather than imported via path.

### 7.2 Model metadata + guards — `clyro_openrouter_metadata.py` ✅
- `build_model_metadata()` pulls context_length, max_output_tokens,
  input/output modalities, supported_parameters, is_moderated. Cached to
  `~/.clyro/openrouter_metadata.json`. **686 entries.**
- Lookups: `model_limits()`, `model_supports()`, `models_supporting()`.
- **Guard 1 (declarative clyro policy):** `capability_allowlist_policy("tools")`
  → `{parameter:"model", operator:"not_in_list" (allowlist), action:"block",
  value:[510 tool-capable models]}` — **validated against clyro's real
  `PolicyRule`.**
- **Guard 2 (pre-invoke check):** `check_within_limits("gpt-4o-mini",
  prompt_tokens=200_000)` → blocks (exceeds 128k window); 5_000 → OK.
- Kept as a **separate module** from pricing (per request); clyro untouched.

---

## 8. Conditional items — how we'd do them (🟡)

| Feature | How |
|---|---|
| Provider allow/deny | clyro `in_list`/`not_in_list` policy on a `provider` param |
| Per-call price ceiling | clyro `max_value` policy on per-request cost |
| Privacy (`data_collection`/`zdr`) | emit OpenRouter routing flags; only when routing through OR |
| Auto failover | agent's job, or pass OR a `models:[...]` fallback list |
| Exact billed cost | read `usage.cost` from response → `clyro.get_session().add_cost(delta)`; cleanest with a small clyro "cost hook" |
| Context compression | rely on OR middle-out; minor |

---

## 9. Open recommendations

1. **Adopt live pricing** as the real win (helps every customer, no downside).
2. **Combine** the two modules into one fetch when convenient (currently
   separate by request).
3. **Tighten model-name matching** (normalize dashes/dots/date-suffixes) for the
   few ids that still fall back.
4. **File a clyro feature request:** a "provider-reported cost" hook so exact
   `usage.cost` can be used *if* we ever route through OpenRouter.
5. Use detailed token fields (cached/reasoning) × detailed prices to close the
   cache-discount gap if we want near-invoice accuracy.
6. **Watch Presets** — OpenRouter doing governance-style config bundling is a
   mild overlap with clyro.

---

## 10. Bottom line

- OpenRouter is **fully explored.** Most of it isn't clyro's job.
- The one real, always-present clyro weakness — **made-up cost data** — is fixed
  by **live pricing**, built and tested, through clyro's public API with no
  latency, no dependency, and no customer data sent.
- **Model metadata + guards** are ready for context-limit/capability enforcement.
- Everything 🟡 waits for a multi-provider need; everything 🔴 stays out.
- **clyro's source was never modified.**

### Artifacts
- `clyro_openrouter_pricing.py` — live pricing helper (+ copy in `AI_agents/`)
- `clyro_openrouter_metadata.py` — metadata + guards
- `AI_agents/refund_agent.py` — integrated (2 lines) & tested
- `AI_agents/test_refund_pricing.py` — stock-vs-live cost comparison
- `openrouter-clyro-complete.md` — this document
