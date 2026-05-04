# Contract: thin caller pipeline (Option 3)

This repository is a **caller only**. Canonical OCI bundle semantics remain in **go-gemara** and the
pinned composite action (today: [`sonupreetam/gemara-publish-oci`](https://github.com/sonupreetam/gemara-publish-oci); org name `complytime/oci-artifact` is an equivalent target when the same `action.yml` is published there).

## A. Caller workflow contract

**Consumer:** `complytime-policies/.github/workflows/publish-policy-oci.yml`  
**Provider:** `sonupreetam/gemara-publish-oci@4314defdfcc96129775d731d7d64a4aa960a8527` (see comment in workflow; [source PR #4](https://github.com/sonupreetam/gemara-publish-oci/pull/4))

### Required `workflow_dispatch` inputs

| Input | Description |
|-------|-------------|
| `release_tag` | Tag on GHCR and destination registry. |
| `bundle_file` | Root Gemara bundle YAML (repo-relative). |
| `dest_image` | Destination path `namespace/repo` without registry host. |
| `trust_mode` | `resign`, `copy-referrers`, or `copy-only` (passed to the action). |
| `verify_quay` | If `true`, run `cosign verify` on the **destination** digest after promote (proves signature in CI). |
| `bypass_unchanged_check` | If `true`, always publish regardless of content-hash cache (default `false`). |

Optional: `allow_unprotected_ref` (forks / unprotected default branch).

### Required secrets

| Secret | Use |
|--------|-----|
| `GITHUB_TOKEN` | GHCR push/pull (`packages: write`). |
| `QUAY_ROBOT_USERNAME` | Destination registry promotion. |
| `QUAY_ROBOT_TOKEN` | Destination registry promotion. |

### Required permissions

- `contents: read`
- `packages: write`
- `id-token: write` (keyless cosign when signing/verify steps run)

## B. Action input mapping (actual `with:`)

The thin caller maps into the composite as implemented in the workflow (names must match `action.yml`):

| Caller / dispatch | Action input |
|---------------------|--------------|
| `publish_mode` | `gemara-file` |
| `registry` | `ghcr.io` |
| `repository` | lowercased `${{ github.repository }}` |
| `tag` | `release_tag` |
| `file` | `bundle_file` |
| `promote_to_destination` | `"true"` |
| `destination_registry` | `quay.io` |
| `destination_repository` | `dest_image` |
| `destination_tag` | `release_tag` |
| `destination_username` / `destination_password` | Destination robot secrets |
| `trust_mode` | dispatch `trust_mode` (default `resign`) |
| `sign_source` / `verify_source` | `"true"` — keyless cosign on GHCR staging digest |
| `sign_destination` | `"true"` — keyless cosign on destination digest after promote |
| `verify_destination` | from `verify_quay` → `"true"` / `"false"` |
| `allowed_identity_regex` | `^https://github.com/<this repo>/.github/workflows/` |

> **Note:** The composite is the **accepted production path** (Session 2026-05-04). It embeds GHCR publish, keyless cosign sign/verify on source and destination, and ORAS copy to the destination registry in a single atomic `uses:` step. No separate `workflow_call` to org-infra promote reusables is required.

## C. Output contract

- `source_ref`, `source_digest` / `digest`
- `destination_ref`, `destination_digest`
- `verified_source`, `verified_destination` (when verify steps run)

## D. Ordering invariant

The composite enforces: publish → optional source sign/verify → optional promote → optional destination sign/verify (see action `finalize` step).
