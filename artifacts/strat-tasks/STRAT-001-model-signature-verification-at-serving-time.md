# STRAT-001: Model Signature Verification at Serving Time

**Source RFE**: RFE-001
**Jira Key**: pending — see strat-jira-guide.md
**Priority**: Major

## Business Need (from RFE)

Users who sign AI models during storage (via Model Registry/AI Hub) currently have no way to enforce signature verification when those models are served using Model Serving.
The ODH/RHOAI platform should allow users to enable signature verification policies so that only models with valid signatures:
- whether stored in S3 (with model.sig), HuggingFace or other repositories supported by Model Serving
- and as OCI ModelCar images (with Cosign signatures and OpenSSF Model Signatures at the OCI Manifest level) — can be served through KServe and llm-d.

### Problem Statement

Today, RHOAI supports signing AI models at storage time through integration of AI Hub/Model Registry Red Hat Trusted Artifact Signer via [RHAISTRAT-513](https://issues.redhat.com/browse/RHAISTRAT-513) (upstream: Sigstore Cosign and OpenSSF Model Signing).

When storing models, users can produce:

- A `model.sig` (OpenSSF Model Signature) alongside models in S3, or in HuggingFace, etc.
- A Cosign-signed container image with embedded `model.sig` for ModelCar (OCI) format

However, when these signed models are served via KServe or llm-d, the signatures are completely ignored unless some platform controller is wired in.

If there is no verification step, it implies the signing workflow is not leveraged for runtime integrity guarantee.

A user who carefully signs their model has no assurance that the model actually being served matches what they signed. An admin has no guidance on how to setup the Red Hat Trusted Artifact Signer with OpenShift AI so to ensure Model Signatures are verified for the deployed inference workload.

### Business Justification

- **Strategic investment**: Model supply chain security is a core part of the RHOAI trust and security strategy. The "sign on store" capability is already delivered with [RHAISTRAT-513](https://issues.redhat.com/browse/RHAISTRAT-513); without "verify on serve", the trust chain is incomplete and the signing investment does not deliver the runtime impact and its intended value.
- **Customer demand**: Customers in regulated industries require verifiable model provenance from storage through serving to meet internal governance and compliance requirements.
- **Competitive positioning**: End-to-end model signing and verification is an emerging differentiator in the enterprise AI platform market. Upstream Sigstore ecosystem projects (ImagePolicy controller, Model Validation operator) provide building blocks that need to work seamlessly with RHOAI model serving.

### Acceptance Criteria

- [ ] Admins can configure signature verification policies for model serving
- [ ] Models sourced from S3, HuggingFace, etc with a `model.sig` are verified against the configured policy before serving
- [ ] Models sourced as OCI ModelCar images are verified (container image signature and/or embedded OpenSSF Model Signature) against the configured policy before serving
- [ ] Admin and Users receive clear feedback when a model fails signature verification
- [ ] Models that fail signature verification are not served

## Strategy

**Effort**: M
**Components**: Sigstore policy-controller, Sigstore model-validation-operator, odh-model-controller, kserve, odh-dashboard, rhods-operator
**Impacted Teams**: KServe/Model Serving, Platform (rhods-operator), Dashboard, Security/Trust (RHTAS)

### Technical Approach

This strategy leverages two existing upstream Sigstore controllers rather than building custom verification logic. The work is dual: (1) **assess** that these controllers work correctly when composed with KServe and llm-d model serving, and (2) **integrate** — provide the necessary configuration, labeling, and UX so the controllers can perform their job seamlessly within the RHOAI platform.

The two controllers handle complementary verification scopes:

#### Controller 1: Sigstore Policy Controller (OCI image signatures)

**Upstream**: [sigstore/policy-controller](https://github.com/sigstore/policy-controller)

The policy-controller is a Kubernetes admission controller that verifies container image signatures via Cosign before pods are admitted. It operates through two webhooks:

- **Validating webhook**: intercepts pod creation and checks that container images satisfy matching `ClusterImagePolicy` CRs. In `enforce` mode, pods with unverified images are rejected (HTTP 403). In `warn` mode, warnings are returned but pods are admitted.
- **Mutating webhook**: resolves image tags to digests (`nginx:latest` becomes `nginx:latest@sha256:abc...`) to prevent tag-swapping attacks post-admission.

**How it applies to KServe/llm-d**: When KServe creates pods for an InferenceService using a ModelCar OCI image, the policy-controller's validating webhook intercepts the pod admission and verifies the Cosign signature on the container image against matching `ClusterImagePolicy` resources. No modification to KServe is needed — the controller operates at the pod admission layer.

**Policy configuration** uses the existing `ClusterImagePolicy` CRD (API group `policy.sigstore.dev/v1beta1`):
- `images[].glob` patterns to match model image references (e.g., `registry.example.com/models/**`)
- Three verification modes: key-based (public key/KMS), keyless (Fulcio certificate + OIDC identity), or static (allow/deny lists)
- Optional attestation verification with Rego/CUE policy evaluation
- Namespace opt-in via label `policy.sigstore.dev/include: "true"`
- `NoMatchPolicy` controls behavior for images matching no policies

**Failure feedback**: Admission rejection with descriptive error messages (e.g., `"no matching signatures"`, `"failed policy: <name>"`). Pod is not created; InferenceService enters a failed state that surfaces through KServe status conditions.

#### Controller 2: Sigstore Model Validation Operator (model.sig on volumes)

**Upstream**: [sigstore/model-validation-operator](https://github.com/sigstore/model-validation-operator)

The model-validation-operator validates cryptographic signatures of AI models mounted as volumes in pods, using the OpenSSF Model Signing (`model.sig`) format. It operates through:

- **Mutating webhook**: intercepts pod creation, looks for the label `validation.ml.sigstore.dev/ml: "<ModelValidation-CR-name>"`, and **injects a `validation-agent` init container** (prepended to the pod's init container list). This init container inherits volume mounts from existing containers, runs signature verification, and exits non-zero on failure — blocking the serving container from starting.
- **Optional continuous validation**: can inject a sidecar (Kubernetes 1.28+ native sidecar) that re-validates at configurable intervals, exposing `/ready` only after successful verification.

**How it applies to KServe/llm-d**: The kserve-storage-initializer downloads model artifacts (from S3, HuggingFace, etc.) to a shared volume, including the co-located `model.sig`. The model-validation-operator's injected init container runs *after* the storage initializer and *before* the model server, verifying the `model.sig` against configured trust policies. If verification fails, the init container exits 1, and the pod enters `Init:Error` state.

**Policy configuration** uses the `ModelValidation` CRD (API group `ml.sigstore.dev/v1alpha1`, namespace-scoped):
- `spec.config`: one of `sigstoreConfig` (keyless: certificate identity + OIDC issuer), `pkiConfig` (CA chain), `publicKeyConfig` (direct key), or `clientTrustConfig` (private Rekor/Fulcio via RHTAS)
- `spec.model.path` / `spec.model.signaturePath`: paths to the model and `model.sig` on the volume
- `spec.continuousValidation`: optional periodic re-verification
- Pod association via label `validation.ml.sigstore.dev/ml: "<CR-name>"`
- Per-pod annotation overrides for ignore paths, symlinks, unsigned files

**Failure feedback**: Init container exits with code 1; pod enters `Init:Error` or `Init:CrashLoopBackOff`. Error details are in the `validation-agent` container logs. The operator tracks injected/uninjected/orphaned pod status on the `ModelValidation` CR and exposes Prometheus metrics.

#### Integration Work

The core integration effort is ensuring these two controllers compose correctly with KServe and llm-d model serving:

**Phase 1 — Assessment**: Deploy both controllers alongside KServe and llm-d in a test environment. Validate:
- Policy-controller correctly intercepts KServe-created pods for ModelCar images and verifies Cosign signatures
- Policy-controller's mutating webhook (tag-to-digest resolution) does not conflict with KServe's own image resolution
- Model-validation-operator's injected init container runs in the correct order relative to the kserve-storage-initializer (must run *after* model download completes)
- Init container ordering works for both InferenceService (KServe) and LLMInferenceService (llm-d) pod specs
- Model-validation-operator can access the model volume mounts used by the storage initializer
- Failure states propagate correctly to InferenceService/LLMInferenceService status conditions
- Both controllers work with RHTAS (private Fulcio/Rekor) for disconnected/air-gapped environments

**Phase 2 — Integration**: Based on assessment findings, provide the necessary glue:
- **Namespace labeling**: Ensure model serving namespaces have `policy.sigstore.dev/include: "true"` for the policy-controller. This may be automated via odh-model-controller or rhods-operator when signature verification is enabled.
- **Pod labeling for model-validation-operator**: KServe/llm-d pods need the label `validation.ml.sigstore.dev/ml: "<CR-name>"` to trigger init container injection. This requires either (a) KServe pod template annotation support, (b) a mutation in odh-model-controller that adds the label when a `ModelValidation` CR exists in the namespace, or (c) a separate labeling webhook.
- **Init container ordering**: Verify the model-validation-operator's injected init container (prepended to the list) runs after the kserve-storage-initializer. If ordering is wrong, contribute an upstream fix to the operator to support configurable injection position, or use odh-model-controller to reorder.
- **Volume mount alignment**: The model-validation-operator inherits volume mounts from existing containers. Confirm this correctly picks up the storage initializer's model volume (`/mnt/models` or equivalent).
- **Status surfacing**: When either controller blocks a pod, KServe reports a generic pod failure. Enhance odh-model-controller or odh-dashboard to detect signature-verification-specific failures (admission rejection messages, `Init:Error` with validation-agent container) and surface actionable error messages to users.
- **Dashboard UX**: Add guidance in odh-dashboard for admins to configure `ClusterImagePolicy` and `ModelValidation` CRs for model serving namespaces. Display signature verification status on InferenceService detail pages.
- **Operator lifecycle**: Package both upstream controllers for deployment via rhods-operator or OLM, integrated with RHTAS configuration.

### Affected Components

| Component | Change | Owner Team |
|-----------|--------|------------|
| Sigstore policy-controller | Deploy and configure for RHOAI model serving namespaces. Validate compatibility with KServe pod creation patterns. No code changes expected — configuration and lifecycle management only. | Security/Trust (RHTAS) |
| Sigstore model-validation-operator | Deploy and configure. Validate init container injection ordering with kserve-storage-initializer. May need upstream contributions for init container positioning if ordering conflicts are found. | Security/Trust (RHTAS) |
| odh-model-controller | Add logic to apply namespace labels (`policy.sigstore.dev/include`) and pod labels (`validation.ml.sigstore.dev/ml`) when signature verification policies are configured. Detect and surface signature verification failures in InferenceService status conditions. | Model Serving Platform |
| kserve | No code changes expected. Assessment to confirm pod spec compatibility with both controllers' webhooks (init container ordering, image tag resolution, volume mounts). | KServe/Model Serving |
| odh-dashboard | Add admin UX for configuring signature verification policies (`ClusterImagePolicy`, `ModelValidation` CRs). Display verification status and actionable error messages on InferenceService pages. | Dashboard |
| rhods-operator | Package and manage lifecycle of both upstream controllers. Integrate with RHTAS configuration (Fulcio/Rekor endpoints, trust anchors). | Platform |

### Dependencies

| Dependency | Type | Status | Impact if Missing |
|------------|------|--------|-------------------|
| RHAISTRAT-513 (Model Signing at Storage Time) | Internal | Delivered | No signatures to verify; feature has no input |
| [Sigstore policy-controller](https://github.com/sigstore/policy-controller) | External | Stable upstream (v0.10.x, K8s 1.27-1.29) | OCI image signature verification at admission time depends on it |
| [Sigstore model-validation-operator](https://github.com/sigstore/model-validation-operator) | External | Early stage / proof of concept (v1alpha1, no formal releases) | Volume-mounted model.sig verification depends on it |
| Red Hat Trusted Artifact Signer (RHTAS) | External | GA product | Provides private Fulcio/Rekor for enterprise/disconnected deployments; both controllers need RHTAS integration for production use |
| OpenSSF Model Signing spec (model.sig format) | External | Exists (github.com/sigstore/model-transparency) | model-validation-operator depends on this for signature format |
| kserve-storage-initializer ModelCar support | Internal | Exists (enabled in RHOAI 3.4) | ModelCar model loading is the path where both controllers apply |

### Non-Functional Requirements

- **Performance**: Policy-controller adds latency to pod admission (signature verification before pod creation). Model-validation-operator adds an init container phase after model download. Combined overhead should be < 15 seconds for typical models. Neither controller affects inference latency once the model is loaded and serving.
- **Scalability**: Verification is per-pod-creation, not per-request. No steady-state performance impact on inference latency or throughput.
- **Security**: Trust anchors (public keys, Sigstore root certificates) must be stored in Kubernetes Secrets with appropriate RBAC. Both controllers must be FIPS-compatible for RHOAI certification. Failure modes must default to denying unverified models when in enforce mode.
- **Availability**: Controller failures must not block all pod creation. Both controllers support `warn`/`audit` modes for gradual rollout. If RHTAS infrastructure (Fulcio/Rekor) is unreachable, offline verification with pre-distributed public keys must be supported.
- **Backwards Compatibility**: Existing InferenceServices without signature policies must continue to work unchanged. Verification is opt-in (namespace labeling + policy CRs). No changes to the InferenceService or LLMInferenceService CRD schemas.

### Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Model-validation-operator immaturity | H | Upstream is self-described "proof of concept" with v1alpha1 API, no formal releases, and a roadmap extending into 2026. API may change. | Engage upstream early; contribute stabilization work; pin to a known-good commit; maintain downstream patches if needed |
| Init container ordering conflict | M | Model-validation-operator prepends its init container to the pod spec. If it runs before the kserve-storage-initializer, the model files won't exist yet and verification will fail. | Assess ordering during Phase 1; contribute upstream fix for configurable injection position if needed |
| Policy-controller tag-to-digest mutation conflict | L | Policy-controller's mutating webhook resolves image tags to digests. If KServe also does this or expects mutable tags, conflicts may arise. | Test during Phase 1; document webhook ordering requirements |
| FIPS compatibility of Sigstore crypto | M | Both controllers use Sigstore Go libraries which may use non-FIPS-compliant crypto primitives | Audit crypto usage in both controllers; use FIPS-compliant builds; contribute upstream fixes |
| Policy misconfiguration blocking all deployments | H | Admin sets overly restrictive ClusterImagePolicy that prevents all model serving pods | Both controllers support warn/audit modes; dashboard UI should validate and preview policy impact; document escape hatch |
| Disconnected environment support | M | Both controllers may require online access to Rekor transparency log and Fulcio CA by default | Configure both controllers with RHTAS private instances; support offline verification with pre-distributed trust material |

### Open Questions

- **Init container ordering**: Does the model-validation-operator's prepend-to-init-containers approach work correctly with KServe's storage initializer? If not, what upstream changes are needed?
- **Pod labeling mechanism**: What is the best way to apply `validation.ml.sigstore.dev/ml` labels to KServe/llm-d pods? Options: (a) KServe pod template annotations, (b) odh-model-controller mutation, (c) separate webhook. Trade-offs between coupling, configurability, and maintenance.
- **Model-validation-operator maturity**: Given its proof-of-concept status, should RHOAI invest in stabilizing it upstream, fork it, or wait for it to mature? The upstream roadmap mentions possible integration into policy-controller — should we align with that direction instead?
- **Scope of initial delivery**: Phase 1 (assessment) for both controllers + Phase 2 (integration) for the more mature policy-controller first, deferring model-validation-operator integration until it stabilizes?
- **RHTAS integration depth**: Should both controllers be configured to use RHTAS out-of-the-box when RHTAS is installed on the cluster, or should configuration be manual?
- **Continuous validation**: The model-validation-operator supports sidecar-based continuous re-validation. Is this needed for RHOAI, or is one-shot init-container validation sufficient for the initial release?

### Scope Boundary

**Delivers**: Assessment of Sigstore policy-controller and model-validation-operator composability with KServe and llm-d. Integration work (namespace/pod labeling, status surfacing, dashboard UX, operator lifecycle) to enable both controllers for RHOAI model serving. Admin-facing configuration guidance and tooling.

**Does NOT deliver**: Model signing capabilities (already covered by RHAISTRAT-513). Custom verification logic built into kserve-storage-initializer (leverages upstream controllers instead). New CRDs — uses existing `ClusterImagePolicy` and `ModelValidation` CRDs from the upstream controllers.

**Assumptions**: RHAISTRAT-513 produces signatures compatible with what both controllers verify (Cosign for OCI images, OpenSSF `model.sig` for model content). RHTAS is available as the Sigstore infrastructure for RHOAI customers. The Sigstore policy-controller is stable enough for production use. The model-validation-operator will require upstream stabilization work as part of this strategy.
