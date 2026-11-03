# supply-chain remediation plan

All work below is proposed, not executed. Observed paths refer to the pinned SHA `448f6c41a6bfe59caddc6d3842b38a2506de1dee`; proposed paths are labeled. No tag creation, attestation, reusable-workflow execution, BCR publication, rollback, or restore was observed in this review.

## Phase gates

| Phase | Entry | Exit |
| --- | --- | --- |
| 1. Fork governance and confirmed defects | Review accepted; findings triaged | T-05 defines fork autonomy/upstream sync/release authority, and T-01/T-02 have implementable defect plans |
| 2. Risk-based automation | T-05 complete; package inventory and fixture format available | T-03/T-04 make CI expectations and SBOM boundary tests explicit |
| 3. Release and recovery | T-01, T-03, and T-05 complete; owner approves future rehearsal scope | T-06 defines a phased release/recovery runbook grounded in the privileged tagging/attestation/BCR path |
| 4. Independent acceptance | Candidate SHA frozen after T-01–T-06 | T-07 records independent verification against the same candidate without treating unobserved release/restore as observed |

## Phase 1 — Fork governance and confirmed defects

### T-05 — Define fork divergence, upstream sync, and release authority (preventive)
Prerequisites: review findings accepted for planning. Observed paths: `evidence/supply-chain/snapshot.json`, `README.md`, `.github/workflows/release_metadata.yml`; proposed paths: `docs/governance/fork-policy.md`, `docs/governance/release-authority.md`.
1. Record the authoritative upstream relationship for this fork, using the observed `isFork: true` metadata and the README statement that the repo is the successor to `rules_license`, and state whether the intended posture is autonomous fork, selective sync, or eventual reconvergence.
2. Catalogue the known divergence areas that affect behavior or trust boundaries, including module/version cadence, release workflow changes, reusable-workflow pins, and BCR publication target, and classify each as `must track upstream`, `fork-owned`, or `case-by-case`.
3. Define the sync decision path for future upstream changes: what gets compared, who reviews it, what evidence is retained, and which classes of upstream delta automatically pause release until reconciled.
4. Assign fork-side release authority for tag approval, permission changes (`contents`, `id-token`, `attestations`), reusable-workflow reference bumps, and BCR publication approval, while preserving any intended fork autonomy rather than assuming upstream owns releases.
Deliverable: owner-approved fork policy and release-authority record that distinguishes intentional divergence from neglected drift. Verification: a non-implementing reviewer can trace one sample upstream change and one sample fork-only release change through the written policy and confirm the required approver/evidence chain is explicit.
- [ ] Fork autonomy versus upstream-tracking intent is written down.
- [ ] Divergence classes and sync triggers are explicit.
- [ ] Release authority for tags, attestation scope, reusable workflows, and BCR publish is named.

### T-01 — Harden release tool trust boundary (S-01)
Prerequisites: T-05; owner picks the allowed source for `buildozer` provenance. Observed path: `.github/workflows/release_metadata.yml`.
1. Replace the network download with a pinned, verified artifact source or a vendored binary flow.
2. Split privilege so the step that mutates version metadata is not also the step that can publish with broad write permissions.
3. Add a negative check that a tampered artifact is rejected before execution.
Deliverable: release workflow that cannot execute an unverified binary. Verification: a checksum mismatch or altered binary must stop before any tag/commit/publish step.
- [ ] Trusted source and digest policy recorded.
- [ ] Verification happens before executable use.
- [ ] Release permissions are reduced or separated.

### T-02 — Make generator precondition failures loud (S-02)
Prerequisites: none beyond the existing generator inputs. Observed path: `maintainers/tools/purl_tables/main.py`; proposed sibling review paths: `maintainers/tools/purl_spec_tests/main.py`, `maintainers/tools/update_spdx_list/main.py`.
1. Change the wrong-locale branch to return a nonzero failure code after printing the warning.
2. Add deterministic tests that confirm the bad-locale path fails and leaves outputs untouched.
3. Keep a golden-path test that proves the generated content is stable under the required locale.
Deliverable: generator scripts and tests with explicit success/failure semantics. Verification: a deliberately reverted zero exit must make the negative test fail.
- [ ] Wrong locale exits nonzero.
- [ ] Output is unchanged on failure.
- [ ] Deterministic golden output is covered.

## Phase 2 — Risk-based automation

### T-03 — Encode the no-tests policy explicitly (S-03; preventive)
Prerequisites: package inventory reviewed and empty-module exceptions approved. Observed path: `.github/workflows/ci.sh`.
1. Document which modules may legitimately have zero tests.
2. Make the wrapper fail on exit code 4 unless the selected package is on the approved empty list.
3. Add a negative test for an unexpected empty package selection.
Deliverable: CI policy that distinguishes intentional emptiness from accidental gaps. Verification: a broken package selection must fail, while an approved empty package still passes.
- [ ] Approved empty-module list exists.
- [ ] Unexpected no-test runs fail.
- [ ] Intentional no-test modules are still allowed.

### T-04 — Add SBOM CLI fixture coverage (preventive)
Prerequisites: fixture format for `graph-only JSON` confirmed. Observed path: `lib/supplychain-go/cmd/sbom/main.go`.
1. Add malformed-input, missing-flag, and write-failure cases around the CLI boundary.
2. Add a test for `allow_missing_root_metadata` so the permissive branch is explicit.
3. Keep classification logic side-effect free except for the single output file.
Deliverable: CLI-level fixtures that pin the observable SBOM contract. Verification: invalid JSON must fail, valid JSON must write output, and removing validation must break a test.
- [ ] Missing flags fail.
- [ ] Malformed JSON fails.
- [ ] Valid output shape is asserted.

## Phase 3 — Release and recovery

### T-06 — Define phased release and recovery evidence at the privileged workflow boundary (preventive)
Prerequisites: T-01, T-03, and T-05. Observed path: `.github/workflows/release_metadata.yml`; proposed path: `docs/operations/release-recovery.md`.
1. Break the current release path into explicit boundaries — `workflow_dispatch`, privileged tag creation, reusable GitHub-release workflow, and BCR publication — and record the expected inputs, outputs, and stop points for each boundary.
2. Define the evidence that a future authorized rehearsal or release must retain at each step: candidate SHA, created tag/ref, commit or module-version diff, attestation/provenance identifiers, reusable-workflow run reference, and BCR publication record.
3. Define forward-only recovery rules for each failure point: stop before tag push when preflight fails; after tag push or GitHub release, issue a corrective version rather than retagging; after BCR publication, preserve the published artifact history and recover with a new reviewed release path rather than silent overwrite.
4. Document who may approve future use of the privileged boundary, including permission-scope changes and BCR publish-token use, without conflating documentation review with live credential use.
Deliverable: phased release/recovery runbook tied to the actual privileged workflow edges. Verification: a reviewer can map every irreversible step in the runbook to the observed workflow boundaries, and any future rehearsal only closes this task when the recorded tag, attestation, reusable-workflow, and BCR evidence all point to the same candidate SHA.
- [ ] Tagging, attestation, reusable-workflow, and BCR boundaries are separated.
- [ ] Forward-only recovery policy forbids silent overwrite or retagging.
- [ ] Future rehearsal evidence requirements are explicit and SHA-bound.

## Phase 4 — Independent acceptance

### T-07 — Perform independent acceptance on the candidate and its evidence package (preventive)
Prerequisites: T-02, T-04, T-05, and T-06; candidate SHA frozen. Observed paths: CI/workflow definitions plus the changed implementation/tests from T-01–T-04; proposed path: `docs/operations/independent-acceptance.md`.
1. Assemble a candidate evidence package that ties the exact SHA to the fork-policy decision, release-boundary docs, negative-control tests, and any future authorized rehearsal outputs.
2. Require a non-implementing reviewer to re-check the S-01 trust-boundary control, the S-02 generator failure semantics, the S-03 no-tests policy, and the fork/upstream authority rules against that same candidate SHA.
3. Require the reviewer to record pass/fail, exceptions, and unrun privileged checks separately, and to refuse sign-off if release, publication, recovery, or restore is claimed without observed evidence.
Deliverable: independent acceptance record tied to one candidate SHA and one evidence bundle. Verification: another reviewer can trace each acceptance statement back to the cited candidate evidence and confirm that unobserved release/recovery steps remain labeled unobserved.
- [ ] Every finding is resolved or explicitly excepted with owner approval.
- [ ] Acceptance evidence is tied to one candidate SHA and reproducible.
- [ ] Unobserved release/recovery steps are not phrased as completed.

## Task dependency map

| Task | Depends on | Blocks |
| --- | --- | --- |
| T-05 | review accepted for planning | T-01, T-06, T-07 |
| T-01 | T-05 + provenance decision | T-06, release confidence |
| T-02 | none | generator correctness and T-07 |
| T-03 | package inventory / owner allowlist | T-06, CI gate trust |
| T-04 | fixture format confirmation | SBOM contract confidence and T-07 |
| T-06 | T-01 + T-03 + T-05 | T-07, release/recovery confidence |
| T-07 | T-02 + T-04 + T-05 + T-06 | production acceptance |

## Production acceptance checklist

- [ ] Fork autonomy, upstream sync triggers, and release authority are explicitly owned.
- [ ] Release metadata cannot execute a network-fetched, unchecked binary.
- [ ] Wrong-locale generator runs fail loudly and do not emit stale output.
- [ ] CI policy distinguishes intentional empty packages from accidental test gaps.
- [ ] SBOM CLI has negative fixtures and a documented permissive branch.
- [ ] Future release/recovery evidence is defined across tag, attestation, reusable-workflow, and BCR boundaries.
- [ ] No document claims release, publication, rollback, or restore was observed until that evidence exists.
- [ ] Independent acceptance is performed by a non-implementing reviewer on the same candidate SHA.

## Changes requiring owner approval

Any change to intended fork autonomy, upstream sync strategy, release authority, release signing/attestation scope, reusable-workflow references, BCR publish-token use, Bazel support window, or the list of intentionally empty modules.

## Rollback and release policy

Do not retag a released version. Preserve immutable tags, release artifacts, attestations, and BCR publication history. If provenance or release integrity is uncertain, stop at the nearest workflow boundary and keep the event classified as unverified until reviewed evidence exists. After a tag or BCR publish, recover forward with a new reviewed version rather than overwriting prior history. Keep planned runbooks, future rehearsals, and any later observed releases/restores as separate evidence classes.
