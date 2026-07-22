# Deep Build checkpoint plan — silent-bug detection (invariant signals)

Implementation map: `.deep-build/implementation-map.md` (approved spec 2026-07-22).

## Checkpoint 1: Structured redaction v2 (Pillar B, crumbtrail-core)
Goal: replace whole-body `[REDACTED]` with structure-preserving, deny-biased per-value redaction for JSON bodies (≤16 KB), tagged `crumbtrail.browser-redaction.v2`, config `redaction: {mode: "structured"|"full", denyFields}` (default structured), shape metadata `{len, charset, hash8}` on redacted values. Reuse `SENSITIVE_NAME_RE`/compact name sets in `packages/core/src/redaction.ts`; add value patterns (email, Luhn, JWT, entropy ≥24). Wire the 4 `redactNetworkTextBody` call sites in `collectors/network.ts`; parse failure/oversize/non-JSON falls back to v1; never throw.
Files: core/src/redaction.ts (+ optional redaction-structured.ts), collectors/network.ts, types.ts, bug-logger.ts, index.ts, tests.
Done when: coupon-style enum values survive; password/card/JWT redacted + shape; v2 tag emitted; `mode:"full"` = v1 output; fallback never throws; existing tests pass (with intentional expectation updates only).
Verify: `pnpm --filter crumbtrail-core typecheck && pnpm --filter crumbtrail-core test && pnpm lint`

## Checkpoint 2: Pillar A detectors (crumbtrail-node)
Goal: `db_delta_mismatch` (score 72, sev high, conf high, uncapped, exact-pairing only, cart-line aggregation summed qty vs summed delta per pk, silent on ambiguity) and `ineffective_input` (score 55, sev medium, conf low, cap 3/session, dedupe by field, stem map coupon→discount|redemption|promo, search|query→results) in `buildEvidenceCandidates` (`packages/node/src/evidence-index.ts`), same draft/dedupe/causal pipeline. v1/missing bodies → silent.
Files: node/src/evidence-index.ts, optional network-body.ts helper, new tests evidence-index-db-delta.test.ts + evidence-index-ineffective-input.test.ts.
Done when: P1 fixture (qty 1, delta 2) → 72/high; matching delta silent; P2 fixture fires; redemption-present silent; cap enforced; existing tests unchanged.
Verify: `pnpm --filter crumbtrail-node typecheck && pnpm --filter crumbtrail-node test && pnpm lint`

## Checkpoint 3: ui.num collector (Pillar C capture, crumbtrail-core)
Goal: `collectors/ui-numbers.ts` emitting `{k:"ui.num", d:{region, items:[{label, value, unit}]}}` on nav commit + MutationObserver settle (500 ms debounce); labels from dt/dd, label+sibling, aria-label, preceding text; caps ≤50 tokens, 1/region/settle, change-only; labels through Checkpoint 1 classifier; registration in bug-logger.ts with degradation wrapping (degradedCollection).
Done when: P3-shaped dl.totals produces spec snapshot; caps/change-only/degradation verified; core tests pass.
Verify: `pnpm --filter crumbtrail-core typecheck && pnpm --filter crumbtrail-core test && pnpm lint`

## Checkpoint 4: Pillar C detectors (crumbtrail-node)
Goal: `ui_arithmetic_mismatch` (score 60, sev medium, conf high, uncapped; subtotal/tax/fee/shipping/discount vs total, qty×price vs line total; ε = 1 cent per component) and `ui_api_divergence` (score 55, conf medium, cap 3; UI number vs same-stem net.res field since last nav, ε = 1 cent) in evidence-index.ts; reuse Checkpoint 2 stem helper.
Done when: 199.00+16.42 vs Total 199.00 → fires; Total 215.42 silent; cap/epsilon verified; no ui.num → inert.
Verify: `pnpm --filter crumbtrail-node typecheck && pnpm --filter crumbtrail-node test && pnpm lint`

## Checkpoint 5: Playground regressions #24–#26 — SEPARATE REPO (crumbtrail-playground)
Goal: patch-activated pairs `over-decrement.patch` (#24 → expects db_delta_mismatch), `silent-coupon-accept.patch` (#25 → expects ineffective_input + visible coupon field), `display-total-drops-tax.patch` (#26 → expects ui_arithmetic_mismatch); manifest.json entries; BUGS.md entries. Commits in the playground repo, NOT this PR.
Verify: cli worktree `pnpm build && pnpm pack:local`; playground `pnpm playground:verify --all --json` green.

## Dependency graph
| CP | Prereqs | Scope | Concurrency |
| --- | --- | --- | --- |
| 1 | none | core: redaction, network, types, bug-logger | wave 1 |
| 2 | none (v1 bodies → silent) | node: evidence-index, tests | wave 1 (disjoint with 1) |
| 3 | 1 (classifier) | core: ui-numbers (new), bug-logger, types | wave 2 (disjoint with 4; conflicts with 1) |
| 4 | 2 (evidence-index, stem helper) | node: evidence-index, tests | wave 2 (disjoint with 3; conflicts with 2) |
| 5 | 1–4 | playground repo | last, cross-repo |

Order: {1 ∥ 2} → {3 ∥ 4} → 5.
Review moments: privacy audit after 1; false-positive posture after 2; P3 vertical slice after 3+4; cross-repo gate before 5.
