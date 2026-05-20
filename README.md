# complytime-policies

Centralized [Gemara](https://github.com/gemaraproj/gemara) policies for [ComplyTime](https://github.com/complytime) tooling. Content is published as OCI to **Quay.io** and consumed with `complyctl get` (and similar clients).

## Repository Structure

This repository contains governance artifacts used to define and enforce security controls across supported platforms (GitHub, GitLab, etc.). The governance content follows the [Gemara](https://github.com/gemaraproj/gemara) framework and is organized into catalogs, guidance, and policies.

```
complytime-policies
├── AGENTS.md
├── bundles                # Declarative manifests defining which layers compose each OCI artifact
├── complytime-content     # Mapping documents expressing relationships to external frameworks
├── governance
│   ├── catalogs           # Security control catalogs and definitions
│   ├── guidance           # Guidance catalogs documenting best practices and standards
│   └── policies           # Implementation policies and technical controls
├── LICENSE
└── README.md
```

```mermaid
flowchart LR
  policyRepo[complytime_policies_workflow] --> ghcrPush[push_to_ghcr]
  ghcrPush --> quayCopy[copy_to_quay]
  quayCopy --> refs[source_ref and destination_ref]
```

### Guidance

Guidance catalogs are a structured set of guidelines -- recommendations, requirements, or best practices -- that help readers achieve desired outcomes. Guidelines are grouped into groups.

- [CIS Fedora Linux Level 1 Benchmark Guidance](governance/guidance/cis-fedora-l1-guidance.yaml)

## Releasing

Manual publish: [`.github/workflows/publish-policy-oci.yml`](.github/workflows/publish-policy-oci.yml) (Actions --> **Publish policy OCI**). Operator steps, inputs, and the pinned action are documented in [`specs/001-policy-oci-publish/quickstart.md`](specs/001-policy-oci-publish/quickstart.md).

**Secrets (repository):** `QUAY_ROBOT_USERNAME`, `QUAY_ROBOT_TOKEN`. GHCR uses `GITHUB_TOKEN` from the workflow. Forks need their own secrets.

**Verification behavior:** successful runs now include destination digest, manifest media type, and
layer retrievability checks against Quay API endpoints. If Quay package UI appears sparse for custom
media types, use workflow verification output and quickstart API checks as the source of truth.

## Usage

See **[docs/usage.md](docs/usage.md)** for full consumer documentation: how to pull artifacts with ORAS, verify Cosign signatures, inspect via registry API, and use with `complyctl`.

Quick start:

```bash
# Pull a bundle (each bundle has its own Quay repo)
oras pull quay.io/complytime/cis-fedora-l1-workstation:latest -o ./output

# Verify the signature
cosign verify \
  --certificate-identity-regexp="https://github.com/complytime/complytime-policies/.github/workflows/" \
  --certificate-oidc-issuer="https://token.actions.githubusercontent.com" \
  quay.io/complytime/cis-fedora-l1-workstation:latest

# Or use complyctl directly
complyctl get
```

## License

Apache-2.0. See [LICENSE](LICENSE).
