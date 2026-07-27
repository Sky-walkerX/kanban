# kanban

Private Kanban boards on Nostr, and an SDK for them.

Currently **research and design only** — no code yet. Everything lives in
[`docs/`](docs/).

## Start here

[`docs/00-index.md`](docs/00-index.md) — reading order, decisions, glossary.

## The short version

[NIP PR #1665](https://github.com/nostr-protocol/nips/pull/1665) specifies public
Kanban boards on Nostr (kinds `30301` board, `30302` card).
[kanbanstr](https://github.com/vivganes/kanbanstr) implements it. Neither has any
notion of privacy — every board title, column, card, and assignee is plaintext on
public relays.

This work adds private boards using the same per-board **view key** pattern that
[`nostr-calendar`](../nostr-calendar) and
[`calendar-sdk`](../common-packages/packages/calendar-sdk) already ship for
private calendar events, and specifies `@formstr/kanban-sdk` to implement both the
public and private halves.

## Docs

| Doc | Contents |
|---|---|
| [01](docs/01-nostr-primitives.md) | The NIP rules that constrain the design |
| [02](docs/02-nip-100-analysis.md) | PR #1665 analysed, plus every unresolved review objection |
| [03](docs/03-kanbanstr-review.md) | The reference client: architecture and defects |
| [04](docs/04-key-models-prior-art.md) | Five ways to do encrypted multi-party state, and the choice |
| [05](docs/05-private-kanban-spec.md) | NIP-100E draft — private boards and cards |
| [06](docs/06-sdk-architecture.md) | `@formstr/kanban-sdk` design |
| [07](docs/07-gaps-risks.md) | What is unresolved, wrong, or dangerous |
| [08](docs/08-new-vs-existing.md) | What is reused from formstr vs genuinely new |

## Decisions

- Private boards use a **per-board view key** (NIP-52E parity), not shared signing
  keys (PR #1228) — see doc 04
- Private cards are found by a **blinded board pointer**, a `b` tag only key
  holders can compute — see doc 05 §2
- The SDK lands in `common-packages/packages/kanban-sdk`, sibling of
  `calendar-sdk`
- Public boards keep NIP-100's wire format exactly, so kanbanstr.com interop
  survives
