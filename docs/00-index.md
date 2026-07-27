# Nostr Kanban — Study Notes and Design Record

Research notes for building **private (encrypted) Kanban boards on Nostr** and a
from-scratch SDK for them, under the `formstr` umbrella.

Compiled 2026-07-28. Everything below reflects the state of the sources on that
date; each doc names the exact commit or PR revision it read.

---

## What this is

The public half of Nostr Kanban already exists as a draft protocol
([NIP PR #1665](https://github.com/nostr-protocol/nips/pull/1665)) and one
reference client ([kanbanstr](https://github.com/vivganes/kanbanstr)). Neither
has any notion of a private board — every board title, column name, card, and
assignee is plaintext on public relays.

These docs do three things:

1. Read the existing protocol and implementation closely enough to know exactly
   what is specified, what is merely implemented, and where the two disagree.
2. Survey the ways Nostr already does encrypted multi-party state, and pick one.
3. Specify private boards and the SDK that will implement both halves.

---

## Reading order

| # | Doc | What it answers |
|---|-----|-----------------|
| 01 | [Nostr primitives](01-nostr-primitives.md) | Which NIPs Kanban stands on, and the specific rules that constrain the design |
| 02 | [NIP-100 analysis](02-nip-100-analysis.md) | What PR #1665 actually specifies, clause by clause, plus every unresolved review objection |
| 03 | [kanbanstr review](03-kanbanstr-review.md) | How the reference client is built, and where it diverges from its own spec |
| 04 | [Key models — prior art](04-key-models-prior-art.md) | View keys vs shared replaceables vs NIP-29 vs MLS, and why we chose what we chose |
| 05 | [Private Kanban spec](05-private-kanban-spec.md) | The draft protocol for encrypted boards and cards |
| 06 | [SDK architecture](06-sdk-architecture.md) | `@formstr/kanban-sdk` — modules, API surface, interop rules |
| 07 | [Gaps and risks](07-gaps-risks.md) | Everything known to be unresolved, wrong, or dangerous |
| 08 | [New vs existing](08-new-vs-existing.md) | What is copied from formstr, what is extended, what is genuinely novel |

Docs 01–04 are **analysis**. Docs 05–07 are **design** and will change as the
implementation lands. Doc 08 is the risk map: it says which parts of the design
have production mileage behind them and which do not.

---

## Decisions already made

Recorded here so they don't get re-litigated. Rationale lives in doc 04.

| Decision | Choice |
|---|---|
| Encryption model for private boards | **Per-board view key**, matching `nostr-calendar`'s NIP-52E — *not* shared-signing-key (PR #1228) |
| Private card discovery | **Blinded board pointer** — a `b` tag whose value only view-key holders can compute |
| SDK location | `common-packages/packages/kanban-sdk`, sibling of `calendar-sdk`, reusing `@formstr/signer` and `@formstr/local-relay` |
| Public-board wire format | Unchanged from NIP-100, so kanbanstr.com interop survives |

---

## Source snapshot

| Source | Revision read |
|---|---|
| NIP PR #1665 "Multi-User Kanban boards on Nostr" | open, unmerged; opened 2025-01-01, last updated 2026-03-04; 8 commits, 1 file |
| `nostr-protocol/nips` master | kind registry checked for `30301`, `30302`, `32xxx`, `10xx` |
| `vivganes/kanbanstr` | `bf36bd8` (2026-04-23) |
| NIP PR #1228 "Shared replaceables via Event-owned keys" | open, unmerged |
| `nostr-calendar/nips/NIP-52E.md` | working tree |
| `common-packages/packages/calendar-sdk` | working tree |
| `nostr-forms/packages/formstr-app/src/nostr/accessControl.ts` | working tree |

---

## Glossary

**Addressable event** — kind `30000`–`39999`. Identified by the coordinate
`kind:pubkey:d-tag`; relays keep only the newest one per coordinate. Both Kanban
kinds are addressable, and nearly every hard problem in these docs traces back to
that.

**Coordinate** — `kind:pubkey:d-tag`, the string that goes in an `a` tag.

**View key** — a freshly generated secret key used *only* to encrypt one object's
content. Not anybody's identity key. Sharing the view key shares read access.
See doc 04.

**Blinded pointer** — a hash of a board coordinate and its view public key, used
as a relay-queryable tag that reveals nothing to the relay. See doc 05.

**Rumor** — an unsigned Nostr event, the innermost layer of a NIP-59 gift wrap.

**Maintainer** — in NIP-100, a pubkey `p`-tagged on the board event, permitted to
create and edit cards.

**Tracker card** — a card that mirrors some other Nostr event (a git issue,
another board's card) instead of holding its own content.
