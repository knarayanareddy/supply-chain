# supply-chain review

## 1. Scope and verdict
Repo: `supply-chain` at pinned SHA `448f6c41a6bfe59caddc6d3842b38a2506de1dee` on `main`; stage: forked implementation / prototype, not production-ready. Runtime NOT RUN. Sampled evidence: repository snapshot metadata, `release_metadata.yml`, `ci.sh`, `maintainers/tools/purl_tables/main.py`, and `lib/supplychain-go/cmd/sbom/main.go`, plus the snapshot’s README summary and file inventory. Uninspected boundaries: `.bcr/**`, `docs/**`, `metadata/**`, `metadata-extensions/**`, `rules_license/**`, `tools/**`, `lib/supplychain-go/internal/sbom/**`, `release_prep.sh`, `release_tools.yml`, upstream diff, and all historical Actions logs.

The visible product boundary is a Bazel supply-chain metadata toolchain: metadata declaration, license/SBOM collection, and release publication into BCR. The visible security boundary is the release runner, generated-table tools, and the SBOM CLI that consumes graph JSON and writes classifications.

## 2. Architecture and product boundary
Actual entry points visible in evidence:
- `README.md` (snapshot summary) frames the repo as Bazel modules for “injecting and collecting supply-chain metadata into builds” and says it is the “successor to rules_license.”
- `.github/workflows/ci.sh` is the repo-wide Bazel test gate.
- `.github/workflows/release_metadata.yml` is the metadata release entrypoint.
- `maintainers/tools/purl_tables/main.py` is a code generator for percent-encoding tables.
- `lib/supplychain-go/cmd/sbom/main.go` is the SBOM CLI that parses graph JSON and emits classifications.

Nested apps were not present; this is a multi-module library/tool repo, not a web app. External dependencies in the inspected path include GitHub Actions, Bazel, `buildozer`, and a reusable BCR publish workflow.

## 3. Findings ledger
- **S-01 — High, confirmed static, high confidence.** [release_metadata.yml:10–13](https://github.com/knarayanareddy/supply-chain/blob/448f6c41a6bfe59caddc6d3842b38a2506de1dee/.github/workflows/release_metadata.yml#L10-L13) / [release_metadata.yml:39–63](https://github.com/knarayanareddy/supply-chain/blob/448f6c41a6bfe59caddc6d3842b38a2506de1dee/.github/workflows/release_metadata.yml#L39-L63): the job grants `id-token: write`, `attestations: write`, and `contents: write`, then does `curl -o "${RUNNER_TEMP}/buildozer" -L ...` and later executes `"${RUNNER_TEMP}/buildozer" "set version ${VERSION}" ...`. Quote: `Download buildozer` / `chmod +x "${RUNNER_TEMP}/buildozer"` / `git commit -m "Release v${VERSION}"`. Impact: a substituted network-fetched binary can directly affect release tags and module version bumps under a privileged runner. Correction: pin a vetted digest or vendor the tool, and shrink permissions or split trust domains before execution.
- **S-02 — Medium, confirmed static, high confidence.** [purl_tables/main.py:24–30](https://github.com/knarayanareddy/supply-chain/blob/448f6c41a6bfe59caddc6d3842b38a2506de1dee/maintainers/tools/purl_tables/main.py#L24-L30) returns `0` when `LC_ALL` is not `en_US.UTF-8`, after warning the environment “will reorder the file badly.” Quote: `Please rerun as:` then `return 0`. Impact: callers can treat a failed generation precondition as success and ship stale or partial generated output. Correction: return nonzero and add a negative test that proves the bad-locale path fails.
- **S-03 — Low, owner decision, high confidence.** [ci.sh:5–12](https://github.com/knarayanareddy/supply-chain/blob/448f6c41a6bfe59caddc6d3842b38a2506de1dee/.github/workflows/ci.sh#L5-L12) maps Bazel exit code `4` to success. Quote: `Status code indicates that the build succeeded but there were no tests to run. Ignore and exit successfully.` Impact: this is acceptable only if the repo intentionally contains testless modules; otherwise it hides accidental empty coverage. Correction: define an allowlist for intentional no-test modules and fail unexpected empties.

## 4. Five-discipline assessment
**Code:** `lib/supplychain-go/cmd/sbom/main.go:13–53` validates flags, reads input, unmarshals JSON, calls `ComputeClassifications`, and writes output. That is a clear, narrow CLI boundary; classification correctness remains unverified because `internal/sbom` was not inspected. **Automation/testing:** the repo has a real CI wrapper and release workflow, but no historical CI evidence was provided and no tests were run here. **DevOps/operations:** release publication is automated through GitHub Actions and an external reusable BCR workflow; tag creation, commit, and publish authority need owner sign-off. **OWASP/security:** the dominant risk is supply-chain compromise via unverified build tools, release runners, and generated code inputs; no user auth/session surface was visible. **UI/UX/accessibility:** no web UI is present; only CLI and generated-file ergonomics apply, so accessibility concerns are limited to readable errors and deterministic output.

## 5. Production blockers and owner questions
Blockers to production confidence: which fork divergences are intentional versus expected to track upstream, who owns upstream sync and release authority for this fork, whether `buildozer` may be downloaded at runtime, whether the no-tests policy is intentional for every package, and whether BCR publish credentials/attestations are approved for this fork. Need confirmation on supported Bazel versions and whether release tags are immutable once pushed.

## 6. Evidence inventory and limitations
Inspected paths/ranges: snapshot metadata for repository SHA/tree; `release_metadata.yml:10–13,39–63`; `ci.sh:5–12`; `purl_tables/main.py:24–30`; `cmd/sbom/main.go:13–53`. Unrun checks: Bazel build/test, Go build, Python generator execution, workflow execution, upstream diff, and historical Actions review. Public URLs only; no local secrets or account data were accessed.
