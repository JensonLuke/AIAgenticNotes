# Agentic Text-to-SQL Systems — Study Guide

Oct 3, 2026 · @Jenson

Based on the "Week 2 — Agentic Text-to-SQL: Reliable Data Reasoning Systems" slide deck, with added explanation and a fully worked example for the running question: *"What were our top 5 products by revenue in EMEA last quarter?"*

## The core thesis

**Text-to-SQL** means translating a natural-language question into a SQL query a database can run. A single LLM call that writes one query and runs it is the *naive* version. The **agentic** version runs the model in a loop: it inspects the schema, tries a query, reads back the result or the error, and tries again — all under rules enforced by ordinary code, not by the model's goodwill.

The failure mode that matters most is not a crash — it's a **confidently wrong** answer: a clean table, a plausible number, no exception, that quietly drives a bad decision. A loud failure (syntax error, timeout, "I don't know") costs a retry. A silent one costs trust.

> **The thesis:** Correctness in text-to-SQL is a **systems property** — retrieval, grounding, validation, execution safety, and verification working together — not a cleverer prompt. The goal isn't an answer every time; it's a trustworthy answer, or an honest abstention.

Every subsystem described below exists to convert a possible "confidently wrong" into either a correct, auditable answer or an explicit "I can't answer that yet, here's what I need."

## The six-phase framework

The deck grows one architecture diagram through six phases, each adding exactly one capability. Everything downstream in this guide is organized the same way.

1. **Clarify** — turn a vague ask ("build text-to-SQL") into a testable spec: who asks, how messy is the schema, what's the correctness bar, read-only or not.
2. **Architecture** — start from a naive baseline (dump the whole schema, one LLM call, run it), break it on purpose, and draw the pipeline that fixes each break.
3. **Tool Use & Safe Execution** — give the agent typed tools instead of letting it guess or fire raw SQL; enforce safety in the execution layer, not the prompt.
4. **Schema Retrieval** — RAG, but for schema: find the ~20 relevant tables out of thousands.
5. **Schema Reasoning & SQL Generation** — disambiguate intent, ground meaning in a semantic layer, plan the join path, decompose into CTEs, self-correct, verify. **(The deepest dive — this is where interviews are won or lost.)**
6. **Reliability & Scale** — size the load, cache by tier, centralize model calls through a gateway, degrade gracefully, isolate tenants.

Three phases go deepest: **Tool Use & Safe Execution**, **Schema Retrieval**, and — above all — **Schema Reasoning & SQL Generation**.

## Phase 1 — Clarify the problem

"Build a text-to-SQL system" hides at least four decisions that change the entire architecture: **who asks** (analysts who read SQL vs. execs who don't), **how many tables and how messy**, **what's the correctness bar**, and **read-only or can it write**. Asking these before drawing anything is itself a signal of seniority.

### Pin the correctness tier first

"Correct" isn't one thing — the tier you target sets your whole verification budget, not the model you pick.

| Tier | Example question | Bar | Verification |
| --- | --- | --- | --- |
| Exploratory | "Roughly how are sales trending?" | Mostly-right, fast, easy to refine | Light |
| Operational | Daily dashboard the warehouse team acts on | Correct + stable across reruns | Verify + regression tests |
| Reporting / finance | Quarterly revenue the CFO signs off | Provably correct or explicit abstention | Heavy verify, abstain if unsure |

### Write the contract: in, out, done

- **Input:** NL question + history, user identity (tenant + row-level permissions), warehouse dialect, access to schema/semantic layer/past queries.
- **Output:** a direct answer, the SQL that produced it (auditable), a confidence signal + stated assumptions, or an explicit abstention.
- **Definition of done:** SQL runs within cost/time budget, result passes verification, answer is traceable to real columns and definitions, low confidence → abstain, never bluff.

### Scope it, and name what's out

In scope: read-only analytical questions over a large, messy warehouse (schema retrieval is core), single-turn + light follow-ups, correctness/safety/verification/scale. Out of scope (stated, not solved): writes/DDL/ETL, open-ended multi-turn conversation, building a BI UI.

**Worked assumptions for this guide:** ~5,000 tables, tens of thousands of columns, thousands of questions/day; mixed analyst/exec audience (show SQL, lead with the answer); operational-reporting tier; read-only per-tenant role with a partial semantic layer; a few seconds of latency is fine, correctness beats speed.

## Phase 2 — Architecture: earn the pipeline

Always start with the simplest thing that could work, then break it on purpose — each break motivates exactly one component.

**The naive baseline:** serialize the entire schema into the prompt, make one LLM call for the SQL, run it directly against the warehouse, return whatever comes back. No retrieval, no validation, no verification.

### Four ways the baseline breaks

| Failure | What happens | Fix |
| --- | --- | --- |
| Won't fit | 5,000 tables blow the context window; even if they fit, the model drowns in irrelevant schema | → Schema Retrieval (Phase 4) |
| Wrong meaning | Guesses "revenue," picks the wrong join, double-counts — confidently | → Schema Reasoning + semantic layer (Phase 5) |
| Unsafe to run | A generated cross-join scans terabytes, or touches data this user can't see | → Tool layer + execution guards (Phase 3) |
| No idea if right | The result looks fine but nothing checked it, and nothing knows when to abstain | → Verification + confidence (Phase 5) |

The four focus areas of the whole deck fall directly out of these four baseline failures.

### A pipeline that can loop — which is what makes it "agentic"

A pipeline is a fixed sequence: step one, then two, then three, every time. An agent chooses what to do next based on what it has observed. This system is a pipeline in shape, but the controller can loop back — e.g., from verification back to reasoning when a check fails — which is what makes it agentic.

Walking the running example through the six stages: **(1) Intent** — parse "top 5 products by revenue, EMEA, last quarter" into metric/filters/grain/limit, clarifying if ambiguous. **(2) Retrieve** — pull ~20 relevant tables + metric definitions out of 5,000. **(3) Reason + generate** — resolve "revenue," plan the join path, emit SQL. **(4) Validate** — parse, lint, then EXPLAIN to check plan and cost before running anything. **(5) Execute** — run under a read-only role with row/byte/time limits on a replica. **(6) Verify** — sanity-check the result, compare to a known metric, attach confidence or abstain. Notice how much correctness work happens *before* a single row is read.

### Storage & serving — name the actual stores

| Component | Role | Example tech |
| --- | --- | --- |
| Vector index | Schema-element & few-shot embeddings | pgvector / Pinecone / OpenSearch |
| Semantic layer | Metric & entity definitions, join keys, grain | dbt metrics / LookML / metadata DB |
| Metadata store | Table/column catalog, types, FK graph, sample rows | Information schema + catalog service |
| Warehouse (read replica) | Executes SQL under scoped roles | Snowflake / BigQuery / Redshift |
| Cache tiers | Schema embeds, generated SQL, result sets | App-side Redis + warehouse-native cache |
| Agent + gateway | Stateless controller; all model calls via one gateway | App servers + LLM gateway |

Vague boxes lose points in an interview; concrete choices with reasons win them.

*[Diagram omitted from this export: complete governed pipeline · 5 stages + feedback loop]*

The top row is the forward path; the dashed line is the one loop to memorize — a failed validation, a failed execution, or low post-verification confidence routes back to Ground+Generate, bounded to ≤3 repair attempts before the system abstains.

## Phase 3 — Tool Use & Safe Execution

The model should never hallucinate schema or fire raw SQL at the warehouse. It calls **typed tools** to inspect, validate, and execute — think of it as a junior analyst (the model) whose queries a senior colleague (the tool layer) decides whether to run.

### The core toolset — inspect, then run safely

| Inspect schema (read-only) | Generate → run safely |
| --- | --- |
| `list_tables(filter?)` — candidate tables by keyword | `validate_sql(sql)` — parse + lint, no execution |
| `get_schema(tables)` — columns, types, keys | `explain_sql(sql)` — plan + estimated cost/rows |
| `sample_rows(table, n)` — real example values | `execute_sql(sql)` — guarded, read-only run |
| `get_fks(tables)` — foreign-key edges for joins | `check_result(sql, rows)` — sanity flags for verify |

Every tool has typed inputs/outputs, and **failures come back as structured data**, never an unhandled exception — a raised exception ends the turn, but a typed error (`type`, `message`, `hint`) lets the agent diagnose and retry. Self-correction depends on this.

**Why sample rows matter:** schema alone leaves enums ambiguous. `WHERE status = 'active'` against a column that actually stores `'A'/'C'/'X'/'R'` silently matches zero rows. `sample_rows(order_items, 5)` resolves it before the query ever runs — cheap, and it prevents a whole class of invisible errors.

### The agent loop: reason → act → observe

Reason about what's needed → call a tool with typed arguments → read the structured result and decide if the goal is met → repeat, bounded by a hard policy:

- Cheap inspection before generation; always `validate_sql` before `execute_sql`; EXPLAIN before executing anything of unknown cost; re-inspect on error before regenerating.
- **Stopping conditions:** success → return; **≤ 8 total tool calls** (bounds inspection + ≤3 repair attempts) → abstain; max cost/wall-clock → stop; repeated identical error → stop, don't loop.

### Treat generated SQL as untrusted code

An LLM writing SQL against your warehouse is remote code execution with extra steps:

| Threat | Guard |
| --- | --- |
| Runaway scan (unfiltered cross-join, terabytes) | EXPLAIN + byte/cost cap |
| Destructive ops (DROP/DELETE/UPDATE) | Read-only role — no DML/DDL |
| Data exfiltration (PII in results/logs/prompt) | Column masking + row-level security |
| Injection via NL or sampled data | Scoped role + validation; treat retrieved rows as untrusted |

Guards are enforced **outside the prompt**, in the execution wrapper: assert read-only → EXPLAIN and check the byte cap → execute under a tenant-scoped read-only role with row limit, timeout, and max bytes. Permissions and masking are properties of the **database role**, not the SQL the model wrote — so the model can't overreach even if it tries (least-privilege role, row-level security for tenant isolation, dynamic column masking for PII, and no raw PII ever entering the prompt or logs).

## Phase 4 — Schema Retrieval (RAG for schema)

You cannot put 5,000 tables in the prompt: at a terse ~80 tokens each that's ~400K tokens before any DDL, samples, or the question itself — and even if it fits, accuracy collapses in a sea of irrelevant tables (wrong-table joins, lost-in-the-middle, higher cost and latency). Retrieval isn't an optimization here; it's what makes the system possible. The task becomes: *given a question, return the smallest set of tables + definitions sufficient to answer it.*

### Why schema retrieval isn't document retrieval

- **Structured, not prose** — you retrieve tables, columns, and definitions, and the foreign-key graph between them carries as much meaning as the nodes.
- **Names are cryptic** — `dim_cust_x` won't match "customers." Elements need descriptions and synonyms before they're findable.
- **Precision in the final context** — one wrong table poisons a join, so you fund a wide, high-recall net first, then rerank and FK-expand down to the precise set.
- **Examples are gold** — similar past Q→SQL pairs teach dialect and house conventions no schema exposes.

### The pipeline: cast wide, then sharpen

1. **Hybrid retrieve + Top-K** — lexical (BM25) + vector search over enriched "schema cards"; hybrid beats either alone, and it's cheap.
2. **Rerank** — a cross-encoder sharpens the top candidates to precision.
3. **FK-expand** — pull in directly-joined dimension & bridge tables that vector similarity misses entirely (see below).
4. **Few-shot + assemble** — add similar Q→SQL examples, prune to a tight context.

**Why vector search alone fails on joins:** "EMEA last quarter" retrieves `orders`, `order_items`, `products` by similarity — but *not* `dim_region` or `date_dim`, because dimension/filter tables rarely resemble the question text at all. The fix is structural, not a bigger embedding model: from the high-similarity seeds, traverse foreign-key edges one hop to pull in the tables needed to join. **Rank, then expand along the graph.**

### Retrieve definitions too, not just tables

The highest-value retrieval target isn't a table — it's the **definition of the business term**. A retrieved metric definition (e.g., `revenue → SUM(order_items.net_amount) WHERE status <> 'R'`, grain `order_item`) converts "guess what revenue means" into a lookup, feeding Phase 5's grounding step directly.

### Operate it: freshness + recall@K

Schemas drift constantly — re-index incrementally on DDL/catalog events, with a nightly full rebuild as backstop; a stale index causes *silent* retrieval misses. Measure **table-recall@K**: of the tables the gold SQL needs, how many are in the top-K? A missing table dooms the query outright, so recall (not precision) is the north star at the candidate stage — precision is recovered downstream by reranking + FK-scoping.

## Phase 5 — Schema Reasoning & SQL Generation (the deepest dive)

Retrieval found the tables; execution runs safely. In between sits the reasoning core — the part interviews are won and lost on, because a **syntactically perfect query can still use the wrong metric, join, or grain and look completely right.**

### Four ways a perfect-looking query is wrong

| Failure | Example | Defense |
| --- | --- | --- |
| Wrong metric | "revenue" = gross vs. net vs. bookings, guessed silently | Ground on the semantic layer |
| Wrong join path | Joins through the wrong key/table, quietly changing which rows are counted | Plan joins on the FK graph |
| Wrong grain | A one-to-many join fans out rows; a SUM double/triple-counts | Aggregate at the right grain |
| Hallucinated column | Invents `orders.revenue` or another plausible-but-absent column | Validate against the catalog |

### A. Disambiguate before you generate

Resolve five axes before writing SQL — each unresolved one is a chance to be confidently wrong: **metric** (which "revenue"?), **grain** (per order/item/customer?), **time** (calendar or fiscal quarter, which timezone?), **entity** (SKU vs. product family?), **filters** (which countries count as "EMEA"?).

Policy: **ask** when the ambiguity changes the answer materially, stakes are high (reporting/finance tier), or no default exists — **assume-and-state** when a sensible default exists in the semantic layer or it's exploratory tier, always echoing the assumption ("Assuming net revenue and fiscal quarters…"). The one unacceptable option is a **silent guess**.

### B. Ground meaning in a semantic layer, then plan the join path

A curated `semantic_layer.yml` maps business terms to SQL deterministically: metric → exact SQL expression, grain → correct GROUP BY, join keys + cardinality → safe joins, sensible defaults → fewer clarifications. Grounding converts "guess what revenue means" into a lookup — the single biggest lever against confidently-wrong answers.

Then plan joins by walking the **foreign-key graph** from the fact table outward (a star, not a free-associated chain) — e.g., `order_items` (fact hub) → `dim_products`, → `orders` → `dim_region`, `date_dim`.

**The fan-out trap:** a 1:N join duplicates rows on the "one" side. Summing `orders.order_total` after joining to `order_items` triple-counts an order with 3 items ($100 → $300). Fix: **aggregate every measure at the grain it lives at** — sum `net_amount` at item grain directly, or pre-aggregate each measure to a common grain in its own CTE before joining measures that live at different grains (item revenue, order shipping, customer count).

### C. Decompose into checkable CTEs, then validate grounding

One monster query is hard to get right and impossible to debug. Layer it: **(1) filter early** at fact grain, **(2) aggregate at grain** (fan-out can't happen here), **(3) join names last** (dimensions don't affect the measure), **(4) rank & limit**. Each CTE is independently checkable — when execution errors, the agent pinpoints the failing step instead of regenerating the whole query.

Before running anything, parse the generated SQL and check every table/column reference against the retrieved catalog (deterministically, with code — e.g. via `sqlglot`'s AST, not by asking the model). An unknown reference returns a `Repair` with a suggestion ("did you mean `order_items.net_amount`?") instead of ever reaching the warehouse. This is layered with input constraints (show only retrieved, real columns), optional constrained decoding, and bounded repair — cheap-to-strict defense in depth.

### D. Self-correct, verify, and know when to abstain

The agent reads the structured error (Phase 3), isolates the failing CTE, and repairs — **bounded to ≤3 attempts**, then abstain. Map error type to targeted repair rather than random regeneration:

| Error | Likely cause | Repair |
| --- | --- | --- |
| Syntax/parse error | Dialect mismatch | Re-emit with correct grammar |
| Unknown column/table | Hallucination or bad retrieval | Re-ground; re-retrieve; use suggestion |
| Empty result set | Wrong enum/format in a filter | `sample_rows` to fix the literal |
| Suspicious numbers | Fan-out or wrong grain | Re-check joins; pre-aggregate |
| Timeout/cost abort | Scan too large | Add filters; narrow date range |

Before trusting a result that *ran*, verify it: row-count sanity, value-range sanity, EXPLAIN/plan sanity, cross-check vs. a known metric, and — when a single sample could be confidently wrong — **self-consistency**: sample N candidates at temperature > 0, execute each, cluster by result set, return the plurality. Agreement across diverse candidates is itself a confidence signal, with no reference answer required.

Combine disambiguation certainty, retrieval quality, verification results, and repair history into a **calibrated confidence score** (calibrated against held-out execution accuracy, never the model's own self-rating): high → answer + SQL + assumptions; medium → answer, flag the shakiest assumption; low → **abstain with a specific, answerable question**. An agent that knows the boundary of its own competence is more trusted than one that always answers.

### E. How you know it works: execution accuracy

Matching SQL text is the wrong metric — run gold and generated SQL and compare **result sets**, not text (two different queries can be equally correct). Calibrate against public benchmarks (Spider, BIRD), but the real test set is your own schema and traffic: mine query logs for Q→SQL pairs, label them, and regression-gate every deploy against that set. Execution accuracy can false-positive (different logic, same result by coincidence on a small test DB) — guard against it with deliberately tricky test data: refunds, multi-item orders, edge dates.

**The eval set is a flywheel:** every production failure or abstention gets triaged, labeled, and routed into both the eval set (prevents regression) and the few-shot store (teaches the next query) — a static benchmark rots, but this loop makes the system measurably harder to break the same way twice.

## Worked example: "Top 5 products by revenue in EMEA last quarter?"

This is the deck's running example, traced through every phase end to end — the 60-second answer to give under pressure.

**1. Clarify + resolve obvious ambiguity.** Operational-reporting tier assumed. "Revenue" and "last quarter" are ambiguous but a semantic-layer default exists for both, so we assume-and-state rather than ask: net revenue, fiscal quarter.

**2. Retrieve.** Hybrid lexical+vector search over schema cards returns high-similarity seeds: `orders`, `order_items`, `dim_products`. FK-expand one hop pulls in `dim_region` and `date_dim` — tables that look nothing like "EMEA" or "last quarter" in text, but are needed to join.

**3. Ground + plan the join path.** The semantic layer resolves "revenue" → `SUM(order_items.net_amount)` with `status <> 'R'` (net of refunds), grain `order_item`. The join path is a star from the fact table: `order_items` → `orders` (carries `region_id`, `order_date_key`) → `dim_region` (EMEA) and `date_dim` (fiscal quarter) — not a path through `customers`, which also exists but isn't what this metric intends.

**4. Decompose and generate.** Filter at item grain first (no fan-out risk), aggregate to product grain, join product names last, then rank and limit:

```sql
WITH items_emea AS (                           -- filter at item grain
  SELECT oi.product_id, oi.net_amount
  FROM order_items oi
  JOIN orders o     ON o.order_id = oi.order_id
  JOIN dim_region r ON r.region_id = o.region_id
  JOIN date_dim d   ON d.date_key = o.order_date_key
  WHERE r.region_name = 'EMEA'
    AND d.fiscal_quarter = '2024-Q1'            -- resolved at gen-time
    AND oi.status <> 'R'                        -- exclude refunds -> net
),
by_product AS (                                 -- aggregate at product grain
  SELECT product_id, SUM(net_amount) AS revenue
  FROM items_emea
  GROUP BY product_id
)
SELECT p.product_name, b.revenue                -- join names, then rank
FROM by_product b
JOIN dim_products p ON p.product_id = b.product_id
ORDER BY b.revenue DESC
LIMIT 5;
```

Why it's correct: revenue is summed at item grain (no fan-out from the `orders` join), refunds are excluded, and names are joined only after aggregation — so the join to `dim_products` can't change the measure.

**5. Validate grounding.** Parse the SQL and check every `table.column` reference against the retrieved catalog before anything executes — every reference here resolves to a real, retrieved column. Then `EXPLAIN` to confirm the plan and estimated cost are within budget.

**6. Execute.** Run under a tenant-scoped read-only role, against a replica, with row/byte/time limits enforced outside the model's control.

**7. Verify.** Row count is sane (5 rows, as asked), revenue values are non-negative and plausible, and — if available — the total cross-checks against a known EMEA revenue figure.

**8. Answer or abstain.** High confidence (clean disambiguation, full retrieval recall, clean first generation, passing verification) → return the table, the SQL, and the stated assumptions ("net revenue, fiscal Q1"). Any of those signals weak → lower confidence, flag the shakiest assumption, or abstain with a specific question ("which revenue definition, and does EMEA include these 12 countries?").

## Phase 6 — Reliability & Scale

The correctness pipeline stays unchanged; it's wrapped in a reliability layer that changes cost, latency, and behavior under stress — never the answer.

### Size the load first

50K questions/day → ~3/s at peak → ~12/s LLM calls (×4 calls/question) → ~6/s warehouse queries (×2 for EXPLAIN+run). Token volume, not request count, is what providers actually cap: ~750M tokens/day works out to ~2.6M tokens/minute at peak. At only ~6 qps, the real limits are provider token-per-minute caps and warehouse concurrency, not raw throughput — so the fixes are a gateway (batch, cache prompts, rate-limit, tier models) and result caching, not a bigger model.

### Cache at three tiers — each with its own key and invalidation

| Tier | Purpose | Key | Invalidate |
| --- | --- | --- | --- |
| Schema-embedding cache | Skip re-embedding | Schema-element hash | On DDL / catalog change |
| Generated-SQL cache | Skip generation for repeat asks | Question + schema version + prompt/model version | On semantic-layer / example-store change |
| Result cache | Skip the warehouse scan entirely | SQL text + data-version tag | On data version, never a timer alone |

Getting the cache key right (SQL text + data-version for results; question + schema-version for SQL) is what separates "fast" from "fast and still correct."

### One gateway for every model call

Route all LLM traffic through one service: batch & queue to absorb bursts and respect token-per-minute limits; model tiering (cheap/fast for intent & routing, stronger for reasoning & repair); fallback/retry on provider error; and one choke point for per-tenant cost and latency observability. Rate limits and cost controls only work if every call goes through one door.

### Govern cost and time per question

Hard per-question ceilings on LLM calls, tokens, warehouse bytes, and wall-clock — enforced, not hoped for — with cost attribution by tenant + question id and timeout ladders that degrade rather than hang.

### Degrade gracefully — never fail loudly or lie

Under load, fall through progressively cheaper rungs, but never to a confidently wrong one: **(1)** serve from cache if an equivalent question was answered recently, **(2)** drop to a cheaper path (skip optional verification, faster model, flag lower confidence), **(3)** return a partial answer and state what was skipped, **(4)** queue or abstain with a clear reason. Every rung stays trustworthy.

### Multi-tenancy and freshness

Isolation lives in the warehouse role and the index, not the prompt: per-tenant scoped role + row-level security, separate vector index and semantic layer per tenant, tenant-scoped cache keys, and fair-use quotas so one noisy tenant can't starve the rest. Freshness: incremental re-index on DDL/catalog events (not just nightly), versioned schema + semantic layer with version-keyed cache busting, and recall monitoring — a drop signals a stale or broken index.

## The tradeoff ledger

Every design choice in this system has a cost. Naming what you gave up — before being asked — is one of the most senior things you can do in a design discussion.

| Decision | What we gave up | Why it's right |
| --- | --- | --- |
| Retrieve schema, don't dump it | Recall risk; an index to maintain | Only way to fit context + focus the model |
| Ground on a semantic layer | Up-front curation effort | Removes the #1 confidently-wrong failure |
| Read-only + execution guards | No writes; per-call latency | Generated SQL is untrusted code |
| Bounded self-correction | Extra LLM calls & cost | Robustness without infinite loops |
| Verify, then abstain if unsure | Lower answer rate; extra queries | Trust beats coverage |
| Cache aggressively by tier | Staleness risk to manage | Makes the economics & latency viable |

## Strong signals vs. common mistakes, by phase

| Phase | Strong signal | Common mistake |
| --- | --- | --- |
| Tool use & execution | Treats generated SQL as untrusted code; errors returned as data | Relying on the prompt to enforce safety |
| Tool use & execution | EXPLAIN-before-execute with cost caps | Executing before validating or estimating cost |
| Tool use & execution | Least-privilege, per-tenant role + row-level security | Giving the agent write access to the warehouse |
| Schema retrieval | Rank-then-expand along the FK graph | Pure vector similarity, no FK expansion |
| Schema retrieval | Recall@K tracked — a missing table dooms the query | Dumping the top-200 tables "to be safe" |
| Schema retrieval | Enriches cryptic names with aka + descriptions | Embedding raw names with no enrichment |
| Schema reasoning | Grounds meaning in a semantic layer | Letting the model guess metric definitions |
| Schema reasoning | Plans joins on the FK graph; sums at grain | Summing an order-grain value across items (fan-out) |
| Schema reasoning | Validates grounding against the catalog | Trusting a result because it ran |
| Schema reasoning | Bounded self-correction, then abstains | Looping forever, or bluffing a number |
| Reliability & scale | Caches by tier with correct invalidation | Caching results without version-based busting |
| Reliability & scale | Degrades gracefully — never confidently wrong | Failing loudly, or hallucinating under load |

### Red flags that cap your score in a design interview

- **Diagram vomit** — opening with a 12-box architecture and no baseline reads as memorized, not reasoned.
- **No correctness story** — treating SQL generation as "just prompt the LLM," with no grounding, grain, or verification.
- **Never abstains** — a system that always answers ignores the core risk this whole deck is about.
- **Hand-waving the hard parts** — "retrieval is just RAG," "joins are easy" — skipping exactly where this problem is hard.
- **Ignoring cost & safety** — no budgets, no read-only role, an unbounded agent hitting production data.
- **Passive** — waiting to be asked the next question instead of driving the design forward.

## Self-test: check your understanding

Try answering each before reading the answer underneath.

**1. In production, which failure erodes trust the most: a syntax error, a slow query, a confident-but-subtly-wrong answer, or an honest "can't find a relevant table"?**

> The confident, plausible, subtly wrong answer. Loud failures cost a retry; the silent one produces a clean table that lands in a deck with no exception fired, so no one checks it.

**2. A generated query would scan 4 TB. Where's the right control point — in the prompt, after execution, before execution, or in code review?**

> Before execution: a dry-run/EXPLAIN estimates cost and a guard aborts with a hint. Prompting reduces but can't guarantee; detecting after execution means you already paid for the scan; human review can't keep up with interactive traffic.

**3. Retrieval returns `orders`, `order_items`, `products` for "top products by revenue in EMEA last quarter," but the query can't filter by region or date. Why, and what's the fix?**

> `dim_region` and `date_dim` carry no text resembling the question, so no embedding model ranks them highly. The fix is structural, not a bigger model or more candidates: FK-expand one hop from the retrieved fact tables to pull in the directly-joined dimensions.

**4. Revenue-by-product returns 3× the real total, with orders joined to order_items and `SUM(orders.order_total)` grouped by product. Why?**

> The 1:N join fans out each order's row once per line item, so `order_total` gets summed once per item. The data isn't wrong and it isn't replication lag — it's deterministic: sum the measure at the grain it actually lives at (`SUM(order_items.net_amount)`), or pre-aggregate before joining.

**5. Three repairs later, verification still fails on a finance-tier request. What now — keep retrying, return the latest result, abstain with specifics, or silently widen the filters?**

> Abstain with a specific, answerable question and the partial finding. The retry budget exists precisely to stop endless retrying; returning an unverified result or quietly widening filters to force a pass is exactly the confidently-wrong failure the system is built to prevent.

**6. Traffic spikes 10× overnight with rising latency and errors. Highest-leverage fix: a bigger model, caching + a rate-limiting gateway, a faster language, or per-user dedicated clusters?**

> Caching + a rate-limiting gateway. At 10×, the first things to break are LLM provider rate limits and warehouse concurrency — exactly what batching, queuing, tiering, and result/SQL caching relieve, and they ship in days. A bigger model raises cost and latency; language choice isn't the bottleneck; per-user clusters are expensive and unnecessary once caching removes repeat scans.

**7. Revenue numbers "look too high" since 9am with no errors firing. Where do you look first — restart the service, roll back the model, hunt a schema/semantic-layer change, or raise the confidence threshold?**

> Hunt a schema or semantic-layer change plus stale caches. "Ran fine, wrong numbers" is the classic fan-out/definition-regression signature: someone edited a metric definition, a migration re-graded a table, or a join's cardinality changed, and a stale generated-SQL or result cache is still serving the broken query. Restarting and rolling back the model are both blind — the model didn't change, the data contract did. Raising the threshold hides the symptom while wrong numbers keep flowing.

## Three-point recap

1. **Correctness is a systems property**, not a cleverer prompt — retrieval, grounding, grain discipline, and verification have to work together.
2. **Ground meaning; get grain right** — a semantic layer resolves what "revenue" means; grain discipline is what stops the silent fan-out double-count.
3. **Trust = verify + abstain** — the system that knows when it doesn't know, and says so, is the one people rely on.

**The 60-second answer to "design a text-to-SQL system":** clarify → retrieve → ground → generate → validate → execute → verify → answer or abstain.

**If you have more time to study:** multi-turn analytical conversations, learning from analyst feedback on past answers, cross-database federation, and a fine-tuned SQL model running behind the same guardrails.
