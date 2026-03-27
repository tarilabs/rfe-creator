# RFE Review Report

**Date**: 2026-03-27
**RFEs reviewed**: 1
**Rubric validation**: PASS (9/10)
**Technical feasibility**: Feasible

## Summary

RFE-001 is well-written and ready for submission. The only rubric gap is the WHY criterion (1/2) due to the placeholder in Affected Customers — this requires the author to fill in named accounts or specific evidence. No auto-revision was applied because the gap requires information only the author has.

The RFE is technically feasible and aligns well with RHOAI's platform direction. Several strategy-level considerations were identified for `/strat.refine`.

## Per-RFE Results

### RFE-001: Model Signature Verification at Serving Time

**Rubric score**: 9/10 PASS

| Criterion | Score | Notes |
|-----------|-------|-------|
| WHAT | 2/2 | Clear, specific customer need with concrete, testable acceptance criteria covering both S3-stored models (model.sig) and OCI ModelCar images (Cosign/OpenSSF signatures). |
| WHY | 1/2 | Three categories of business justification provided: strategic investment (completing RHAISTRAT-513 trust chain), customer demand from regulated industries, and competitive positioning. The strategic investment argument has a reasonable causal chain. However, the "Affected Customers" section is a placeholder with no named accounts, no revenue/deal impact, and no analyst data. Justification stays at generic segments ("regulated industries"). |
| Open to HOW | 2/2 | Describes the need without prescribing architecture. KServe, llm-d, S3, ModelCar/OCI, Cosign, OpenSSF Model Signing are all established RHOAI platform vocabulary. Mentioning "platform controller" in the problem statement describes the current gap, not a prescribed solution. |
| Not a task | 2/2 | Clear business need — runtime integrity verification for AI models. Acceptance criteria describe outcomes (admins configure, users get feedback, unverified models blocked), not implementation tasks. |
| Right-sized | 2/2 | Maps well to a single strategy feature. Two model formats (S3 + OCI) are two facets of the same verification capability at a single enforcement point. |

**Actionable suggestion for WHY (scored 1/2):**
- Fill in the "Affected Customers" placeholder with named customer accounts or specific deal impacts. Example: "Customer X in financial services requires model provenance verification for regulatory compliance; $Y deal contingent on this capability."
- If specific names cannot be shared, cite concrete evidence: number of customer support tickets, specific RFP requirements encountered, or analyst report findings with demonstrated customer consequences.
- The strategic investment argument ("completing the trust chain from RHAISTRAT-513") is solid but alone does not reach the bar for a 2 — pairing it with named accounts or revenue impact would bring it to full marks.

**Technical feasibility**: Feasible

The RFE is technically achievable with the current RHOAI architecture. Key findings:

- **OCI/ModelCar path**: Well-understood. Cosign signature verification can be enforced via an ImagePolicy controller (e.g., Sigstore policy-controller). KServe's existing webhook infrastructure provides clear integration points. No major architectural change needed.
- **S3/HuggingFace path**: Harder. No off-the-shelf solution — requires custom logic or the upstream OpenSSF Model Signing verifier. Could be implemented as an additional init container injected via the existing `/mutate-pods` webhook pattern.
- **LLMInferenceService path**: Separate model loading path from classic InferenceService (uses vLLM's native loading, not KServe storage initializer). Requires distinct integration work.
- **Disconnected environments**: Sigstore verification typically needs Rekor/Fulcio access. RHTAS disconnected mode addresses this but integration details matter.

| Risk | Severity |
|------|----------|
| S3/HuggingFace verification is not off-the-shelf | Medium |
| LLMInferenceService has divergent model loading path | Medium |
| Disconnected/air-gapped environment support | Medium |
| Upstream maturity of OpenSSF Model Signing | Medium |
| Performance impact on cold start | Low-Medium |

**Strategy considerations** (flag for `/strat.refine`):
- Consider phasing: OCI/ModelCar verification first (simpler, more mature), S3/HuggingFace second
- Determine verification integration point: admission-time (block pod creation), init-container (fail pod startup), or controller-level (block InferenceService readiness)
- Confirm RHAISTRAT-513 is GA and signature format is stable before building verification side
- Scope whether LLMInferenceService support is in v1 or deferred
- Address disconnected environment verification (RHTAS disconnected mode integration)
- Define user feedback surface: InferenceService status conditions, events, dashboard UI, or all
- Impacted teams: KServe, odh-model-controller, Model Registry, RHTAS, llm-d

**Recommendation**: **Submit** — fill in the Affected Customers placeholder to strengthen the WHY score from 1 to 2.
