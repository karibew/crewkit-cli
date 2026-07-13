# F2 — pgvector + Voyage rollout runbook

> How to take F2 (merged in #199/#200/#201) from "built, default-off" to "pgvector +
> Voyage live in production." Design: [`docs/designs/f2-pgvector-migration.md`](../designs/f2-pgvector-migration.md).
>
> **Everything is flag-default-off today.** `RETRIEVAL_BACKEND=do_kb`,
> `EMBEDDING_PROVIDER=null` (→ lexical). Nothing below happens until you set the flags.

---

## Status

| Gate | State |
|---|---|
| Voyage provider verified against live API (voyage-code-3, 1024-dim, cosine) | ✅ done (2026-06-18, synthetic input) |
| **Voyage ToS §3(iii) opt-out** (no training use; post-opt-out content deleted after processing) | ✅ **done (2026-06-18)** |
| Voyage **DPA** signed + sub-processor list + privacy-notice update (D1) | ⏳ pending (legal/ops) |
| DO-KB teardown owner named (D2) | ⏳ pending |

---

## ⚠️ Data-governance gates — order matters

These are **blocking** and the ordering is load-bearing.

1. **§3(iii) opt-out FIRST, before any real customer content is embedded.** ✅ Done.
   The opt-out is **forward-only and irreversible**: content sent *before* opt-out keeps
   Voyage's **perpetual, irrevocable** training license with no clawback; content sent
   *after* is not used for training and is **deleted after processing**. So the opt-out
   had to precede any dual-write/backfill — which it now does. (It also **voids the
   200M-token free tier** — see cost below. Opt-out requires a payment method on file
   and **cannot be reversed**.)
2. **DPA + sub-processor list + privacy notice (D1).** ⏳ Voyage is a new sub-processor
   of customer data (transcripts, code, emails). Sign Voyage's DPA, add Voyage to the
   sub-processor list, update the privacy notice, and (if customer contracts require)
   give advance notice — **before** flipping `EMBEDDING_PROVIDER=voyage` in any
   customer-data environment.
3. **Name the teardown owner (D2)** who signs off DO-KB decommission after the soak.

> Sensitive sessions / private repos / non-scan-clear / `embedding_no_egress` orgs are
> **never** sent to Voyage regardless — the gate runs before egress (`Embeddable#embeddable?`).
> The local provider (`EMBEDDING_PROVIDER=local`) needs none of the above (no egress).

---

## Cost (post-opt-out — free tier is voided)

`voyage-code-3` ≈ **$0.18 / 1M tokens** sync, **~$0.12 / 1M** batched. Steps:

- Run the preflight (below) to get the **real eligible-token count → real $ figure**.
- Set a **hard** `BACKFILL_DOLLAR_CEILING` (it defaults to advisory-only / `0`).
- **Build the Voyage Batch-API submission path before a large backfill** — it's
  currently deferred (the backfill uses the sync path), and the ~33% discount is now
  real money, not free-tier savings.

---

## Config surface (from the merged code)

| Env var | Default | Meaning |
|---|---|---|
| `EMBEDDING_PROVIDER` | `null` | `null`→lexical (fail-closed); `voyage`; `local` |
| `EMBEDDING_MODEL_ID` | `voyage-code-3` | pinned model id; a change ⇒ full re-embed |
| `VOYAGE_API_KEY` | — | credentials `voyage.api_key` (preferred) or env |
| `EMBEDDING_LOCAL_URL` | — | TEI/BGE-M3 endpoint for `local` (self-host) |
| `RETRIEVAL_BACKEND` | `do_kb` | read path: `do_kb` or `pgvector` |
| `RETRIEVAL_DUAL_WRITE` | off | when on, index path writes both stores |
| `RETRIEVAL_SHADOW_READ_SAMPLE` | `0` | fraction of reads to shadow-compare |
| `RETRIEVAL_SHADOW_OVERLAP_K` | `5` | k for overlap@k |
| `BACKFILL_TOKEN_CEILING` | `200_000_000` | preflight aborts above this |
| `BACKFILL_DOLLAR_CEILING` | `0` (advisory) | **set a real value post-opt-out** |
| `BACKFILL_PRICE_PER_MILLION` | `0.18` | preflight pricing |
| `BACKFILL_THROTTLE_SECONDS` | `0` | politeness delay between batches |

Rake tasks: `embedding:backfill_preflight` · `embedding:backfill` · `embedding:backfill_async`.

---

## Technical rollout (staging first, then prod)

0. **Store the key** in credentials (`voyage.api_key`) or `VOYAGE_API_KEY` env (DO App
   Platform: encrypted app env var). Deploying main already created `artifact_chunks` +
   the `vector` extension (prod pgvector 0.8.1, ≥0.8 asserted at boot/CI).
1. **Preflight:** `bin/rails embedding:backfill_preflight` → record the real token/$ estimate; set `BACKFILL_DOLLAR_CEILING`.
2. **Enable embedding + dual-write** (read still DO KB): `EMBEDDING_PROVIDER=voyage`, `RETRIEVAL_DUAL_WRITE=1`.
3. **Backfill:** `bin/rails embedding:backfill` (add the Batch-API path first if the count is large).
4. **Shadow soak (~2 weeks, D2):** `RETRIEVAL_SHADOW_READ_SAMPLE=0.1`; gate cutover on **overlap@5 ≥ 0.8** (D6) sustained + `retrieval_tenant_isolation` sentinel green.
5. **Cut over:** `RETRIEVAL_BACKEND=pgvector`. **Instant rollback:** unset → back to DO KB (kept warm).
6. **Retire DO KB:** disable dual-write, decommission DO KB (teardown owner signs off).

## Self-host alternative (no Voyage, no DPA)

`EMBEDDING_PROVIDER=local` + `EMBEDDING_LOCAL_URL=<TEI/BGE-M3 @1024-dim>` — same
`halfvec(1024)` table/index, zero third-party egress. Everything else identical.
