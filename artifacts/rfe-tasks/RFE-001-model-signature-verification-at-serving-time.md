# RFE-001: Model Signature Verification at Serving Time

**Priority**: Major
**Size**: M

## Summary

Users who sign AI models during storage (via Model Registry/AI Hub) currently have no way to enforce signature verification when those models are served using Model Serving.
The ODH/RHOAI platform should allow users to enable signature verification policies so that only models with valid signatures:
- whether stored in S3 (with model.sig), HuggingFace or other repositories supported by Model Serving
- and as OCI ModelCar images (with Cosign signatures and OpenSSF Model Signatures at the OCI Manifest level) — can be served through KServe and llm-d.

## Problem Statement

Today, RHOAI supports signing AI models at storage time through integration of AI Hub/Model Registry Red Hat Trusted Artifact Signer via [RHAISTRAT-513](https://issues.redhat.com/browse/RHAISTRAT-513) (upstream: Sigstore Cosign and OpenSSF Model Signing).

When storing models, users can produce:

- A `model.sig` (OpenSSF Model Signature) alongside models in S3, or in HuggingFace, etc.
- A Cosign-signed container image with embedded `model.sig` for ModelCar (OCI) format

However, when these signed models are served via KServe or llm-d, the signatures are completely ignored unless some platform controller is wired in.

If there is no verification step, it implies the signing workflow is not leveraged for runtime integrity guarantee.

A user who carefully signs their model has no assurance that the model actually being served matches what they signed. An admin has no guidance on how to setup the Red Hat Trusted Artifact Signer with OpenShift AI so to ensure Model Signatures are verified for the deployed inference workload.

## Affected Customers

<!-- Adam to fill-in segment/customer names -->
- [Placeholder: specific customer accounts and segments to be confirmed]

## Business Justification

- **Strategic investment**: Model supply chain security is a core part of the RHOAI trust and security strategy. The "sign on store" capability is already delivered with [RHAISTRAT-513](https://issues.redhat.com/browse/RHAISTRAT-513); without "verify on serve", the trust chain is incomplete and the signing investment does not deliver the runtime impact and its intended value.
- **Customer demand**: Customers in regulated industries require verifiable model provenance from storage through serving to meet internal governance and compliance requirements.
- **Competitive positioning**: End-to-end model signing and verification is an emerging differentiator in the enterprise AI platform market. Upstream Sigstore ecosystem projects (ImagePolicy controller, Model Validation operator) provide building blocks that need to work seamlessly with RHOAI model serving.

## Acceptance Criteria

- [ ] Admins can configure signature verification policies for model serving
- [ ] Models sourced from S3, HuggingFace, etc with a `model.sig` are verified against the configured policy before serving
- [ ] Models sourced as OCI ModelCar images are verified (container image signature and/or embedded OpenSSF Model Signature) against the configured policy before serving
- [ ] Admin and Users receive clear feedback when a model fails signature verification
- [ ] Models that fail signature verification are not served

## Success Criteria

Admins and Users can opt in to signature verification policies and receive clear pass/fail feedback before models are served through KServe or llm-d.
The end-to-end story arc — sign on store, verify on serve — provides a complete trust chain for AI model provenance.
