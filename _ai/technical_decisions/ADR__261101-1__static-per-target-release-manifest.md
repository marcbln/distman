---
title: Static per-target release manifest on a plain fileserver
status: Accepted
date: 2026-11-01
deciders: Marc
tags: [distribution, release-engineering, contract, fileserver]
adrId: 261101-1
---

# Static per-target release manifest on a plain fileserver

## Context

distman publishes internal app releases to a fileserver reachable only from inside
the Tailscale network. Staff-facing clients (macOS, Linux, Windows) must discover
the latest version of each app, verify downloads, and show release notes. Builds
run on each target operating system — cross-compilation is not reliable — so
different machines publish different targets at different times. There is no
database, API server, or CI coordinator we want to operate for v1.

## Decision

The fileserver holds only static files:

```
/v1/catalog.json              apps metadata, curated links, message of the day
/v1/latest-<os>-<arch>.json   per-target latest per app
/artifacts/<app>/<version>/<os>-<arch>/<app>-<version>-<os>-<arch>.tar.zst
```

`latest-<os>-<arch>.json` maps app id → `{version, publishedAt, notes, entry,
archive:{path, size, sha256}}`. A publisher only ever writes the manifest for the
target it built, so no file has two writers and there is no merge step.

Publish order is crash-safe: upload the artifact to a temporary name and rename it
into place, verify size and sha256 on the server, then write the manifest
atomically (`.tmp` + rename), retaining the previous manifest. Clients fetch the
catalog plus their own target manifest with conditional GET (ETag); artifacts are
immutable once published. `schema` is a major version; new fields are added
additively.

## Consequences

- No server logic, no database, no API quota; nginx/rsync, a mounted share, or an
  S3-compatible store all work.
- Concurrent publishing across targets is safe; a half-finished publish is
  invisible to clients.
- Different platforms may legitimately run different versions; this is accepted.
- `catalog.json` is globally written (apps, links, MOTD) and therefore needs a
  single writer or an explicit merge step.
- No per-user access control yet (see ADR 4); manifest size grows with app count
  but stays small.
- Rollback of a bad publish is restoring the previous manifest.

## Alternatives Considered

- **One global manifest for all targets** — needs a coordinator and merge logic;
  concurrent publishes race. Rejected.
- **A small API/database server** — adds operational surface and state for data
  that only changes at publish time. Rejected for v1.
- **Client reads directory listings** — no checksums, notes, or atomic publish
  semantics; every client would parse server-specific listings. Rejected.

## Related Decisions

`ADR__261101-2__portable-versioned-dir-installs.md`,
`ADR__261101-4__tailnet-auth-server-side-filtering.md`
