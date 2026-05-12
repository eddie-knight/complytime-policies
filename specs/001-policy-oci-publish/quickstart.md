# Quickstart: Publish policy OCI (maintainers)

## Prerequisites

- Policy YAML under **`governance/`** and **`bundles/`** passes merge-time validation in this repository.
- GitHub **secrets** (names only): [`README.md` root](../../README.md) and this file — `QUAY_ROBOT_USERNAME`, `QUAY_ROBOT_TOKEN` for the destination registry; GHCR uses `GITHUB_TOKEN` from the workflow.
- **Pinned** composite SHA: in [`.github/workflows/publish-policy-oci.yml`](../../.github/workflows/publish-policy-oci.yml) only (bump when the action changes; **FR-006** in `specs/001-policy-oci-publish/spec.md`).

## How the thin caller works

1. **Checkout** the **default branch** in the workflow.
2. The caller invokes one pinned composite: `uses: gemaraproj/gemara-registry-cli@14b177e380a5797cc236775be8be2712b5bac393` ([PR #2](https://github.com/gemaraproj/gemara-registry-cli/pull/2) `feat/gemara-bundle-publish`); see the workflow and **FR-006** for the current pin.
3. The action publishes to GHCR, signs and verifies with keyless cosign on both source and destination, and promotes via ORAS copy to the destination registry per `with:` inputs (`trust_mode`, `verify_quay`, `dest_image`, etc.) in a single `uses:`.

**Defaults:** `trust_mode: resign`, `verify_quay: true`, `sign_source: true`, `verify_source: true`, `sign_destination: true` (see **spec** Sessions 2026-04-27 and 2026-05-04). You can set `copy-only` for a stricter no-resign path when debugging.

No in-repo redefinition of OCI manifest layout; see [contracts/publish-pipeline.md](contracts/publish-pipeline.md).

**Content-hash caching:** the workflow hashes `bundles/` and `governance/` on each run and skips publish when content is unchanged since the last successful run. Set `bypass_unchanged_check: true` to force a publish (re-promote, emergency, or after cache eviction).

**Tag-exists guard:** before invoking the composite, the workflow runs `oras resolve` against both GHCR and the destination registry. If the `release_tag` already exists on both registries, publish is skipped. If it exists only on GHCR (but not the destination), the workflow proceeds to complete promotion. `bypass_unchanged_check: true` overrides both the content-hash gate and the tag-exists gate.

**Overlapping runs:** workflow `concurrency` group `publish-policy-oci` with `cancel-in-progress: false` (**FR-002**). The composite defines tag/overwrite behavior (see [contracts/publish-pipeline.md](contracts/publish-pipeline.md)).

**Who can run:** the job runs only when the ref is **protected** (`github.ref_protected`). Forks without branch protection cannot run this workflow. Optional [GitHub Environment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment) gating is not in the default workflow; add `environment: <name>` in agreement with your org if required.

## Run the release workflow

1. Go to **Actions** → **Publish policy OCI** → **Run workflow** (on the default branch).
2. **release_tag (required):** one tag for GHCR and destination registry.
3. Optional: `bundle_file` (default `governance/policies/cis-fedora-l1-workstation-policy.yaml`), `dest_image`, `trust_mode`, `verify_quay`, `bypass_unchanged_check`.
4. On success, copy `source_ref`, `destination_ref`, and (when `verify_quay: true`) `verified_destination` from logs.
5. Confirm destination artifact verification summary in workflow output:
   - `destination_digest`
   - `manifest_media_type`
   - `artifact_type` (when present)
   - `layer_count`

**Workflow file:** [`.github/workflows/publish-policy-oci.yml`](../../.github/workflows/publish-policy-oci.yml)

## Local / fork checks

- **Forks** usually cannot complete E2E without fork-level destination registry secrets and permissions.
- **Dry checks:** `yamllint` / schema validation of edited YAML only, not a full push.

## First E2E (org credentials)

1. Merge the desired policy/bundle state to the **default branch**.
2. Run **Publish policy OCI** with a **new** `release_tag`.
3. Confirm source and destination refs plus verification outputs.
4. For consumers, use **Usage** in the root `README` (or the **Consumers** section below) and **SC-002** in the spec.

## Verified E2E

*After a successful* `workflow_dispatch`*, add run URL,* `release_tag`*, and destination digest here (for* **SC-003** */ maintainer record).*

- Run [25314153282](https://github.com/sonupreetam/complytime-policies/actions/runs/25314153282): `release_tag=verify-4314def-destination`, destination `sha256:35854f26...`, `verified_destination=true` (signing off, verify only)
- Run [25314721510](https://github.com/sonupreetam/complytime-policies/actions/runs/25314721510): `release_tag=verify-cosign-enabled`, destination `sha256:29ffb7bd...`, `verified_destination=true`, cosign sign+verify on both GHCR and Quay
- Run [25320372118](https://github.com/sonupreetam/complytime-policies/actions/runs/25320372118): `gemara-publish-oci` pin (`50b4f59`), 3 layers (policy 72KB + catalog 89KB + guidance 11KB), `verified_destination=true`
- Run [25423828312](https://github.com/sonupreetam/complytime-policies/actions/runs/25423828312): **`gemaraproj/gemara-registry-cli`** pin (`14b177e`), tag-exists guard active, all stages passed

**Migration status:** the composite action has moved to [`gemaraproj/gemara-registry-cli`](https://github.com/gemaraproj/gemara-registry-cli) ([PR #2](https://github.com/gemaraproj/gemara-registry-cli/pull/2)). The workflow is pinned to a full SHA on that repo. When PR #2 merges to `main`, update the pin to a stable release tag and retire the branch SHA pin per **FR-006**.

## Consumers: fetch and verify (SC-002)

After a successful publish, the public image is at **`quay.io/complytime/complytime-policies:<release_tag>`** (see spec **Clarifications** if that path changes). Replace **`<tag>`** with the dispatch **`release_tag`** (or a **`<sha-…>`**-derived tag if your org workflow enabled SHA-style tags).

1. **Resolve digest** (optional but recommended for pinning):

   ```bash
   crane digest "quay.io/complytime/complytime-policies:<tag>"
   ```

2. **Verify cosign signature**:

   ```bash
   cosign verify "quay.io/complytime/complytime-policies@sha256:<digest>" \
     --certificate-identity-regexp '.*' \
     --certificate-oidc-issuer-regexp '.*'
   ```

  Tighten identity/issuer regexes to your org policy.

3. **Pull or copy** the artifact with your usual OCI tool (`skopeo`, `oras`, `complyctl get`, etc.).

## Destination verification (UI-independent)

Use these checks when Quay package UI appears sparse for custom media types.

1. **Fetch destination manifest metadata**:

   ```bash
   curl -sS \
     -H 'Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.oci.artifact.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json' \
     "https://quay.io/v2/<dest_image>/manifests/<tag>" \
     | jq '{mediaType, artifactType, layers: [.layers[] | {mediaType, digest, size}]}'
   ```

2. **Verify every destination blob is retrievable**:

   ```bash
   curl -sS \
     -H 'Accept: application/vnd.oci.image.manifest.v1+json, application/vnd.oci.artifact.manifest.v1+json, application/vnd.docker.distribution.manifest.v2+json' \
     "https://quay.io/v2/<dest_image>/manifests/<tag>" \
     | jq -r '.layers[].digest' \
     | while read -r d; do
         echo "checking $d"
         curl -fsSIL "https://quay.io/v2/<dest_image>/blobs/$d" >/dev/null
       done
   ```

3. If the API checks pass but UI remains sparse, treat the artifact as valid and continue using
   digest-pinned references.

## Cosign caveat (FR-004)

If a run fails on verification, check `allowed_identity_regex`, `trust_mode`, and destination
signing behavior for the selected mode.
