# 06 — `@formstr/kanban-sdk` Architecture

A headless TypeScript SDK for Nostr Kanban: public boards byte-compatible with
kanbanstr.com, plus the private boards specified in
[doc 05](05-private-kanban-spec.md).

Written from scratch, but deliberately shaped like
[`@formstr/calendar-sdk`](../../common-packages/packages/calendar-sdk) — same
layering, same runtime contract, same crypto. A developer who has read one should
be able to navigate the other without re-learning anything.

---

## 1. Placement

```
common-packages/packages/
  core/
  signer/            @formstr/signer — NIP-07 / NIP-46 / NIP-55 / NIP-49
  local-relay/       @formstr/local-relay — shared-worker relay, outbox, offline
  calendar-sdk/
  kanban-sdk/        ← new
```

A sibling in the existing pnpm workspace, not a standalone repo. It reuses
`@formstr/signer` and takes `@formstr/local-relay` as an **optional peer
dependency**, exactly as `calendar-sdk` does.

Runtime dependencies: `nostr-tools`, `@noble/hashes`. Nothing else. No NDK — the
whole formstr stack is `nostr-tools`-based, and kanbanstr's NDK coupling is one of
the reasons its code is not liftable (doc 03 §8).

Build: `tsup` → ESM + CJS + `.d.ts`. Tests: `vitest`.

---

## 2. Module map

```
src/
  KanbanSDK.ts              facade — the only thing most consumers touch
  contracts.ts              KanbanSigner, NostrRuntime, KanbanCtx, error types
  kinds.ts                  kind registry with the reasoning inline
  types.ts                  Board, Card, Column, BoardList, Invitation, …

  crypto/
    nip44.ts                selfEncrypt / selfDecrypt        ← from calendar-sdk
    nip59.ts                seal / wrap / unwrap + verify    ← from calendar-sdk
    viewKey.ts              generate / encrypt / decrypt     ← from calendar-sdk
    localSigner.ts          raw-secret-key signer            ← from calendar-sdk
    blindedPointer.ts       NEW — doc 05 §2

  codec/                    pure event ⇄ object. No I/O.
    board.ts                public 30301 + private 32301
    card.ts                 public 30302 + private 32302
    boardList.ts            32303
    invitation.ts           1053 / 53 rumor
    link.ts                 i-tag card links, both directions
    tracker.ts              k / e / refs/* + git-issue status derivation
    rank.ts                 fractional index math

  services/                 orchestration. I/O + codec + crypto.
    boards.ts
    cards.ts
    members.ts              invite, remove, rotateBoardKey
    invitations.ts          fetch / accept / dismiss
    tracking.ts
    links.ts

  discovery/
    dedupe.ts               supersedes(), newestByDTag()     ← from calendar-sdk
    deletions.ts            NIP-09 client-side filtering     ← from calendar-sdk
    relays.ts               NIP-65 outbox resolution         ← from calendar-sdk

  runtime/
    pool.ts                 SimplePoolRuntime (default)
    localRelay.ts           LocalRelayRuntime (optional subpath export)

  adapters/
    signer.ts               toKanbanSigner() — binds prototype methods
```

The layering rule, borrowed wholesale: **`codec/` is pure and synchronous**, takes
events and returns objects or vice versa, never touches the network. Everything
testable without a relay is tested without a relay.

`crypto/`, `discovery/`, `runtime/`, `adapters/` are near-verbatim from
`calendar-sdk`. That is the concrete payoff of choosing the view-key model
(doc 04): the security-critical code is code that already ships and is already
cross-validated against a second implementation.

**Open question, deliberately deferred:** these shared modules are duplicated
rather than extracted. Extracting them into `@formstr/core` is the right end
state, but refactoring a shipped SDK before the new one exists inverts the risk.
Revisit once `kanban-sdk` is working — doc 07 tracks it.

---

## 3. Contracts

Identical in shape to `calendar-sdk/src/contracts.ts`, so **one signer object
satisfies all three formstr SDKs**:

```ts
export interface KanbanSigner {
  getPublicKey(): Promise<string>;
  signEvent(event: EventTemplate): Promise<Event>;
  nip44Encrypt(pubkey: string, plaintext: string): Promise<string>;
  nip44Decrypt(pubkey: string, ciphertext: string): Promise<string>;
}

export interface NostrRuntime {
  querySync(relays: string[], filter: Filter, timeoutMs?: number): Promise<Event[]>;
  subscribe(relays: string[], filters: Filter[],
            options?: { onEvent?: (e: Event) => void; onEose?: () => void }): SubscriptionHandle;
  publish(relays: string[], event: Event, timeoutMs?: number): Promise<void>;
  dispose?(): void;
}
```

All I/O goes through `NostrRuntime`. `SimplePoolRuntime` is the zero-config
default; `LocalRelayRuntime` (imported from `@formstr/kanban-sdk/local-relay`)
routes through the shared worker relay for offline persistence, NIP-65 outbox
routing, and a durable delivery outbox. Hosts with their own pool implement the
interface directly.

Errors are typed, not strings: `SignerRequiredError`, `ViewKeyRequiredError`,
`NotAMaintainerError`, `BoardNotFoundError`.

Without a signer the SDK still reads public boards; anything private throws
`SignerRequiredError` naming the operation.

---

## 4. API surface

```ts
const sdk = new KanbanSDK({ signer, relays?, runtime?, wrapTimestamps? });
```

| Area | Methods |
|---|---|
| Boards | `createBoard`, `updateBoard`, `fetchBoards`, `fetchBoardByCoordinate`, `deleteBoard`, `subscribeToBoard` |
| Cards | `createCard`, `updateCard`, `moveCard`, `fetchCards`, `deleteCard`, `binCard` |
| Board lists | `createBoardList`, `fetchBoardLists`, `addBoardToList`, `removeBoardFromList`, `lookupBoardViewKey` |
| Members | `invite`, `removeMember`, `rotateBoardKey`, `fetchMembers` |
| Invitations | `fetchInvitations`, `acceptInvitation`, `dismissInvitation`, `extractInvitation` |
| Tracking | `trackEvent`, `resolveTrackedStatus`, `untrack` |
| Links | `linkCards`, `unlinkCards`, `fetchIncomingLinks`, `fetchOutgoingLinks` |
| Infra | `fetchRelayLists`, `dispose`, `relays` |

Public and private go through the **same** methods:

```ts
const board = await sdk.createBoard({
  title: "Q3 Roadmap",
  columns: [{ id: "col-1", name: "To Do", order: 0 }],
  private: true,          // 32301 + view key + board list + invitations
  maintainers: [bobPubkey],
});

await sdk.createCard(board, { title: "Ship the SDK", status: "col-1" });
```

`private: false` (the default) writes `30301`/`30302` in kanbanstr's exact wire
format. The caller does not branch on privacy; the codec does.

Pure building blocks are exported too, for hosts and tests working below the
facade: `generateViewKey`, `blindedPointer`, `parseBoard`, `buildBoardTags`,
`parseCard`, `parseInvitationRumor`, `supersedes`, `newestByDTag`,
`normalizeRelayUrl`, `computeRank`.

---

## 5. Invariants the SDK enforces

Each of these is a bug someone will otherwise write. They are enforced in the SDK,
not left to callers.

**A private board is always linked into a board list.** Unlisted, its view key is
lost the moment the return value is dropped and the board is unrecoverable.
`createBoard({private:true})` links into the named list, or the first existing
one, or an auto-created "My Boards" — even when a supplied `listId` fails to
resolve.

**Card edits reuse the board's existing view key.** Minting a new key on edit
would re-encrypt the board away from every member. `updateCard` resolves the key
from the board list when the caller does not pass one.

**Strict supersession.** `created_at = max(now, previous + 1)` on every republish
of an addressable object. Doc 05 §7.

**NIP-01 tie-breaking on reads.** `supersedes()` — newest `created_at`, ties by
lowest id. Doc 03 §6.1 is the bug this exists to prevent.

**Full re-emission on edit.** Every update path builds tags by **merging into the
fetched event**, never by rebuilding from the in-memory model. This is the direct
answer to kanbanstr's two data-loss bugs (doc 03 §6.2, §6.3): unknown and
unmodelled tags survive a round trip. Tracker references and `nozap` in particular
must never vanish because the model did not know about them.

**Gift-wrap verification.** Seal signature verifies, and seal signer equals the
rumor's claimed author. Wraps failing either check are dropped before the caller
sees them. Self-sent invitations are filtered rather than shown as bogus pending
entries.

**Window-free own-board fetches.** Relays filter `since`/`until` on publish time.
Never use a time window to fetch a user's boards or cards.

---

## 6. Read path, private board

```
fetchBoardLists()                 32303 by author, self-decrypt
  └─ refs: [coordinate, relayHint, nsecViewKey, role]

for each ref:
  fetch 32301 by coordinate (relay hint first, then known relays)
  decrypt with viewKey        → title, columns, maintainers, members
  b = sha256("nip100e:v1:" + viewPubkey + ":" + coordinate)
  fetch { kinds:[32302], "#b":[b] }
  for each card:
    decrypt with board viewKey        (failure → discard)
    inner a-tag == coordinate?        (no → discard)
    author ∈ maintainers ∪ owner?     (no → discard)
  group by inner d, resolve with supersedes()
  filter NIP-09 deletions
```

Two round trips per board, both relay-side filtered. No client-side scan of a
global kind.

---

## 7. Interop

**Public boards must be byte-compatible with kanbanstr.** That includes its
non-spec conventions (doc 03 §5): assignees written to **both** `p` and `zap`
tags, `binned`, `nozap`, `t` labels, and `i` links in the `kanban:` form the repo
uses — not the `refs/link` form of `old-NIP-100.md`.

The read path must additionally tolerate **v0 legacy boards**, where columns and
description live in stringified JSON in `content` and the board carries `a` tags
listing its cards (doc 03 §7). Read-only support; the SDK does not migrate
silently.

`test/interop.test.ts` follows `calendar-sdk`'s pattern: run the SDK's output
through **ports of kanbanstr's actual parsers**, copied rather than paraphrased,
and parse kanbanstr-shaped fixtures back. Its quirks are the contract. When a wire
shape changes, that suite is what says whether the two clients just desynced.

Private boards have no interop obligation — nothing else implements them.

---

## 8. Testing

| Layer | Approach |
|---|---|
| `codec/`, `crypto/`, `discovery/` | Pure unit tests. No relay, no mocks |
| Services | Two SDK users against an in-memory relay fake, real NIP-44/NIP-59 — **no crypto is mocked** |
| Interop | Ported kanbanstr parsers, both directions (§7) |
| Runtime | `LocalRelayRuntime` against a real `RelayService` over an in-memory channel pair with fake sockets |

Scenarios that must be covered because they are where this protocol breaks:

- Two maintainers edit one card in the same second → both clients and the relay
  agree on the winner.
- Column rename → cards keep their `s` values; **tracker cards keep tracking**.
- `rotateBoardKey` → removed member's key no longer decrypts; remaining members'
  lists updated; every card re-pointered under the new `b`.
- Invitation with a forged seal signer → rejected.
- Board list lost mid-flight → `createBoard` still leaves a recoverable board.
- Card published by a non-maintainer holding the view key → not displayed.

---

## 9. Build order

1. `contracts.ts`, `kinds.ts`, `types.ts`, `runtime/pool.ts` — skeleton, no logic
2. `crypto/*` ported + `blindedPointer.ts` with its own vectors
3. `codec/board.ts`, `codec/card.ts`, **public** path only → interop suite green
   against kanbanstr fixtures
4. `services/boards.ts`, `services/cards.ts`, public. Working public SDK
5. Private variants in the same codecs + `codec/boardList.ts` + `services/members`
6. `codec/invitation.ts` + `services/invitations.ts` — the full gift-wrap flow
7. `codec/tracker.ts`, `codec/link.ts` and their services
8. `runtime/localRelay.ts` subpath
9. `rotateBoardKey`, deletion paths, `discovery/deletions.ts`

Public-first is deliberate: step 4 produces something verifiable against a live
network and a live counterpart client before any of the private machinery exists.
