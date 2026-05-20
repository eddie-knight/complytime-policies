# ADR-0001: One Quay Repository Per Policy Bundle

- **Status:** Accepted
- **Date:** 2026-05-20
- **Deciders:** gvauter, jpower432, sonupreetam
- **PR Context:** [complytime-policies#33](https://github.com/complytime/complytime-policies/pull/33)

## Context

The initial CI implementation published all Gemara policy bundles to a
single OCI repository (`quay.io/complytime/complytime-policies`) using
the bundle name as the tag (e.g., `:ampel-branch-protection`,
`:cis-fedora-l1-workstation`). This overloads the tag field as both
an artifact selector and a version identifier, preventing standard
version tagging per bundle.

Three alternatives were evaluated:

1. **Single repo, bundle-name tags (status quo)** — no version history.
2. **Single repo, compound tags** (e.g., `ampel-branch-protection-v1.0.0`)
   — enables versioning but is non-standard and breaks tooling assumptions.
3. **One Quay repo per bundle** — each bundle gets its own repository with
   standard version tags.

## Decision

Adopt **one Quay repository per policy bundle** under the `complytime`
namespace. Each bundle published from `bundles/*.yaml` will be pushed to
its own repository (e.g., `quay.io/complytime/ampel-branch-protection`)
with standard version tags (`latest`, semver).

## Rationale

- **Standard OCI semantics.** Tags mean "version" — consistent with Helm,
  Tekton, Wolfi, and other OCI-native ecosystems.
- **Independent release cadence.** Each bundle can version independently
  without coordinating across unrelated policies.
- **Tooling compatibility.** Cosign, Renovate, Dependabot, and ORAS all
  assume `tag = version of one artifact`.
- **Cleaner Quay UI.** Each repo shows version history for exactly one
  bundle.
- **Per-bundle access control.** Quay permissions can be scoped per
  repository if needed in the future.
- **Consumer clarity.** `complytime.yaml` references become self-describing:
  `url: quay.io/complytime/ampel-branch-protection@v1.0.0`.
- **complyctl is agnostic.** The consumer CLI treats the URL as an opaque
  OCI reference — both single-repo and multi-repo references work without
  code changes (verified in `internal/registry/client.go` and
  `internal/complytime/config.go`).

## Consequences

### CI Workflow (complytime-policies)

- The `publish-policy-oci.yml` workflow must compute
  `destination_repository` per matrix entry as
  `complytime/${BUNDLE_NAME}` instead of a single shared repo.
- Tag strategy: `latest` on main-branch pushes; semver on release
  (via `workflow_dispatch` input or Git tag trigger).
- Quay repos will be auto-created on first push if the org robot
  account has org-level repository-create permissions. No manual
  repo creation required.
- Content-hash caching keys must include the per-bundle repo name
  (already does via `matrix.bundle.name`).

### Consumer (complyctl)

- No code changes required. `ParsePolicyRef` handles any valid OCI
  reference.
- `complytime.yaml` examples and `QUICK_START.md` must be updated to
  use per-bundle repository paths.

### Documentation

- `docs/usage.md` must be updated with new reference format.
- This ADR is the canonical record of the decision.
- A cross-reference to this ADR should exist in `complyctl/docs/`.

### Migration

- Existing tags in the single repo remain available until deprecated.
- New publishes will target per-bundle repos immediately.
- No breaking change to consumers who pin by digest.

## Alternatives Considered

### Single repo with compound tags

```
quay.io/complytime/complytime-policies:ampel-branch-protection-v1.0.0
```

**Rejected because:** Non-standard tag format breaks automated version
detection tooling. Tag listing grows noisy with bundles x versions.
Functionally equivalent to multi-repo but without the ecosystem benefits.

### Status quo (bundle name as sole tag)

```
quay.io/complytime/complytime-policies:ampel-branch-protection
```

**Rejected because:** No version history per bundle. Cannot roll back
to a previous version. Overwrites on every publish.
