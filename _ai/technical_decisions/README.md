# Architecture Decision Records

Architecture decision records (ADRs) for **distman**: significant,
hard-to-reverse technical decisions, one file per decision named
`ADR__<YYMMDD-N>__<kebab-case-title>.md`.

## Index

### 2026

- `ADR__261101-1__static-per-target-release-manifest.md` — static fileserver
  contract with one manifest per target, atomic publish order.
- `ADR__261101-2__portable-versioned-dir-installs.md` — tar.zst payloads,
  versioned install directories, no OS installers, instant rollback.
- `ADR__261101-4__tailnet-auth-server-side-filtering.md` — tailnet as v1 auth,
  client renders server-filtered manifests, IAM-ready.
