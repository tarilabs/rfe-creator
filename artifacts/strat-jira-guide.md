# Manual RHAISTRAT Creation Guide

For each RFE below, clone it in Jira to the RHAISTRAT project:

1. Open the RHAIRFE in Jira
2. Use Clone (... menu > Clone) and set the target project to RHAISTRAT
3. The issue type will be Feature
4. Record the new RHAISTRAT key below

| Source RFE | RHAISTRAT Key | Title |
|------------|---------------|-------|
| RFE-001 / RHAIRFE-NNNN | <fill in after cloning> | Model Signature Verification at Serving Time |

After cloning, update `artifacts/strat-tasks/STRAT-001-model-signature-verification-at-serving-time.md` with the RHAISTRAT key, then run `/strat.refine` to add the technical strategy.
