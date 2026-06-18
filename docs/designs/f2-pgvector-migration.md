# F2 — pgvector + Voyage Migration (design)

> One-line: Replace the per-project DigitalOcean Knowledge Base with a single shared pgvector store + Voyage AI embeddings, behind two swappable seams (retrieval backend, embedding provider), so semantic retrieval runs everywhere (dev/test/prod), self-host is a config swap, and the rest of Phase F has one seam to build on.
>
> Status: **proposed** (design-first). Phase F2 in [PLAN.md](../../PLAN.md). Format/voice mirror [d1-override-resolution.md](./d1-override-resolution.md). This document is the output of the SDLC design front-half (Gate 1 threat model done, Gate 2 security review done and folded in) — it spells out the implementation-time gates in [§13](#13--sdlc--delivery-plan) and the decisions that still need your call in [§14](#14--open-decisions-need-your-call).

---

## 1. Problem / Motivation

Semantic retrieval today lives entirely in the DigitalOcean Knowledge Base (`DoKnowledgeBaseService`, `gte-large-en-v1.5`, **one KB per project**). Per-project KBs give *physical* tenant isolation and server-side chunking for free, but the arrangement is blocking us three ways:

1. **It is the hardest dependency to self-host.** An on-prem enterprise deployment cannot run DO KB. F2 is the seam that turns self-host into a config swap instead of a refactor (per `self-host-readiness` memory: keep the DO-KB / Anthropic / webhook seams clean; pgvector is the pre-work).
2. **dev/test silently degrade to `ILIKE`.** There is no real semantic recall outside prod, so retrieval-dependent features (contradiction detection, blueprint co-pilot, agent replies, project routing) ship having never been exercised against real embeddings.
3. **It blocks the rest of Phase F.** Every downstream retrieval feature needs one swappable seam to build on.

F2 replaces DO KB with **pgvector (dev/test/prod) + Voyage AI embeddings by default**, behind **two independent seams** (retrieval backend, embedding provider), re-embeds existing eligible content, and retires DO KB on a staged, rollback-safe cutover.

**The dominant new risk is tenant isolation.** Collapsing N per-project KBs into one shared pgvector table **destroys the physical isolation boundary** and replaces it with a *logical* one — a `WHERE` clause over a shared, ANN-indexed table. ANN search returns the globally nearest neighbors, **not** tenant-scoped rows: a naive `ORDER BY embedding <=> :q LIMIT k` can return a competitor's PRD. Simultaneously, **Voyage is a new sub-processor** receiving raw artifact + session text. This design starts from those two risks (the Gate 1 threat model), not from provider mechanics — pgvector and Voyage are both well-trodden; the math is not the hard part.

---

## 2. Goals & non-goals

**Goals**

- One **retrieval-backend seam** (Seam A) and one **embedding-provider seam** (Seam B), each swappable by env, so cloud (pgvector + Voyage), self-host (pgvector + local model), and rollback (DO KB) are all config changes.
- `ArtifactSearchService.search(...)` public contract **unchanged** — all 6 callers untouched.
- pgvector in **every** environment → kills the dev/test ILIKE gap.
- **Default-deny / fail-closed**: unconfigured ⇒ lexical, **never** a silent Voyage call.
- Tenant / sensitivity / scan / soft-delete filters enforced **in-query**, proven by a CI sentinel.
- Re-embed existing eligible content; retire DO KB on a staged, rollback-safe cutover.

**Non-goals**

- Chunking-algorithm *tuning* (window/overlap constants). The chunker *location* and *size bound* are designed here; the optimal token window is a follow-up. The bound (`MAX_INPUT_TOKENS = 8000`) is **chosen as a safe value, not tuned** — flagged explicitly, not silently resolved.
- The rake backfill task *internals*. Runbook, idempotency contract, and cost model are here; the task code is gated on D3.
- A non-pgvector self-host storage backend (not required today).
- GraphQL / JSON:API reshaping of search responses — the legacy hash is preserved verbatim.

---

## 3. Security constraints this design is built from (Gate 1 restated)

Every constraint maps to one enforcement point. The **sentinel test (a3) is what proves a1/a2/a4 hold** — not the prose.

| # | Constraint | Where enforced |
|---|------------|----------------|
| a1 | org/project/status/`deleted_at`/source filters are SQL `WHERE` in the **same** query as the ANN order-by; never post-filtered in Ruby; **all** of them on denormalized `artifact_chunks` columns (no `artifacts` subquery) | §6 `PgvectorBackend#search` |
| a2 | recall holds under restrictive `WHERE`; never filter **after** the ANN `LIMIT` | §6 ANN-driven subquery + iterative-scan + bounded `max_scan_tuples` + recall benchmark (§12) |
| a3 | cross-org sentinel: near-identical A/B at large k → 0 cross-tenant rows | §12 `test/sentinels/retrieval_tenant_isolation_test.rb` |
| a4 | read-gate re-checks org from the Pundit-resolved scope (keep **both** read + write gates) | §5 backend receives a **pre-scoped relation**, never raw params |
| a5 | re-apply sensitivity / private-repo / **scan-clear** / **capture-current** gate at WRITE time; **no chunk row** for ineligible content | §8 `embeddable?` re-checked at the seam |
| b1 | Voyage egress = explicit USER DECISION (DPA / sub-processor) | §14 D1; org-level `embedding_no_egress` (§10.5) |
| b2 | sensitivity gate runs **before** the embedding HTTP call | §8 gate in `Embedding::Service`, upstream of provider |
| b3 | query text also egresses → gate derived from the query **source**, not an opt-in caller flag | §5/§8 `query_source:` → untrusted forces lexical, no provider call |
| b4 | self-host local provider by env, **zero** external calls; fail-closed if unconfigured | §7 selection + `NullProvider` default-deny |
| c1 | `VOYAGE_API_KEY` in credentials/env; never in logs/Sentry/traces | §7 `VoyageProvider` client rules |
| c2 | REST client never logs request body (artifact text) or `Authorization` | §7 client rules |
| d1 | per-org rate limit + monthly token budget; **atomic** reservation before egress; hard per-request ceiling; chunk oversized | §8 `Embedding::Service` + `Embedding::Budget` |
| d2 | bound query length before embedding; validate `limit` 1..20 | §5/§8 input guards |
| d3 | pgvector params **bound** (typed binds), element-validated; no user input in vector literal / `SET` | §6 binding rules |
| d4 | backfill throttled (429 backoff, 12h batch), idempotent | §11 backfill; §14 D3 |
| e1 | dual-write inherits the SAME gates; query path too | §8 + §11 |
| e2 | backfill re-embeds only currently-eligible artifacts; never soft-deleted | §11 `embeddable?` re-run at backfill |
| e3 | destroy purges **both** stores from Stage 0/1; teardown sweeps both | §10.4 destroy callback + §11 |
| e4 | rollback-safe: additive schema; feature-flag read path reverts to DO KB/lexical | §10 + §11 |

---

## 4. Key facts (from code scout, verified against source)

- **Single integration point already exists.** `ArtifactSearchService.search(...)` is the only semantic-retrieval call site; **6 callers** depend on it (`ArtifactsController#search`/`#context`, `AgentResponseJob`, `ProjectRoutingService`, `BlueprintCopilotService`, `ContradictionDetectionService`). The swap is caller-invisible **iff** the return shape is preserved exactly.
- **Three of the six callers are fed untrusted text** and were analyzed explicitly: `AgentResponseJob` (inbound email/Slack body), `ProjectRoutingService.route(text:)` (the inbound router — raw inbound subject+body), and `ContradictionDetectionService` (embeds artifact content that can be an untrusted inbound artifact). The egress decision is therefore **derived from the query's SOURCE inside the service** (§5 `query_source:`), default-deny — not an opt-in per-caller boolean.
- **Return shape callers depend on TODAY:** `Array<{ artifact:, content:, score: }>` — `artifact` = `Artifact` instance (or nil on metadata mismatch), `content` = String excerpt, `score` = Float(0..1) | nil. Plus `service.search_mode` → `"semantic"` | `"lexical_fallback"`. **This is the contract; the new seam returns it unchanged.**
- **Write path:** `ProcessArtifactJob` → `ContentSanitizer.clean` (verified: the method is `.clean`, *not* `.call`) → `DoKnowledgeBaseService.upload_and_index`. Session path: `SessionVectorizerJob` → `SessionVectorizationService` (gated by `vectorizable?` at `session_vectorization_service.rb:73`: analysis complete + not sensitive + repo public). Sessions create a `source:"session"` Artifact, then enqueue `ProcessArtifactJob` — the paths converge.
- **Session content is rendered at READ time, capture-aware.** `SessionVectorizationService#build_transcript` calls `SessionTranscriptRenderer.render(@session, roles:)` with an explicit code comment: *"Shared renderer honors capture_mode at read time — content stored before server-side enforcement must not leak into the project KB."* **This corrects the draft's capture-drift premise** (see §8 note): the leak only exists if we persist `content` on the chunk at embed time and never re-render. The fix is to re-render through `SessionTranscriptRenderer` at embed time — no new column, no org-wide re-embed-storm callback.
- **Write gate exists, read gate is weaker.** `vectorizable?` blocks sensitive/private at index time. `ArtifactSearchService#build_scope` filters org/project/status/`deleted_at` but does no sensitivity re-check. a4 requires keeping the write gate **and** the in-query org scope on read. `Artifact#scan_clear?` (`artifact.rb:119` → `!scan_required? || scan_status == "clean"`) is a distinct, asserted gate.
- **Destroy leak.** `artifact.destroy!` (acts_as_paranoid) has **no KB cleanup callback** today. `DoKnowledgeBaseService#delete_file` **does exist** (`do_knowledge_base_service.rb:96,142`) — only the destroy-callback wiring is missing. Under pgvector this becomes a stale-embedding leak unless `deleted_at IS NULL` is in every search and chunks are purged on destroy. **Both stores fixed from Stage 0**; the implementer **wires the existing `delete_file`**, does not rebuild a deleter.
- **DO KB over-fetches then post-filters in Ruby** — the exact anti-pattern pgvector MUST NOT copy (a1).
- **Net-new `organizations` columns required and DO NOT exist today.** The egress opt-out and budget gate reference org-level state with no current migration/attribute. Added explicitly in §10.5.
- **`capture_mode` is NOT a column.** It is a derived method (`organization.rb:103` → `effective_telemetry_settings["capture_mode"]`), `CAPTURE_MODES = %w[full structured minimal]`. There is no `after_update` hook to observe it directly (see §8 and the capture-drift correction in §11).
- **Schema ready.** `enable_extension "vector"` already present; no embedding storage yet. Test DB is pg18 + pgvector. An `artifact_chunks` table was created (`20260208033717`) and **dropped** (`20260208034606`) when DO KB owned chunking — reintroduced deliberately (§10).
- **No first-class Voyage Ruby SDK.** Thin REST client, mirroring `AnthropicService` / `AnthropicBatchService`.
- **`input_type` asymmetry DO KB hides:** Voyage wants `"document"` at ingest and `"query"` at search. The seam threads this; DO KB never exposed it.
- **`strong_migrations` is NOT installed** (no gem in `api/Gemfile`, no initializer — verified). The draft wrongly cited an existing `algorithm: :concurrently` enforcement. **Adding `strong_migrations` + initializer is a Stage 0 deliverable** (§10.4 / §11), not an assumed pre-existing gate. `CONCURRENTLY` index creation is correct on its own merits regardless.
- **No `Registry` pattern exists** anywhere in `api/app/` (verified). We therefore use a **plain env-keyed factory function**, not a mutable global singleton (§7) — simpler to test, simpler to diagnose.

---

## 5. Architecture

```
                          callers (6, unchanged)
                                  │
                                  ▼
                     ArtifactSearchService.search(...)        ← unchanged public contract
                                  │   builds org/project-scoped relation = READ GATE (a1/a4)
                                  │   derives query egress from query SOURCE (b3/e1)
                                  ▼
                          Retrieval.backend (Seam A)          ← Retrieval.backend_for(env)  (pure fn of ENV)
              ┌───────────────────┴───────────────────────────┐
              ▼                                                 ▼
        PgvectorBackend                               DoKnowledgeBaseBackend
        (default; also self-host)                     (transitional / rollback)
              │
              └──────────── embed_document / embed_query ──────┐
                                  ▼                            │
                    Embedding::Service (Seam B front door)     │   (DO KB embeds server-side →
                        │  GATES LIVE HERE: eligibility +       │    Seam B unused for that backend)
                        │  size + ATOMIC budget, BEFORE egress  │
              ┌─────────┼──────────────┬──────────────────┐
              ▼         ▼              ▼                   ▼
        VoyageProvider LocalProvider NullProvider    (no provider call)
        (Voyage REST)  (TEI/BGE-M3)  (fail-closed)
```

**Two independent seams.** Seam A (retrieval backend) and Seam B (embedding provider) swap separately: `PgvectorBackend + VoyageProvider` (cloud default), `PgvectorBackend + LocalProvider` (self-host), `DoKnowledgeBaseBackend` (own server-side embedding, Seam B unused). This is the lean cut — exactly the seams self-host needs.

> **Folded recommendation — no `LocalBackend`.** The draft listed a `LocalBackend` that was an explicit alias of `PgvectorBackend` "to keep the axes explicit." Deleted. Self-host is documented as **`PgvectorBackend + LocalProvider`**; the self-host axis lives entirely at Seam B. A no-op class is exactly the speculative layer to cut.

> **Folded recommendation — no `Registry`.** No registry/singleton pattern exists in the codebase. We use a pure env-keyed factory (`Retrieval.backend_for(Rails.env)` / `Embedding.provider_for(Rails.env)`) returning a memoized instance, rather than a mutable configurable singleton. A pure function of ENV is trivially overridable per-test (`with_env`) and has no global state to reset between tests.

**Gate placement.** All eligibility/size/budget gates live in `Embedding::Service` (Seam B front door), upstream of every provider, so they run before any egress regardless of which backend calls them. The retrieval backend (Seam A) receives a **pre-scoped `ActiveRecord::Relation`** from `ArtifactSearchService` and **never sees raw params** — its only job is "rank within this relation." The two seams compose: Seam A calls Seam B for the query vector; Seam B owns the egress decision.

### Caller-invisible contract (Seam A)

```ruby
# app/services/retrieval/backend.rb — abstract contract
module Retrieval
  class Backend
    Result = Data.define(:artifact, :content, :score)  # maps 1:1 to legacy { artifact:, content:, score: }

    # scope:  pre-scoped AR::Relation — org + project + status:"ready" + deleted_at:nil ALREADY applied (a1/a4)
    # query:  String, already length-bounded by caller (d2)
    # filters: { artifact_type:, tags:, source_scope:, injectable_only: }
    # limit:  Integer 1..20, validated by caller (d2)
    # allow_egress: Boolean — resolved by ArtifactSearchService from query_source (b3/e1).
    #               false ⇒ backend MUST NOT egress the query; lexical only.
    # => { results: Array<Result>, mode: "semantic" | "lexical_fallback" }
    # INVARIANT: every returned row is reachable from `scope`. A backend MUST NOT widen scope.
    def search(scope:, query:, filters:, limit:, allow_egress: false) = raise NotImplementedError
    def index_document(artifact) = raise NotImplementedError   # idempotent; honors embeddable?. => :indexed | :skipped
    def delete_document(artifact) = raise NotImplementedError  # best-effort. => :deleted | :noop
  end
end
```

`ArtifactSearchService` keeps its exact public signature and re-maps the seam result to the legacy hash. **Egress is derived from the query SOURCE, inside the service:**

```ruby
class ArtifactSearchService
  # query_source: :trusted   — only an explicit :trusted opts INTO egress
  #               :untrusted — inbound email/Slack/foreign-artifact text — NEVER egresses (DEFAULT, fail-closed)
  def search(query:, organization:, project: nil, artifact_type: nil, tags: nil,
             scope: nil, limit: 5, injectable_only: false, query_source: :untrusted)
    limit = limit.clamp(1, 20)                                  # d2
    allow_egress = (query_source == :trusted)                   # b3/e1 — egress is a property of the SOURCE
    rel = build_scope(organization:, project:, artifact_type:, tags:, scope:, injectable_only:)  # a1/a4 read gate
    out = Retrieval.backend.search(
      scope: rel, query:, limit:, allow_egress:,
      filters: { artifact_type:, tags:, source_scope: scope, injectable_only: },
    )
    @search_mode = out[:mode]
    out[:results].map { |r| { artifact: r.artifact, content: r.content, score: r.score } }  # exact legacy shape
  end
  attr_reader :search_mode
end
```

The three untrusted-fed callers (`AgentResponseJob`, `ProjectRoutingService`, `ContradictionDetectionService`) pass `query_source: :untrusted` explicitly. Any caller that does not assert `:trusted` gets lexical, not a silent Voyage call — this kills the "index everything then filter later" anti-pattern on the query path.

---

## 6. Concrete backends (Seam A)

```ruby
# app/services/retrieval/pgvector_backend.rb
module Retrieval
  class PgvectorBackend < Backend
    def search(scope:, query:, filters:, limit:, allow_egress: false)
      rel = apply_filters(scope, filters)                       # artifact_type/tags/source/injectable — all WHERE on rel
      embedding = (Embedding::Service.embed_query(query) if allow_egress)  # b3 + b4: untrusted/no-provider → lexical, ZERO egress
      return { results: lexical(rel, query, limit), mode: "lexical_fallback" } if embedding.nil?

      conn = ActiveRecord::Base.connection
      conn.exec_query("SET LOCAL hnsw.ef_search = #{Integer(ann_ef_search)}")           # Integer() ⇒ no injection (d3)
      conn.exec_query("SET LOCAL hnsw.iterative_scan = relaxed_order")                  # a2 (pgvector ≥ 0.8)
      conn.exec_query("SET LOCAL hnsw.max_scan_tuples = #{Integer(ann_max_scan_tuples)}")  # a2 — BOUND

      { results: hydrate(ann_query(rel, embedding, limit)), mode: "semantic" }
    end
  end
end
```

### The a2 query — ALL scope predicates on denormalized chunk columns

The draft routed project/status/source scope through `c.artifact_id IN (rel.select(:id).to_sql)` — a subquery against `artifacts` beside the vector `ORDER BY`. The planner can apply that `IN (subquery)` **after** the HNSW inner `LIMIT` — the exact "filter-after-ANN-LIMIT" failure a2 forbids, and under `relaxed_order` + bounded `max_scan_tuples` a selective project filter can starve recall. **Fixed:** every scope predicate lives on the denormalized `artifact_chunks` columns (org, project, source, soft-delete) inside the indexed-scan `WHERE`, mirroring the `org_id = $2` treatment. `ArtifactSearchService` still hands a pre-scoped relation; `PgvectorBackend` translates that relation's tenant predicates onto the chunk table rather than re-querying `artifacts`.

```ruby
# app/services/retrieval/pgvector_backend.rb (cont.)  — all literals are BOUND typed params (d3)
private

def ann_query(rel, embedding, limit)
  inner_limit = [limit * INNER_FANOUT, MAX_INNER_LIMIT].min      # generous candidate pool, bounded
  s = scope_predicates(rel)                                      # {org_id:, project_ids:, sources:, status_ready:} from rel

  sql = <<~SQL
    WITH ann AS (
      SELECT c.artifact_id, c.content, (c.embedding <=> $1) AS dist
      FROM artifact_chunks c
      WHERE c.organization_id = $2          -- tenant WHERE, same statement as ANN order-by (a1)
        AND c.project_id = ANY($3)          -- project scope on DENORMALIZED column (NOT an artifacts subquery)
        AND c.source = ANY($4)              -- source scope ("sessions" etc.) on denormalized column
        AND c.deleted_at IS NULL            -- soft-delete on denormalized column
        AND c.embedding IS NOT NULL
        AND c.embedding_model = $5          -- §11.4: steady-state single-model; mixed-model window is shadow-read only
      ORDER BY c.embedding <=> $1           -- *** ANN distance is the ONLY sort → HNSW drives, iterative-scan engages (a2)
      LIMIT $6                              -- inner candidate limit (BOUND)
    ),
    best AS (
      SELECT DISTINCT ON (artifact_id) artifact_id, content, dist
      FROM ann
      ORDER BY artifact_id, dist            -- dedup to best chunk per artifact, on the small `ann` set only
    )
    SELECT artifact_id, content, (1 - dist) AS score
    FROM best
    ORDER BY dist                           -- *** final ordering by similarity, NOT artifact_id (correctness)
    LIMIT $7
  SQL

  ActiveRecord::Base.connection.exec_query(
    sql, "ann_query",
    [bind_halfvec(embedding), s[:org_id], pg_array(s[:project_ids]), pg_array(s[:sources]),
     Embedding::Service.active_model_id, inner_limit, limit]    # typed binds; nothing string-built (d3)
  )
end

INNER_FANOUT    = 8       # candidate pool = limit*8 …
MAX_INNER_LIMIT = 500     # … capped so a huge corpus can't unbound the inner scan
```

**Why this satisfies a2:** the `ann` CTE's only `ORDER BY` is `embedding <=> $1`, so Postgres uses the HNSW index and `hnsw.iterative_scan = relaxed_order` keeps walking the graph until the inner `LIMIT` is filled **with rows already passing the tenant `WHERE`** (every filter is a denormalized column inside the same indexed scan), bounded by `hnsw.max_scan_tuples`. Dedup and final score-sort run on the tiny `ann` set, so the returned rows are the **most similar**, one best chunk per artifact. The original `DISTINCT ON (artifact_id) … ORDER BY artifact_id, distance LIMIT k` was broken three ways (leading `artifact_id` sort defeats HNSW; `LIMIT` returns lowest-id not most-similar; iterative-scan never engages). **This must be proven by the recall benchmark (§12) before it is called a mitigation.**

**`max_scan_tuples` bound (a2/d3).** `relaxed_order` trades ordering for recall and stops at `hnsw.max_scan_tuples` (pgvector default 20000). Under a highly selective org filter (a small tenant in a large shared table), the scan can hit the cap before finding `limit` in-tenant rows — returning `<k` (acceptable under-recall, never cross-tenant). It is an **explicit bound param** (`RETRIEVAL_HNSW_MAX_SCAN_TUPLES`, `Integer()`-cast); the per-org **partial HNSW index** (§10.3, optional) restores full recall for tiny-tenant-in-huge-corpus cases the benchmark flags. `relaxed_order` may return rows slightly out of global distance order, but the tenant `WHERE` is *inside* the indexed scan, so it can **never** return a cross-tenant row.

```ruby
# binding helpers (d3) — typed binds, element-validated; NEVER string-interpolated provider data
def bind_halfvec(vec)
  raise ArgumentError, "bad embedding width" unless vec.length == Embedding::Service::EMBEDDING_DIMS
  vec.each { |f| raise ArgumentError, "non-finite embedding element" unless f.is_a?(Float) && f.finite? }
  ActiveRecord::Relation::QueryAttribute.new(   # $1 referenced as $1::halfvec in SQL — parameterized cast, no literal
    "qvec", "[#{vec.join(',')}]", ActiveRecord::Type::Text.new
  )
end
def ann_ef_search       = ENV.fetch("RETRIEVAL_HNSW_EF_SEARCH", "100")
def ann_max_scan_tuples = ENV.fetch("RETRIEVAL_HNSW_MAX_SCAN_TUPLES", "40000")  # bound; tune via §12 benchmark
```

A provider returning `NaN`/`Infinity`/short vectors becomes a caught `ArgumentError`, not a raw-SQL failure — satisfying d3 ("break the trust assumption") verifiably.

- **`DoKnowledgeBaseBackend`** — wraps today's `DoKnowledgeBaseService` behind the same contract. Transitional + rollback target through cutover (§11). It still over-fetches + post-filters in Ruby; **`PgvectorBackend` MUST NOT replicate it.**

**a2 mitigations, in order:** (1) ANN-driven subquery (above); (2) `hnsw.iterative_scan = relaxed_order` (pgvector ≥ 0.8); (3) bounded `max_scan_tuples` + raised `ef_search`; (4) optional per-org partial HNSW index (§10.3), decided by the §12 benchmark. **MUST-NOT:** the DO KB over-fetch-then-Ruby-filter pattern.

---

## 7. Embedding-provider seam (Seam B)

A provider is a **stateless adapter**: it knows nothing about orgs, sensitivity, or budgets (enforced by `Embedding::Service` before the provider is called). It only turns text into normalized vectors.

```ruby
# app/services/embedding/provider.rb (duck-typed interface)
module Embedding
  class Provider
    # input_type :document | :query  (REQUIRED — no default; forces caller intent — Voyage asymmetry)
    # => EmbeddingResult(vectors:, model_id:, dimensions:, token_usage:) | nil (unavailable → fail closed)
    # Invariants: output length == input length, order-preserved; each vector EMBEDDING_DIMS (1024) finite
    #   L2-normalized floats; model_id is a stable string stored alongside every vector (mixed models not comparable);
    #   NEVER logs `texts` or auth material.
    def embed(texts:, input_type:) = raise NotImplementedError
    def model_id   = raise NotImplementedError
    def available? = raise NotImplementedError
  end
end
```

**`VoyageProvider` (default, hosted)**
- **Model:** `voyage-code-3` @ **1024 dims**, `output_dtype: float`. Highest-value crewkit content is analyzed session markdown (narrative + tool I/O + code); `voyage-code-3` is the strongest single model for code-heavy mixed content. `voyage-4` ($0.06 vs $0.18) is the documented downgrade lever — both 1024-dim, so switching is a `model_id` change + re-embed, not a schema change. **Pinned, versioned model id (`EMBEDDING_MODEL_ID`) — never "latest".**
- **input_type:** `document` at ingest, `query` at search.
- **Batching:** ingest uses the **Voyage Batch API** (JSONL → Files → batch job, 12h window, −33%, ≤100K inputs/batch), mirroring `AnthropicBatchService`; output order ≠ input order → match on `custom_id = "chunk:<chunk_id>"`. Search uses the **sync** endpoint.
- **Retry/backoff:** 3 attempts, exponential backoff + jitter on `429`/`5xx`. The per-org budget (§8) is the primary throttle.
- **Secrets (c1/c2):** `VOYAGE_API_KEY` from `credentials.dig(:voyage, :api_key)` or ENV; added to the Sentry `before_send` denylist + `config.filter_parameters`. Thin `Net::HTTP`/Faraday client that **never** logs `request.body` or `Authorization`; logs only model, input_type, token count, status, latency. **One key, rotatable without redeploy.** Blast radius documented: one key = all embedding egress for the deployment.

**`LocalProvider` (self-host)**
- **Backend:** HuggingFace **Text Embeddings Inference (TEI)** serving **BGE-M3 @ 1024 dims** → dimensional parity with Voyage, so the same `halfvec(1024)` column + HNSW index serve both and switching is config-only. (Qwen3-Embedding-0.6B @1024 is the documented alternative; avoid 2560/4096-dim models — they break parity and force a re-embed.)
- **Egress:** `EMBEDDING_LOCAL_URL` → TEI inside the customer network. **Zero external calls.** When `EMBEDDING_PROVIDER=local` the Voyage client is **never constructed** — no code path reaches `api.voyageai.com` (the SecAudit self-host assertion).

**`NullProvider`** — `available? => false`, `embed => nil`. The default-deny floor.

---

## 8. Embedding flow + gates (Seam B front door)

`Embedding::Service` is **default-deny**: no path produces or transmits a vector without passing the gate, enforced **inside the service** (belt-and-suspenders — a future caller that forgets the gate fails closed, not open-to-Voyage).

```
caller (job / search) ──► Embedding::Service
      ├─ 1. eligibility gate (sensitive? private repo? scan-clear? deleted? org no-egress?) ─► REJECT, no egress
      ├─ 2. input-size bound + chunk (≤ MAX_INPUT_TOKENS; hard per-request ceiling)
      ├─ 3. per-org rate-limit + monthly token budget — ATOMIC reservation (write path only) ─► REJECT, lexical
      └─ 4. provider.embed(...)  (only reached if 1–3 pass)
```

```ruby
# app/services/embedding/service.rb — the gate layer (provider-agnostic)
module Embedding
  class Service
    EMBEDDING_DIMS     = 1024          # hard invariant; provider returning other width RAISES
    MAX_INPUT_TOKENS   = 8_000         # per-CHUNK cap below Voyage's 32K hard cap (d1). BOUND CHOSEN, window deferred (non-goal).
    MAX_REQUEST_TOKENS = 200_000       # d1 HARD per-artifact ceiling — one giant artifact can't blow the monthly cap
    MAX_QUERY_CHARS    = 2_000         # query bound before egress (d2) — a CHAR guard, not a token equivalent

    def self.active_model_id = provider.model_id

    # DOCUMENT side — chunks, embeds, persists artifact_chunks rows. Idempotent. => :indexed | :skipped
    def self.embed_artifact(artifact)
      return :skipped unless artifact.embeddable?               # a5/b2/e1/e2 — re-checked at the seam, every time
      prov = provider
      return :skipped unless prov.available?                    # b4 fail-closed

      # a5 capture-correctness: re-render session content through the read-time, capture-aware renderer
      # at embed time (NOT the stored artifact.content). No new column; no org-wide re-embed callback.
      chunks = chunk(ContentSanitizer.clean(artifact.embedding_input), MAX_INPUT_TOKENS)  # d1 size bound; .clean (verified)
      est = est_tokens(chunks)
      return :skipped if est > MAX_REQUEST_TOKENS               # d1 hard per-request ceiling

      reservation = Budget.reserve(artifact.organization, est)  # d1 ATOMIC reserve BEFORE egress
      return :skipped if reservation.nil?                       # over cap → unembedded, search degrades to lexical
      begin
        result = prov.embed(texts: chunks.map(&:text), input_type: :document)
        Budget.settle(reservation, actual: result.token_usage)  # reconcile estimate → actual
        persist_chunks(artifact, chunks, result)                # delete-then-insert on (artifact_id, chunk_index, model_id)
        :indexed
      rescue => e
        Budget.release(reservation)                             # provider failed → give the reservation back
        raise e
      end
    end

    # QUERY side — no budget gate (cheap + latency-sensitive). => Array<Float>(1024) | nil
    def self.embed_query(query)
      prov = provider
      return nil unless prov.available?                         # b4
      return nil if query.blank? || query.length > MAX_QUERY_CHARS  # d2
      prov.embed(texts: [query.strip], input_type: :query).vectors.first
    end

    def self.retract_artifact(artifact)                         # e3 — destroy-purge (BOTH stores via §10.4 callback)
      ArtifactChunk.where(artifact_id: artifact.id).delete_all
      :deleted
    end
  end
end
```

### The eligibility gate — delegate, don't fork

> **Folded recommendation — single source of truth.** `scan_clear?` (Artifact) and the sensitive/private-repo logic (`vectorizable?` in `SessionVectorizationService`) already exist. The gate **delegates** to them rather than re-listing the predicates, so the gate cannot drift from the live write gate.

```ruby
# app/models/concerns/embeddable.rb
module Embeddable
  def embeddable?
    return false if deleted_at.present?                         # e2 never embed soft-deleted
    return false unless status == "ready"
    return false unless scan_clear?                             # a5 — asserted EXPLICITLY (ready+scan_status:error exists)
    return false if organization.embedding_no_egress?           # org-level opt-out (self-host / DPA refusers) — §10.5
    return false if llm_session && !llm_session.vectorizable_for_embedding?  # DELEGATES to live sensitive/repo gate
    true
  end

  def embedding_input
    # a5 capture-correctness: session-derived artifacts re-render through the capture-aware renderer
    # at embed time; non-session artifacts use stored fields.
    return SessionTranscriptRenderer.render(llm_session, roles: TRANSCRIPT_ROLES) if llm_session
    [name, summary, content].compact.join("\n\n")
  end
end
```

> **SEC-FIX corrected — capture-drift premise was a misread.** The draft claimed session content is "rendered at creation, never re-rendered, so a later capture tighten leaks the old transcript," and on that basis invented a persisted `captured_under_mode` column + a `capture_mode_stale?` check + an org-wide `after_update` re-embed-storm callback. **Verified false:** `SessionVectorizationService#build_transcript` already renders via `SessionTranscriptRenderer.render`, which honors `capture_mode` **at read time** (explicit code comment). The drift vector exists *only* if we persist `content` on the chunk at embed time and never re-render. The simple, reuse-first fix is to **re-render through the existing capture-aware renderer at embed time** (`embedding_input` above) and gate on the session's *current* sensitivity. **No new column. No org-wide callback.** (There is no `capture_mode` column to observe — and now we don't need one.)

> **Reconciliation — chunking location.** We adopt a dedicated **`artifact_chunks` table** (1:N), not a single `embedding` column on `artifacts`. A single column cannot represent a multi-chunk document and would force whole-document embedding — worse recall and a guaranteed breach of Voyage's 32K-token input cap on large session markdown. `Embedding::Service` owns the chunker; the *size bound before egress* is enforced here, the *window/overlap tuning* is a non-goal (§2).

### Per-org budget + rate limit (d1) — atomic, store-contingent

> **SEC-FIX — atomicity is store-contingent, and the draft's `AlertService` does not exist.** Two corrections: (1) The draft asserted "increment is atomic in Solid Cache" as if universal. It is not. **test uses `:null_store`** (`increment` returns nil — the §12 atomicity test cannot run against it), dev uses `:memory_store`, and a self-host on `:file_store` is **not atomic** — silently reopening the TOCTOU the fix claims to close. (2) `AlertService.budget_exceeded` does not exist (verified: only `SlackNotifier`, `EventPublisher`, `UpgradeNotificationService`, `SecurityEventService`). We reuse **`EventPublisher`** (the documented ESB seam for domain events) — no new alerting service for one call site.

The atomicity guarantee is therefore made **explicit and contingent**: `Embedding::Budget` requires a cache store that supports atomic `increment`/`decrement`. When the configured store does **not** (null/file/memory), the budget falls back to **fail-closed degraded mode** — it refuses to embed past a conservative per-process estimate rather than silently allowing unbounded spend. Production and the atomicity test both run against a **real increment-capable store** (Solid Cache / Redis-backed); the §12 test explicitly swaps the store rather than running against `:null_store`.

```ruby
# app/services/embedding/budget.rb
module Embedding
  Reservation = Data.define(:org_id, :amount, :period_key)

  class Budget
    # ATOMIC reserve before egress; closes the read-then-write TOCTOU across concurrent jobs.
    # CONTINGENT: requires a store supporting atomic increment. atomic_store? guards self-host/test stores.
    def self.reserve(organization, est_tokens)
      cap = organization.embedding_monthly_token_cap || DEFAULT_CAP   # §10.5 column
      return reserve_degraded(organization, est_tokens, cap) unless atomic_store?  # fail-closed on non-atomic stores
      k = key(organization)
      Rails.cache.increment(k, 0, expires_in: 32.days)               # ensure counter exists
      new_total = Rails.cache.increment(k, est_tokens)               # ATOMIC reserve
      if new_total > cap
        Rails.cache.decrement(k, est_tokens)                         # roll back over-reservation
        EventPublisher.publish("embedding.budget_exceeded", organization_id: organization.id,
                               attempted: est_tokens, cap: cap)       # ESB seam, not a new AlertService
        return nil                                                   # caller → :skipped → lexical
      end
      Reservation.new(org_id: organization.id, amount: est_tokens, period_key: k)
    end

    def self.settle(reservation, actual:)                            # reconcile estimate → actual
      delta = actual.to_i - reservation.amount
      Rails.cache.increment(reservation.period_key, delta) unless delta.zero?
    end
    def self.release(reservation)                                    # provider failed / skipped → give it back
      Rails.cache.decrement(reservation.period_key, reservation.amount)
    end

    def self.atomic_store? = Rails.cache.respond_to?(:increment) && !Rails.cache.is_a?(ActiveSupport::Cache::NullStore)
  end
end
```

---

## 9. Config / selection (per-env + flags) — b4/c1

Env-driven, **default-deny** (pure factory, no singleton):

```ruby
# config/initializers/retrieval.rb
module Retrieval
  def self.backend = @backend ||= backend_for(ENV.fetch("RETRIEVAL_BACKEND", default_backend_for(Rails.env)))
end
module Embedding
  def self.provider = @provider ||= provider_for(ENV.fetch("EMBEDDING_PROVIDER", default_provider_for(Rails.env)))
end
# EMBEDDING_PROVIDER unset → "null" (fail-closed → lexical). RETRIEVAL_BACKEND unset → safe per-env default.
```

| Env | `RETRIEVAL_BACKEND` | `EMBEDDING_PROVIDER` | Behavior |
|-----|---------------------|----------------------|----------|
| cloud prod (pre-cutover) | `do_kb` | n/a (DO embeds) | today, unchanged |
| cloud prod (dual-write) | `do_kb` + `RETRIEVAL_DUAL_WRITE=pgvector` | `voyage` | write both, read DO KB, shadow-read pgvector (§11) |
| cloud prod (post-cutover) | `pgvector` | `voyage` | pgvector only; DO KB kept as rollback |
| dev / test | `pgvector` | `local` / `null` | **real semantic locally — fixes ILIKE degradation** |
| **self-host** | `pgvector` | `local` (TEI/BGE-M3) | **zero external calls** (b4) |
| any, misconfigured | falls to `do_kb` / lexical | `null` | fail-closed, never silent Voyage |

---

## 10. Schema & migration

> **Reconciliation — dimension/index.** **`halfvec(1024)` + `halfvec_cosine_ops` + HNSW.** Voyage's 1024 dims fit pgvector's 2000-dim `vector` index cap with room to spare — halfvec is not *required*, but fp16 halves storage/index memory at negligible recall loss; 1024 keeps Voyage↔local a config-only swap; cosine matches L2-normalized vectors. HNSW (no training step, builds on near-empty tables) is correct for incremental `SessionVectorizerJob` ingest; IVFFlat would be wrong here.

### 10.1 `artifact_chunks` table

```
artifact_chunks
  id                bigint  PK
  artifact_id       bigint  NOT NULL  FK -> artifacts(id)
  organization_id   bigint  NOT NULL   # DENORMALIZED (§10.2)
  project_id        bigint  NOT NULL   # DENORMALIZED (§10.2)
  chunk_index       int     NOT NULL   # 0-based ordinal within artifact
  content           text    NOT NULL   # chunk text (snippet return + reindex)
  token_count       int                # tokens sent to embedder (budget/telemetry)
  embedding         halfvec(1024)      # NULL until embedded (resumable backfill)
  embedding_model   string  NOT NULL   # provenance; mixed models NOT comparable (§11.4)
  embedding_version int     NOT NULL DEFAULT 1   # bump to force re-embed on model/dim change
  source            string  NOT NULL   # MIRRORED from artifact.source (in-query "sessions" scope filter)
  deleted_at        datetime           # MIRRORED soft-delete; acts_as_paranoid
  created_at / updated_at
```

A prior `artifact_chunks` (`20260208033717`) was dropped (`20260208034606`) when DO KB owned chunking; reference the drop migration in the new migration comment to avoid "created by mistake" confusion.

### 10.2 Why denormalize `organization_id` / `project_id` / `source` / `deleted_at`

a1 requires tenant/status filters as `WHERE` predicates **in the same query as the ANN order-by**. If those columns lived only on `artifacts`, the ANN query needs a JOIN/subquery, and the planner can apply the HNSW `LIMIT` **before** the filter — returning <k or cross-tenant rows (the §6 bug). Denormalizing puts every filter directly beside `ORDER BY embedding <=> :q` on the indexed table. Mirrored columns are kept consistent via `Artifact` callbacks (§10.4).

### 10.3 Indexes

```sql
-- ANN index, built CONCURRENTLY in a SEPARATE migration AFTER backfill (cheaper than per-insert HNSW churn)
CREATE INDEX CONCURRENTLY index_artifact_chunks_embedding_hnsw
  ON artifact_chunks USING hnsw (embedding halfvec_cosine_ops) WITH (m = 16, ef_construction = 64);

-- composite tenant-scope support so the planner combines filter + ANN scan
CREATE INDEX CONCURRENTLY index_artifact_chunks_scope
  ON artifact_chunks (organization_id, project_id, source) WHERE deleted_at IS NULL;

-- OPTIONAL per-org partial HNSW (a2 tiny-tenant-in-huge-corpus mitigation; built only if §12 benchmark flags)
-- CREATE INDEX CONCURRENTLY index_artifact_chunks_embedding_hnsw_org_<id>
--   ON artifact_chunks USING hnsw (embedding halfvec_cosine_ops) WHERE organization_id = <id> AND deleted_at IS NULL;
```

Plus `(artifact_id)` FK index, a `(artifact_id, chunk_index)` unique index (ordinal integrity), `(deleted_at)` for paranoia scoping.

**Status/readiness (a):** chunk rows are created **only at the `ready` transition**; if an artifact leaves `ready`, its chunks are soft-deleted / embeddings nulled. The `deleted_at` mirror guards post-`ready` soft-deletes.

### 10.4 Migration mechanics + destroy-purge (BOTH stores) + strong_migrations

- **schema.rb churn trap:** one **dedicated** migration for the table + a **separate** one for the HNSW index (post-backfill). After running, `db:schema:dump` in isolation and hand-verify the diff is **only** the new lines + the existing `enable_extension "vector"`.
- **Extension floor:** migration is idempotent (`enable_extension "vector" unless extension_enabled?`) and **raises loudly** if pgvector `extversion` < **0.7** (`halfvec`) or < **0.8** (iterative scan) rather than create an unindexable/under-recalling column. `down` never drops the extension. (The boot/CI assertion in §10.6 is the *primary* version guard; this raise is a backstop.)
- **`strong_migrations` is a Stage 0 deliverable** (verified absent today). Add the gem + initializer **before** writing the F2 migrations, so `add_index` enforcement (`algorithm: :concurrently` → `disable_ddl_transaction!`) is real, not assumed. Org-tier policy treats `strong_migrations` as a block-deployment gate.
- **`ArtifactChunk` model:** `acts_as_paranoid`, `belongs_to :artifact`, denormalized columns set from the artifact at create.
- **`Artifact` callbacks (mirror integrity + bug fixes):**
  - `after_update`: if `organization_id/project_id/source` change → propagate to chunks; if `status` leaves `ready`, `deleted_at` set, **or `scan_status` leaves clear** → soft-delete chunks.
  - **`after_destroy` / `before_real_destroy` → purge BOTH stores.** Deletes chunk rows (`Embedding::Service.retract_artifact`) **and** wires the existing `DoKnowledgeBaseService#delete_file` (verified to exist — not rebuilt) whenever DO KB holds the document (throughout the dual-write/cutover window).
  - `SessionVectorizationService#retract` (sensitivity-flip) calls the same retract path.
- **No `captured_under_mode` column, no capture re-embed callback.** Resolved at the gate by re-rendering at embed time (§8) — corrects the draft.

> **e3 — DO-KB destroy leak is a Stage 0 prerequisite, not Stage 5 teardown.** DO KB is the authoritative read source until Stage 4, so a destroy that purges only pgvector during the weeks-long window leaves user-deleted content retrievable via the live DO KB read path. `DoKnowledgeBaseService#delete_file` already exists; only the destroy-callback wiring is missing. Wiring it is a **Stage 0 deliverable**, and the `after_destroy` callback purges **both** stores from the moment dual-write begins.

### 10.5 `organizations` column migration (net-new; required by b1 + d1)

`embeddable?` and `Budget` reference org columns that **do not exist today** (verified: no model attribute / schema entry). Without this migration the gates `NoMethodError` or fail open.

```ruby
# db/migrate/XXXX_add_embedding_controls_to_organizations.rb  (additive, e4)
class AddEmbeddingControlsToOrganizations < ActiveRecord::Migration[8.1]
  def change
    # SAFE DEFAULT: false on cloud (egress allowed only once D1/DPA signed). A DPA-refuser or self-host
    # tenant can force true per-org; self-host also forces local provider (can't reach Voyage at all).
    add_column :organizations, :embedding_no_egress, :boolean, null: false, default: false
    add_column :organizations, :embedding_monthly_token_cap, :bigint, null: true  # nil → Budget::DEFAULT_CAP
  end
end
```

Both additions are **additive** (e4) and listed in Acceptance. No `captured_under_mode` is added — capture-correctness is handled by re-rendering at embed time (§8).

### 10.6 pgvector ≥ 0.8 — verified boot/CI assertion (not a checkbox)

The entire a2 mitigation depends on `hnsw.iterative_scan = relaxed_order` (pgvector ≥ 0.8) and `halfvec` (≥ 0.7). **Prod pgvector version is unconfirmed** (test DB is pg18; prod unknown). If prod is < 0.8 the `SET LOCAL` is silently ignored, iterative-scan never engages, and the ANN query degrades to top-k-then-filter under-recall — a correctness regression that passes CI on pg18 and fails only in prod. **FIX:** a **boot-time + CI assertion** that queries `extversion` from `pg_extension` and **hard-raises** if < 0.8 (a Stage 0 deliverable). This runs *before* the read path goes live; the migration's extension-floor raise alone runs too late (only at migrate time, not on every boot).

---

## 11. Embedding flow wiring + cost model

### 11.1 Index / query wiring
- **Document side:** `ProcessArtifactJob` (sessions converge here via `SessionVectorizerJob`) → `ContentSanitizer.clean` → `Embedding::Service.embed_artifact` (gate → re-render+chunk → atomic budget reserve → Batch embed → settle → persist chunks) → `status: ready`.
- **Query side:** `ArtifactSearchService#search` → (egress derived from `query_source`) → `Embedding::Service.embed_query` (sync, bounded, gated) → `PgvectorBackend` runs the in-query tenant-filtered ANN with the query vector as a typed bound param.

### 11.2 Re-embed / idempotency
- **Re-analysis:** `SessionVectorizationService#refresh` deletes old rows then re-embeds; key = `(artifact_id, chunk_index, model_id)`; delete-then-insert ⇒ a retried job converges. Because session content is **re-rendered at embed time** through the capture-aware renderer, a re-analysis automatically picks up a tightened `capture_mode` — no separate capture-drift trigger needed.
- **Backfill:** re-embeds **only currently-eligible** artifacts (re-runs `embeddable?` at backfill time — including `scan_clear?` and the live sensitivity gate), **excludes `deleted_at`**, skips rows already at the current `model_id`/`embedding_version`. Resumable. Throttled (429 backoff, 12h Batch window). **Runbook internals gated on D3.**

### 11.3 Cost model

Pricing (Voyage, 2026): `voyage-code-3` = **$0.18/1M** sync, **~$0.12/1M** batched (−33%); `voyage-4` = $0.06/$0.04 (downgrade lever); 200M-token one-time free tier.

| Symbol | Meaning | How to get it (run before sign-off) |
|---|---|---|
| `A` | eligible uploaded artifacts | `Artifact.where(deleted_at: nil, status:"ready").where.not(source:"session").count` |
| `S` | eligible session artifacts | `Artifact.where(source:"session", deleted_at: nil).count` |
| `T̄_doc` | avg tokens/artifact (post-chunk, summed) | sample mean of `content` token length |
| `n_new/mo` | new eligible/month | artifact `created_at` trend |
| `q/mo` | searches/month | `ArtifactSearchService` call count (6 callers) |
| `T̄_q` | avg query tokens | sample (queries are char-bounded by `MAX_QUERY_CHARS`, which is **not** a token cap — see note) |

```
backfill_tokens = (A + S) × T̄_doc ;  backfill_$ = backfill_tokens/1e6 × $0.12 (batched)
ingest_$/mo     = n_new/mo × T̄_doc /1e6 × $0.12   (batched, document side)
query_$/mo      = q/mo × T̄_q /1e6 × $0.18         (sync, query side — no batch discount)
monthly_$       = ingest_$/mo + query_$/mo
```

> `MAX_QUERY_CHARS = 2000` is a **char** guard, not a token bound; `T̄_q` is measured in tokens independently. Voyage's 32K per-input cap is comfortably safe regardless.

Worked example (illustrative — replace with live counts): `A+S = 50,000 × 1,500 tok = 75M tok` → **within the 200M free tier ⇒ $0** one-time (worst case ~$9). Steady-state ≈ **$1/mo**, ~$10/mo at 10×. The binding backfill constraint is **throttle/throughput**, not dollars. Self-host (`LocalProvider`) = **$0 marginal**; the per-org rate limit still applies as a **DoS bound** on the local TEI endpoint.

### 11.4 Model-change strategy
A model change is a **full re-embed** — vectors from different models are **not comparable**. Every chunk carries `embedding_model`; the active model is pinned (`EMBEDDING_MODEL_ID`, never "latest"); the query path filters ANN to the active model (`embedding_model = $5` in §6). Procedure: set new id → backfill under it (delete-then-insert per `(artifact_id, chunk_index, new_model_id)`) → flip the read path only at **100% coverage** → purge old rows. **Steady-state is single-model, so the `embedding_model = $5` predicate is a no-op; the only mixed-model window is shadow-read (Stage 3), never the live read path** — so the extra equality predicate cannot starve live recall. (If a future design needs mixed-model live reads, add a partial HNSW index `WHERE embedding_model = '<active>'`.) Dimensional parity at 1024 keeps the column/index stable across swaps.

---

## 12. Security & tenant-isolation invariants (testable MUST / MUST-NOT) + testing strategy

### Invariants
- **MUST (a1)** — every search carries org + project + `status="ready"` + `deleted_at IS NULL` + `source` as bound `WHERE` predicates **in the same statement** as the ANN order-by, **all on denormalized `artifact_chunks` columns**. **MUST-NOT** route project/source scope through an `artifacts` subquery, and **MUST-NOT** over-fetch + filter in Ruby.
- **MUST (a2)** — recall holds under the tenant filter via the ANN-driven subquery (distance is the sole inner sort key) + iterative index scan + **bounded `max_scan_tuples`**. **MUST-NOT** lead the sort with `artifact_id` or apply the filter after the ANN `LIMIT`. **The recall benchmark MUST prove this before it is called a mitigation.**
- **MUST (a3)** — the cross-org sentinel is GREEN and lives in `test/sentinels/`.
- **MUST (a4)** — read gate re-derives org from the Pundit-resolved scope; write gate stays.
- **MUST (a5/b2)** — sensitive / private-repo / **non-scan-clear** content (and any session whose *current* capture/sensitivity excludes it) produces **no chunk row** and makes **zero** provider calls; the gate runs **before** egress; session content is **re-rendered through the capture-aware renderer at embed time**.
- **MUST (b3/e1)** — query text from an **untrusted source** is gated to lexical (no egress); egress is derived from the query source inside the service; `ProjectRoutingService` and `ContradictionDetectionService` pass `:untrusted`.
- **MUST (b4)** — `EMBEDDING_PROVIDER=local` makes **zero** external calls; unconfigured fails **closed** to lexical, never Voyage.
- **MUST (c1/c2)** — `VOYAGE_API_KEY`, `Authorization`, and request bodies never appear in logs/Sentry/traces.
- **MUST (d1)** — input ≤ `MAX_REQUEST_TOKENS` per artifact and ≤ 32K/chunk bounded **before** egress; per-org budget enforced via an **atomic reserve-before-egress** counter on an atomic-capable store (fail-closed degraded mode otherwise); rate limit enforced.
- **MUST (d2)** — query ≤ `MAX_QUERY_CHARS`; `limit` clamped 1..20.
- **MUST (d3)** — all pgvector params **bound** (typed binds; `$1::halfvec`), every vector element validated finite-Float + length `== EMBEDDING_DIMS` before binding; `ef_search`/`max_scan_tuples` `Integer()`-cast in `SET`.
- **MUST (e1/e2/e3)** — dual-write inherits the same gates; backfill skips soft-deleted and re-runs the gate; destroy purges **both** stores from Stage 0/1; teardown sweeps both.
- **MUST (e4)** — schema is additive (incl. the two `organizations` columns); read path is feature-flagged and reverts to DO KB/lexical with no data loss.

### Testing strategy
- **`test/sentinels/retrieval_tenant_isolation_test.rb` (a3, CI-gating, FAST):** seed cosine-≈0 artifacts in org A + org B, search as A at k=20 → assert **zero** org-B rows; repeat with a project filter inside one org. **Kept cheap** (a handful of vectors) so it stays in the always-on sentinels tier.
  > **Folded recommendation — move the 100k-corpus proof out of the sentinel.** The large-noise-corpus variant (tiny tenant under ≥100k other-org near-identical vectors) is the realistic under-recall/leak case but is far too slow for the per-PR sentinels tier (it would get disabled within a week). It lives in the **QA/benchmark tier** (recall benchmark below), not in `test/sentinels/`.
- **Query-source gate test (b3/e1):** `ProjectRoutingService` and `ContradictionDetectionService` searches → assert `embed_query` is **never** invoked (provider mocked, 0 calls) and `mode == "lexical_fallback"`; a `:trusted`-source search **does** embed.
- **DO-KB destroy-purge test (e3, Stage 0):** create artifact → index to DO KB (mocked) → `destroy!` → assert `DoKnowledgeBaseService#delete_file` was invoked AND chunk rows purged. Runs from Stage 0.
- **Budget atomicity test (d1):** **run against a real increment-capable store** (swap `Rails.cache` to Solid Cache / memory-int-atomic in the test; `:null_store` cannot exercise it). Spin N concurrent `embed_artifact` against an org whose cap admits only M<N → assert exactly M embed, counter never exceeds cap; assert a single artifact estimated `> MAX_REQUEST_TOKENS` is skipped; assert the non-atomic-store path takes the fail-closed degraded branch.
- **Bind/validation test (d3):** provider stub returns a vector containing `Float::NAN` / wrong length → assert `bind_halfvec` raises `ArgumentError` (no SQL executed).
- **Secret-leak test (c1/c2):** stub the Voyage call, capture logs/Sentry breadcrumbs, assert no `VOYAGE_API_KEY`, no `Authorization`, no artifact/query body.
- **Gate tests (a5):** sensitive / non-`scan_clear?` / capture-excluded session → `embed_artifact` returns `:skipped`, **0** provider calls, **no chunk row**; assert session content is taken from `SessionTranscriptRenderer.render` at embed time, not stored `content`.
- **Backfill idempotency + deleted-exclusion (e2/d4):** re-run → no duplicate chunk rows; `deleted_at` rows never embedded; gate re-evaluated at backfill.
- **pgvector version assertion (§10.6):** boot/CI assertion hard-raises on `extversion < 0.8`.
- **Recall benchmark (a2, QA tier):** labeled set, recall@k under org filter vs unfiltered, **including the tiny-tenant-in-huge-corpus (≥100k) case** — **must be run to validate the §6 query as a real a2 mitigation**; decides whether the per-org partial index (§10.3) is needed.
- **Caller-contract test:** `ArtifactSearchService.search` return shape (`{artifact:, content:, score:}`) + `search_mode` **byte-identical** across `do_kb` and `pgvector` backends.
- **dev/test ILIKE-gap closure:** dev/test set `RETRIEVAL_BACKEND=pgvector` + `EMBEDDING_PROVIDER=local`; a hermetic **`StubProvider`** yields stable 1024-dim vectors with **zero** network so the semantic path is exercised in CI. Lexical-fallback tests set `EMBEDDING_PROVIDER=null`.

---

## 13. SDLC / delivery plan

This run produced the design front-half. Implementation follows the team "security sandwich":

1. **Gate 1 — ThreatModel: DONE.** Restated as the §3 constraint table; this design starts from it.
2. **This design doc — DONE (Gate 2 input).**
3. **Gate 2 — security-expert adversarial design review: DONE, folded in.** All blocking issues resolved (a2 query rewrite + denormalized scope plumbing; `max_scan_tuples` bound; org columns; capture-drift premise corrected; query-source default-deny; Stage 0 DO-KB destroy wiring via existing `delete_file`; typed binds; atomic budget made store-contingent + `EventPublisher` instead of nonexistent `AlertService`; `strong_migrations` + pgvector-≥0.8 boot assertion as Stage 0 deliverables). Worthwhile recommendations folded; declines noted inline (`LocalBackend` deleted, `Registry` replaced by pure factory, 100k sentinel moved to QA tier). **Design must be re-confirmed signed-off at Gate 2 before any code** (the a2 query and the §14 decisions are the gating items).
4. **Staged Build, gates per stage:** each stage (§11 runbook Stages 0–5) implements → component gates (`bin/rails test`, sentinels, lint/Brakeman, OpenAPI) → **adversarial-code-reviewer** pass → fix → next stage. Stage 0 (schema + seams + `strong_migrations` + pgvector assertion + DO-KB destroy wiring) is the gating prerequisite; no dual-write until its tests are green.
5. **QA:** recall benchmark under org filter (incl. tiny-tenant-in-huge-corpus); sensitive/scan/capture never-egress (0 provider calls, 0 chunk rows); backfill idempotent + skips `deleted_at`; caller-contract byte-identical.
6. **Gate 3 — SecAudit:** adversarial cross-tenant retrieval fails; destroy/retract purges chunks (both stores from Stage 0); scan-fail / capture-excluded never egress; self-host `local` makes zero Voyage calls; DPA sign-off.
7. **Release:** staged behind `RETRIEVAL_BACKEND` / `RETRIEVAL_DUAL_WRITE`; DO KB kept as rollback until pgvector recall + isolation validated in staging, then prod.

### Migration / cutover / retirement runbook (+ rollback)

```
Stage 0  DEPLOY  — schema (§10, incl. organizations columns §10.5) + seams behind flags.
                   STAGE 0 PREREQUISITES (all before dual-write): add strong_migrations gem+initializer;
                   add pgvector>=0.8 boot/CI assertion; wire DO-KB delete-on-destroy via existing delete_file
                   so destroy purges BOTH stores the instant dual-write starts.
                   RETRIEVAL_BACKEND=do_kb. No behavior change otherwise.
                   Go: migrations clean, schema diff isolated, version assertion green, DO-KB-destroy test green.
                   Rollback: revert migration (additive).
Stage 1  DUAL-WRITE — ProcessArtifactJob writes BOTH backends (pgvector guarded by embeddable? → same gates
                   incl. scan/re-render, e1). destroy purges BOTH stores. Reads still 100% DO KB.
                   Go: dual-write error rate <1%; sentinel green; ZERO sensitive/soft-deleted/non-scan-clear
                       chunk rows; destroy purges both stores.
                   Rollback: disable pgvector write; DO KB untouched.
Stage 2  BACKFILL — run cost estimator (D3) → approve ceiling → Batch backfill eligible corpus (§11.2).
                   Go: completes; chunk count reconciles; 0 sensitive/deleted embedded.
                   Rollback: truncate artifact_chunks; re-run later.
Stage 3  SHADOW-READ — live reads serve DO KB (authoritative); run PgvectorBackend.search in parallel on a
                   sample %, log {overlap@k, score_delta, mode, latency} under the org filter.
                   STRUCTURED LOG TAG: `f2.shadow` (operators grep) + Sentry transaction tag; overlap
                   threshold is D6 (a concrete number, not "≥ threshold").
                   Go: overlap >= D6 AND isolation sentinel green at large k.
                   Rollback: keep DO KB read; iterate ef_search / max_scan_tuples / partial index.
Stage 4  CUTOVER — RETRIEVAL_BACKEND=pgvector (STAGING first, validate isolation+recall, THEN prod).
                   DO KB kept warm (still dual-writing + destroy-purging) as rollback. Query-egress gate (b3) active.
                   Go: prod recall + p95 latency within budget; zero cross-tenant hits in canary.
                   Rollback: flip RETRIEVAL_BACKEND=do_kb instantly — both stores live, no data loss (e4).
Stage 5  TEARDOWN (after the D2 window) — stop DO KB dual-write; final reconciliation sweeps anything
                   deleted/reclassified during the window from BOTH stores (defense-in-depth — destroys already
                   purged live since Stage 0); delete DO KB data sources + Spaces objects; retire
                   DoKnowledgeBaseBackend. Go: reconciliation report clean.
                   Rollback: NONE past here — hence D2 honored + Stage 4 staging+prod-validated first.
```

---

## 14. Open decisions (need your call)

- **D1 — Voyage as a new sub-processor (b1). ✅ RESOLVED (2026-06-17): approve cloud-wide + DPA.** Voyage is approved for all cloud tenants with the sensitivity/scan/private-repo gate enforced before any egress; the org-level `embedding_no_egress` control ships (default `false` cloud, forced `true` for refusers), and `local` (BGE-M3/TEI) remains the self-host answer. **Action: sign the Voyage DPA + update the privacy notice (legal/ops, owner = user) before flipping `EMBEDDING_PROVIDER=voyage` in any cloud env.**
- **D2 — DO KB retirement window (e3). ✅ RESOLVED (2026-06-17): ~2-week soak.** DO KB stays dual-written + destroy-purged as rollback for **≈2 weeks** of green staging + prod metrics after shadow-read goes green, then Stage 5 teardown. **Action: name the teardown sign-off owner (user).**
- **D3 — Re-embed cost + backfill runbook (d4/e2). ✅ RESOLVED (2026-06-17): Batch API, count-first.** Backfill via Voyage **Batch API** — throttled, idempotent, eligible-only, deleted-excluded, resumable — with a **cost preflight** that runs the live `(A+S)` count and refuses to proceed past a configurable token/$ ceiling (expected $0 within the 200M free tier). **Action: run the preflight count at execution time.**
- **D4 — Default embedding model. ✅ RESOLVED (2026-06-17): `voyage-code-3`.** crewkit's corpus is mixed narrative + code (analyzed session markdown, transcripts, emails, code); the research found `voyage-code-3` the strongest single model for code-heavy mixed content, and it is already the implemented default (`EMBEDDING_MODEL_ID`, pinned, 1024-dim). `voyage-4` remains the documented cost-downgrade lever (a `model_id` change + re-embed, no schema change).
- **D5 — pgvector prod version. ✅ RESOLVED (2026-06-17).** Verified `production-crewkit-db` (Postgres **18.4**) exposes pgvector **0.8.1** as `pg_available_extensions.default_version` — so `CREATE EXTENSION vector` installs 0.8.1, satisfying both the a2 iterative-scan requirement (≥ 0.8) and the optional `halfvec` optimization (≥ 0.7). **No Stage-0 upgrade needed.** The extension is not yet `CREATE`d (expected — F2's migration creates it). Action retained: still ship the boot/CI `extversion ≥ 0.8` assertion so a future provider/version regression fails loudly rather than silently degrading to top-k-then-filter under-recall.
- **D6 — Shadow-read overlap go/no-go threshold. ✅ RESOLVED (2026-06-17): overlap@5 ≥ 0.8.** Stage 4 cutover gates on **overlap@5 ≥ 0.8** between DO KB and pgvector results on the shadow sample, sustained over the ~2-week Stage 3 window (D2), with the tenant-isolation sentinel green. Falls back to DO KB instantly via the `RETRIEVAL_BACKEND` flag if the bar isn't met.

> **All open decisions resolved 2026-06-17.** Remaining non-code actions before cutover: sign the Voyage DPA + privacy-notice update (D1) and name the teardown owner (D2). Build status: the flag-gated seam shipped in #201; the Batch-API backfill + dual-write enablement is the final code increment.

> **Silent-resolution disclosures (not decisions, flagged for honesty):** (a) the chunk size **bound** `MAX_INPUT_TOKENS = 8000` is a *chosen safe value*, with the window/overlap *tuning* deferred as a non-goal — not a tuned constant. (b) `capture_mode_stale?`-style directional logic was **removed**: capture-correctness is enforced by re-rendering at embed time through the existing read-time renderer, which is conservative in the safe direction (a loosened mode never leaks; a tightened mode is honored on the next embed). If a later optimization reintroduces a directional capture check, it MUST fail toward lexical, never toward egress.
