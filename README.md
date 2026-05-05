# complytime-policies

Centralized repository for Gemara policies used by [ComplyTime](https://github.com/complytime) tooling. Policies defined here will be released as OCI artifacts to Quay.io and consumed via the `complyctl get` command.

## Overview

This repository contains governance artifacts used to define and enforce security controls across supported platforms (GitHub, GitLab, etc.). The governance content follows the [Gemara](https://github.com/gemaraproj/gemara) framework and is organized into catalogs, guidance, and policies.

## Repository Structure

```
complytime-policies
├── AGENTS.md
├── bundles # Declarative manifests defining which layers compose each OCI artifact
├── complytime-content  # Mapping documents expressing relationships to external frameworks
├── governance
│   ├── catalogs    # Security control catalogs and definitions
│   ├── guidance    # Guidance catalogs documenting best practices and standards
│   └── policies    # Implementation policies and technical controls
├── LICENSE
└── README.md
```

## Governance Content

### Guidance

Guidance catalogs are a structured set of guidelines — recommendations, requirements, or best practices — that help readers achieve desired outcomes. Guidelines are grouped into groups. 

- [CIS Fedora Linux Level 1 Benchmark Guidance](governance/guidance/cis-fedora-l1-guidance.yaml)

### Catalogs

Security control catalogs define the set of controls and assessment requirements. Each catalog groups controls into families and specifies the objectives and applicability of each control.

- [Branch Protection Catalog](governance/catalogs/ampel-branch-protection-catalog.yaml) - Controls for protecting source code repositories via branch protection rules

### Policies

Implementation policies define how controls are evaluated, who is responsible, and what automated tools are used for assessment.

- [Branch Protection Policy](governance/policies/ampel-branch-protection-policy.yaml) - Automated evaluation policy for branch protection controls using AMPEL

## Usage

Policies from this repository are planned to be released as OCI artifacts to Quay.io. Once available, they can be consumed using the `complyctl get` command, which retrieves policies based on a configuration file.

```bash
complyctl get
```

## License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.
