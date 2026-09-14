# Recovery notice — Phase 2 working tree loss

**Date:** 2026-09-14

## What happened

The remote development container that held `telegram-rewards/` was recycled. The
repository was re-cloned fresh at `5d30d82`, and the working tree containing the
Phase 2 implementation was destroyed.

Twenty-three commits of Phase 2 work never reached this remote. Throughout that
session `git push` failed with HTTP 403 — the Claude GitHub App did not have
access to this repository. Because the work existed only in the container, the
container's recycling lost it.

`git ls-remote` at the time of writing showed exactly one ref, `refs/heads/main`
at `5d30d82`, carrying the four original data-science files. No Phase 2 code was
ever present on GitHub.

## Current state of access

GitHub App access has since been restored; pushes now authenticate successfully.

One server-side policy remains: pushing the pre-existing upstream commits to a
new ref is rejected with `GH007` ("Your push would publish a private email
address"), because commits `fe0ba2c`..`5d30d82` are authored with a private
address. This branch was therefore created through the GitHub API instead.
Commits made from the development container are authored `noreply@anthropic.com`
and are not affected. To remove the restriction entirely, uncheck *"Block command
line pushes that expose my email"* at <https://github.com/settings/emails>.

## What survives

The complete Phase 2 source exists only in build artifacts that were delivered
into the working session, not in this repository:

| Artifact | Contents | Use as restore point? |
| --- | --- | --- |
| `telegramrewardsphase2-secure.zip` / `.bundle` (final delivery) | Round 5 final state, clean HEAD `018edb2`, 115 files, squashed history | **Yes** |
| `heartdata-branch-claude-telegram-rewards-phase-1-s6zf2l.bundle` | Commit-by-commit history to `059c892` | **No** — predates rounds 4 and 5 |

A partial salvage was additionally recovered from the retained session
transcript: eight complete files (migrations `0008`–`0011`, `internal/plane/plane.go`,
and the `cmd/user-api`, `cmd/admin-api`, `cmd/webhook` entrypoints) plus edit
hunks for sixteen more. That is 8 of roughly 115 files and is a fallback only.

## Restoring

```sh
git clone telegramrewardsphase2-secure.bundle restored
# or
unzip telegramrewardsphase2-secure.zip -d restored
```

The Phase 2 security gate stood at **BLOCKED** at the end of round 5 — not on any
open implementation defect, but on missing evidence: CI had never executed the
test suite, no hardware WebAuthn drill had been performed, and `govulncheck`
could not reach `vuln.go.dev` or `api.osv.dev`. Restored GitHub access clears the
path for the first of those.

## What must not happen

The Phase 2 tree must not be reconstructed from memory. It carries five rounds of
independent security review; a reconstruction would reproduce the shape of the
reviewed artifact without its review or test history, which is worse than having
nothing. Restore from the bundle, or rebuild deliberately from scratch — not from
recollection.
