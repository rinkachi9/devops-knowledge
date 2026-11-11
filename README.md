# DevOps Knowledge

A personal knowledge base about DevOps: the culture and practices, the operating system and networking foundations they rest on, and the tool families for building, shipping, running and observing software. Organized along the roadmap.sh DevOps roadmap and the DevOps lifecycle.

## Table of Contents

- [About](#about)
- [Purpose](#purpose)
- [Scope](#scope)
- [Areas](#areas)
- [Full table of contents](#full-table-of-contents)
- [How to use this repository](#how-to-use-this-repository)
- [Reading the knowledge base](#reading-the-knowledge-base)
- [Repository layout](#repository-layout)
- [Sources and migration](#sources-and-migration)
- [Related repositories](#related-repositories)
- [Contributing and working with AI agents](#contributing-and-working-with-ai-agents)
- [Status](#status)

## About

This repository collects what I learn as a DevOps engineer: how operating systems and networks work, how software moves from commit to production, how infrastructure is defined as code, how containers and clouds run it, and how it is observed and secured. Each note explains one topic in plain English, with diagrams, a worked example, practice questions and links to primary sources. The notes are written so that they are still useful when I come back after a year, and so that someone new to a topic can follow them from top to bottom.

This is a personal knowledge base, not a tutorial site or official documentation. Tools change quickly; every note records what was checked and when, and marks what still has to be verified with `TODO:` lines.

## Purpose

The repository exists to:

- build a durable, vendor-neutral DevOps reference that I can search, extend and trust,
- force real understanding by explaining every concept precisely, with examples, analogies and diagrams,
- give every new topic a predictable place, by following an established division of the field,
- keep one consistent standard, so the collection stays coherent as it grows,
- provide a source of truth that both people and AI assistants can work with under the same rules.

## Scope

**In scope**

- DevOps culture, practices and metrics.
- Operating system and Linux administration knowledge, and networking from the link layer to applications, as far as DevOps work needs it.
- CI/CD, GitOps, infrastructure as code, configuration management, containers, virtualization, cloud service models, observability, DevSecOps, SRE and operational data pipelines.
- Tools as examples of these categories, with commands, configuration and versions.

**Out of scope**

- Credentials, real host names, internal address plans or any other real identifiers.
- Topics large enough for their own repository: Terraform, Kubernetes, provider-specific cloud services, deep data analytics. This repository links to them.

## Areas

The top level follows the skill groups of the [roadmap.sh DevOps roadmap](https://roadmap.sh/devops), regrouped along the DevOps lifecycle and the [DORA capability catalog](https://dora.dev/capabilities/). Inside an area, subareas follow the established division of the discipline: the classic operating system textbook division for operating systems, and the layers of the Internet protocol suite (RFC 1122 and RFC 1123) for networking. The placement rules are in [GUIDELINE.md](GUIDELINE.md#311-placement-of-a-note).

| Area | What it covers |
| --- | --- |
| [Foundations](foundations/README.md) | What DevOps is: culture, principles, lifecycle, DORA metrics. |
| [Operating systems](operating-systems/README.md) | OS concepts, processes, memory, file systems and I/O, permissions, boot and systemd, shell, distributions. |
| [Networking](networking/README.md) | Models, link, internet, transport and application layers, DNS, email, network security, traffic management, operations. |
| [Version control](version-control/README.md) | Git, branching strategies, repository hosting (planned). |
| [CI/CD](ci-cd/README.md) | Continuous integration, delivery and deployment, GitOps, Jenkins. |
| [Infrastructure as code](infrastructure-as-code/README.md) | IaC concepts and configuration management; Terraform lives in its own repository. |
| [Containers and virtualization](containers-and-virtualization/README.md) | Hypervisors and virtual machines, Docker. |
| [Cloud computing](cloud-computing/README.md) | Cloud providers, IaaS, PaaS, serverless. |
| [Observability](observability/README.md) | Observability, monitoring, monitoring tools, log management. |
| [DevSecOps](devsecops/README.md) | Secrets management and security in delivery. |
| [Site reliability engineering](site-reliability-engineering/README.md) | SLOs, error budgets, incidents, postmortems (planned). |
| [Data processing](data-processing/README.md) | ETL and operational data pipelines. |

## Full table of contents

[ToC.md](ToC.md) lists every file in the repository as an indented tree, with planned topics shown without links. Use the area READMEs for the recommended reading order.

## How to use this repository

- To learn a topic, open the area, then read the notes in the order the area `README.md` gives.
- To look something up, start from [ToC.md](ToC.md) or search with `grep -rn "term" --include="*.md" .`.
- To find open gaps and claims still to verify, run `grep -rn "^TODO:" --include="*.md" .`.

## Reading the knowledge base

This repository is set up for [Librarian OS](https://japonics.io), a reader for Markdown knowledge repositories. [librarian-os.json](librarian-os.json) names the project and lists the entry points (the area directories). Librarian OS renders the notes with Mermaid diagrams, search, a graph of the relations between notes and practice questions.

```bash
# in the librarian-os directory
npm install
npm run dev -- --repo /absolute/path/to/this/repository
```

Every `note.md` has a `note.md.json` sidecar with its id, relations, sources and key concepts, as described in [GUIDELINE.md](GUIDELINE.md#35-sidecar-metadata). The tools that keep them valid run from Librarian OS: `npm run check`, `npm run scaffold` and `npm run toc`, each with this repository's path as the argument.

## Repository layout

```text
ToC.md                  # full tree of every file
GUIDELINE.md            # rules for structure, note format and style
INSTRUCTIONS.md         # Librarian OS adaptation guide
_sources/               # original exports that notes are migrated from (not committed)
librarian-os.json       # configuration for Librarian OS (project name, entry points, rules)
_assets/<area>/         # images and other files used by notes
<area>/
    README.md           # notes in reading order and planned topics
    README.md.json      # its sidecar
    <topic>.md          # a single note
    <topic>.md.json     # its sidecar: node header, relations, sources, concepts
    <subarea>/          # group of related notes, with its own README
```

## Sources and migration

The notes were migrated from an export of my earlier DevOps notes (a ClickUp document) kept locally in `_sources/` and excluded from version control. The content was restructured into the areas above, translated into the note skeleton of the GUIDELINE, extended with diagrams, worked examples and practice questions, and checked for outdated or incorrect statements, which were corrected in the notes. Statements that still need verification are marked with `TODO:` lines. Images from the original notes, mostly third-party graphics, are kept in `_assets/` for personal study with credit in their alt text; they should be redrawn as Mermaid or SVG over time. The Terraform part of the export was migrated to the separate Terraform knowledge repository.

## Related repositories

- `terraform-knowledge`: Terraform in depth (configuration language, state, modules, testing, HCP Terraform).
- `kubernetes-knowledge`: Kubernetes.
- `google-cloud-knowledge`: Google Cloud products and architectures.
- `cybersecurity-knowledge`: security beyond DevSecOps.
- `software-engineering-knowledge`: software engineering concepts, architecture and design.

## Contributing and working with AI agents

All notes follow [GUIDELINE.md](GUIDELINE.md). It applies to people and to AI agents alike. Agent-specific entry points are [AGENTS.md](AGENTS.md) and [CLAUDE.md](CLAUDE.md). Both point to the GUIDELINE and do not repeat its rules.

## Status

All twelve areas exist with their subareas and planned topics; ten areas have notes, and version control and site reliability engineering contain only their structure and planned topics. Every note is in `draft` status until it has been checked against current documentation.
