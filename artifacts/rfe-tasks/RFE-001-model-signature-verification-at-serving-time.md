# RFE-001: Model Signature Verification at Serving Time

**Priority**: Major
**Size**: M

## Summary

Users who sign AI models during storage (via Model Registry) currently have no way to enforce signature verification when those models are served. The platform should allow users to enable signature verification policies so that only models with valid signatures — whether stored in S3 (with model.sig) or as OCI ModelCar images (with Cosign signatures and OpenSSF Model Signatures) — can be served through KServe and llm-d.

## Problem Statement

Today, RHOAI supports signing AI models at storage time through Red Hat Trusted Artifact Signer (Sigstore Cosign and OpenSSF Model Signing). When storing models, users can produce:

- A `model.sig` (OpenSSF Model Signature) alongside models in S3
- A Cosign-signed container image with embedded `model.sig` for ModelCar (OCI) format

However, when these signed models are served via KServe or llm-d, the signatures are completely ignored. There is no verification step, meaning the signing workflow provides no runtime integrity guarantee. A user who carefully signs their model has no assurance that the model actually being served matches what they signed.

## Affected Customers

<!-- Adam to fill-in segment/customer names -->
- [Placeholder: specific customer accounts and segments to be confirmed]

## Business Justification

- **Strategic investment**: Model supply chain security is a core part of the RHOAI trust and security strategy. The "sign on store" capability is already delivered; without "verify on serve," the trust chain is incomplete and the signing investment does not deliver its intended value.
- **Customer demand**: Customers in regulated industries require verifiable model provenance from storage through serving to meet internal governance and compliance requirements.
- **Competitive positioning**: End-to-end model signing and verification is an emerging differentiator in the enterprise AI platform market. Upstream Sigstore ecosystem projects (ImagePolicy controller, Model Validation operator) provide building blocks that need to work seamlessly with RHOAI model serving.

## Acceptance Criteria

- [ ] Users can configure signature verification policies for model serving
- [ ] Models sourced from S3 with a `model.sig` are verified against the configured policy before serving
- [ ] Models sourced as OCI ModelCar images are verified (container image signature and embedded OpenSSF Model Signature) against the configured policy before serving
- [ ] Users receive clear feedback when a model fails signature verification
- [ ] Models that fail signature verification are not served

## Success Criteria

Users can opt in to signature verification policies and receive clear pass/fail feedback before models are served through KServe or llm-d. The end-to-end workflow — sign on store, verify on serve — provides a complete trust chain for AI model provenance.
