# 03 — kanbanstr Implementation Review

**Source:** [vivganes/kanbanstr](https://github.com/vivganes/kanbanstr) at `bf36bd8`
(2026-04-23). Deployed at kanbanstr.com.

The only client implementing NIP-100. Everything we build must read what it
writes, so this is a compatibility study as much as a code review.

---

## 1. Shape

Svelte 5 + Vite + `@nostr-dev-kit/ndk` 2.18, ~11.4k lines under `src/`.

| File | Lines | Role |
|---|---|---|
| `src/lib/stores/kanban.ts` | 1396 | All protocol logic — fetch, parse, build, publish |
| `src/lib/components/CardDetails.svelte` | 1362 | Card editor |
| `src/lib/components/Board.svelte` | 979 | Board view, drag-and-drop, column rename |
| `src/lib/components/Card.svelte` | 751 | Card rendering |
| `src/lib/components/Column.svelte` | 739 | Column rendering |
| `src/lib/ndk/index.ts` | 735 | NDK singleton, login, zap wallet |
| `src/lib/utils/MigrationUtilV1.ts` | 197 | Legacy → current format migration |

There is no protocol layer. `kanban.ts` is a Svelte store that also parses
events, also builds events, also publishes them, also caches. Extracting an SDK
means separating those four concerns, which is most of the argument for writing
ours from scratch rather than lifting this file.

Dependencies worth noting: `@nostr-dev-kit/ndk-wallet` and `alby-js-sdk` for
zaps, `tiptap` + `marked` for markdown card descriptions.

---

## 2. Privacy: none

```
$ grep -rniE 'nip44|nip04|encrypt|decrypt|giftwrap' src/
# only matches: TypeScript `private` modifiers and NDKPrivateKeySigner
```

Every board title, column name, card title, description, assignee, and status is
plaintext on public relays. There is no private board, no encrypted field, and no
mechanism to add one. This is the entire gap the work in doc 05 fills.

Board discovery makes it worse:

```ts
// stores/kanban.ts:112
let filter: NDKFilter = { kinds: [30301], limit: 500 };
```

"All Boards" fetches every Kanban board on the configured relays, up to 500. Every
board on kanbanstr.com is world-readable by design, and the default view is a
global listing of them.

---

## 3. Relays

```ts
// ndk/index.ts:49
'wss://relay.damus.io', 'wss://nos.lol', 'wss://relay.primal.net'
```

Three hardcoded relays. **No NIP-65 handling anywhere** — no reading of kind
`10002`, no outbox routing. Maintainers on disjoint relay sets simply do not see
each other's cards, with no diagnostic.

For public boards this is a scaling annoyance. For private boards it would be
fatal: gift-wrapped keys sent to relays the recipient never reads are silently
lost. The SDK treats NIP-65 as mandatory (doc 06).

---

## 4. Read path

`loadCardsForBoard(boardId)` — `stores/kanban.ts:310`:

1. Fetch the board: `{kinds:[30301], "#d":[boardId]}` — **no `authors` filter**,
   so any stranger's board with a colliding `d` tag can be picked up; the code
   takes `Array.from(boardEvents)[0]`, i.e. whichever the relay returned first.
2. Fetch cards: `{kinds:[30302], "#a":["30301:<boardPubkey>:<boardId>"]}`.
3. Filter to `boardMaintainers ∪ boardAuthor` (line 337).
4. Dedupe by `d` (line 1246).
5. Per card with a `k` tag, fetch the tracked event, and for git issues fetch
   status events `1630`–`1633` as well.

Steps 3–4 are NIP-100's "latest by any maintainer" rule, correctly implemented in
spirit. Step 4 is where it goes wrong — see 6.1.

---

## 5. Write path

`createBoard`, `createCard`, `updateBoard`, `updateCard` all follow the same
shape: build a fresh `NDKEvent`, assign a complete `tags` array from local state,
publish.

That "complete tags array from local state" is the source of the two most serious
bugs in the codebase (6.2, 6.3): any tag the in-memory model does not represent is
**dropped on every write**.

Non-spec tags kanbanstr writes or reads:

| Tag | Used for | In NIP-100? |
|---|---|---|
| `zap` | Assignee, duplicated alongside `p` so zaps route to them | No |
| `binned` | Soft-delete flag on a card | No |
| `nozap` | Board opts out of zap-splitting | No |
| `t` | Free-form card labels | No |

Assignees are written **twice**, as `["zap", pk]` and `["p", pk]`
(`kanban.ts:737`), and read back from either (`kanban.ts:521`). Any consumer must
tolerate both.

---

## 6. Defects

### 6.1 Dedupe ignores NIP-01 tie-breaking — correctness

```ts
// stores/kanban.ts:1246
if (!existingEvent || existingEvent.created_at! < event.created_at!) {
    eventsByDTag.set(dTag[1], event);
}
```

Newest `created_at` wins; on a **tie**, whichever event the iterator reached first
wins. NIP-01 says ties break by lowest event id. Since set iteration order follows
relay response order, two maintainers editing the same card within one second can
make different clients — and the relay itself — disagree about the card's content,
non-deterministically.

Two maintainers dragging cards during the same standup is not an exotic scenario.
Our `discovery/dedupe.ts` implements `supersedes()` properly (doc 06).

### 6.2 Column rename destroys tracker cards — data loss

`Board.svelte:329` `handleRenameColumn`:

```ts
const cardsToUpdate = cards.filter(card => card.status === oldName);
for (const card of cardsToUpdate) {
    await kanbanStore.updateCard(board.id, { ...card, status: newName });
}
```

The filter does not exclude tracker cards, and `updateCard` (`kanban.ts:830`)
rebuilds `tags` as `d, title, description, alt, rank, s, binned?, u*, t*,
zap*/p*, a*, i*`. It **never re-emits `k`, `refs/board`, or `refs/card`**.

So renaming a column that contains a tracker card rewrites that card without its
tracking tags. The card stops being a tracker card permanently and silently. The
UI correctly makes tracker cards non-draggable and read-only
(`Card.svelte:336,453`) — this path bypasses that guard.

The rename loop is also serial, non-atomic, and one signature per card. Behind a
NIP-46 bunker that is one approval prompt per card; abandon halfway and the
remaining cards sit in a column that no longer exists (the UI shows them as
"unmapped", `Board.svelte:170`).

Root cause is NIP-100's decision to store status as the column **name** (doc 02
§3.1). Storing the column **id** removes the entire code path.

### 6.3 Board update drops the `nozap` flag — data loss

`loadBoards` reads `nozap` (`kanban.ts:151`) and so does
`loadBoardByPubkeyAndId` (`kanban.ts:986`); `updateBoard` (`kanban.ts:1043`)
rebuilds tags as `d, title, description, alt, col*, p*` and does not write it
back. Any board edit silently re-enables zap splitting on a board that opted out.

Same root cause as 6.2: rebuild-from-model instead of merge-into-fetched-event.
`updateBoard` even fetches the current board event first — then uses it only for
an existence check and throws the tags away.

### 6.4 N+1 fetches on card links — performance

`getOutgoingLinkedCards` (`kanban.ts:563`) issues **two** relay round trips per
link — one for the linked card's board, one for the card — inside the loop. A card
with eight links makes sixteen sequential fetches. `getIncomingLinkedCards` does
the same per incoming link. Nothing is batched or cached.

### 6.5 Unbounded board discovery — scaling

`{kinds:[30301], limit:500}` with no author filter (§2). Grows with global
adoption, not with the user.

---

## 7. Legacy migration — useful precedent

`MigrationUtilV1.ts` migrates "v0" boards, where:

- columns and description lived in **stringified JSON in `content`**, and
- the **board** listed its cards via `a` tags (board → cards),

to the current format, where columns are `col` tags and the **card** points at the
board via an `a` tag (cards → board).

`loadBoards` detects v0 by "has `a` tags **or** `content` parses to an object with
`columns`" (`kanban.ts:133`) and flags `needsMigration`. `loadCardsForLegacyBoard`
still reads the old direction.

Two things to take from this:

- The direction reversal is instructive: board-holds-card-list means every card
  add rewrites the board and serializes concurrent writers. That is exactly the
  trade-off we re-evaluated for private cards in doc 05, and we again chose
  card-points-at-board.
- Any SDK must tolerate v0 boards on relays. They exist and are not going away.

---

## 8. What we reuse

**Take the ideas:** fractional `rank` with midpoint insertion; the tracker-card
status derivation including the git-issue kind mapping (`1630`→Open,
`1631`→Resolved/Merged, `1632`→Closed, `1633`→Draft, `kanban.ts:445`); the `i`
link tag with its forward/reverse label pair; "unmapped cards" as a visible
recovery state for orphaned statuses.

**Take none of the code.** No layer separation, no tests beyond two toast-component
specs, NDK-coupled throughout while our stack is `nostr-tools` + `@formstr/signer`.

**Match the wire format exactly** for public boards, including the non-spec tags in
§5 — including writing assignees to both `p` and `zap` — or boards created in our
client render wrong in theirs.
