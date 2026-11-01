---
title: Tailnet as transport auth, server-side filtering for future IAM
status: Accepted
date: 2026-11-01
deciders: Marc
tags: [security, auth, tailscale, iam, toolbox]
adrId: 261101-4
---

# Tailnet as transport auth, server-side filtering for future IAM

## Context

Only devices inside the Tailscale network can reach the release fileserver, and
for v1 that is the entire access-control story. Later, identity and authorization
will move to IAM (groups, possibly per-role app and link visibility). We do not
want to write an auth system now, and we do not want the client to be rewritten
when IAM arrives.

## Decision

- v1 authorization is the tailnet: if a machine can reach the fileserver, it may
  read every artifact and manifest. Access revocation is a Tailscale ACL change.
- The client has **no permission model**. It always asks the server for its
  catalog and target manifest and renders whatever it receives. Today the static
  server returns everything; later an IAM-gated endpoint (or per-group manifest
  generation) filters server-side and the client is unchanged.
- Manifest asset paths stay **relative** to a configurable base URL, so moving
  from `http://100.x/dist/` to `https://apps.topdata.internal/` is configuration,
  not code.
- The client degrades gracefully off-network: cached catalog and apps still work,
  update checks report "off network", link health dots go grey.

## Consequences

- No authentication or authorization code in v1.
- IAM becomes a server-side change; the client contract does not shift.
- Every staff member sees every app and link until server-side filtering exists.
- Tailnet membership is coarse-grained; per-team visibility waits for IAM.
- The client must never embed assumptions such as "if it is in the manifest, this
  user may install it" beyond rendering what the server sent.

## Alternatives Considered

- **Client-side group checks** — duplicates policy in every client, drifts from
  the server. Rejected.
- **Application-level SSO now** — significant scope for a v1 that is already
  gated by the tailnet. Rejected.
- **Per-group static manifest files** — viable later, but needs a generator that
  knows identities; defer until IAM exists.

## Related Decisions

`ADR__261101-1__static-per-target-release-manifest.md`
