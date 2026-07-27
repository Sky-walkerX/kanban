# 02 — NIP-100 (PR #1665) Analysis

**Source:** [nostr-protocol/nips#1665](https://github.com/nostr-protocol/nips/pull/1665),
"Multi-User Kanban boards on Nostr", by `vivganes`. Opened 2025-01-01, last
updated 2026-03-04, 8 commits, 1 file (`100.md`). **Open and unmerged after 19
months.**

Three versions of this document exist and they are not the same. See
[Version drift](#version-drift).

---

## 1. Status

| Fact | Value |
|---|---|
| Merged | No |
| Kinds registered in `nips/README.md` | **No** — `30301` and `30302` appear nowhere in the kind registry on master |
| Neighbours in registry | `30311` Live Event, `30312` Interactive Room, `30313` Conference Event (NIP-53), `30315` User Statuses |
| Review comments | 19, from pablof7z, fiatjaf, vitorpamplona, dluvian, Silberengel, auggie-lahey, cypherhoodlum |
| Unaddressed objections | Most of them. See [Open objections](#open-objections). |

The unregistered kinds are a real, if unglamorous, risk: nothing stops another
draft from claiming `30301`, and `30302` sits close enough to the NIP-53 block
that a future extension could collide. Doc 07 tracks this.

---

## 2. Board event — kind `30301`

```jsonc
{
  "kind": 30301,
  "tags": [
    ["d", "<board-d-identifier>"],
    ["title", "Board Name"],
    ["description", "Board Description"],       // markdown allowed
    ["alt", "A board to track my work"],        // NIP-31

    ["col", "col1-id", "To Do", "0"],           // ["col", id, name, order]
    ["col", "col2-id", "In Progress", "1"],
    ["col", "col3-id", "Done", "2"],

    ["p", "<maintainer pubkey>"],               // repeated
  ]
}
```

Required: `d`, `title`.

**Rules as written.**

- "In case there are no `p` tags to designate maintainers, the owner of the board
  is the only person who can publish cards."
- "Editing the board event is possible only by the creator of the board."

**Observations.**

- Columns carry an `id` *and* a `name` *and* an `order`. Only the board author can
  ever change them, so column identity is stable — but see the `s`-tag problem
  below, which throws that stability away.
- `col` is not a single-letter tag, so columns are not queryable. Fine — you have
  to fetch the board anyway.
- `p` here means "maintainer". On the card event `p` means "assignee". Same tag,
  two different meanings depending on parent kind. Any generic Nostr client that
  renders `p` tags as mentions will notify assignees and maintainers
  indiscriminately.
- Nothing distinguishes a maintainer from an ordinary `p` mention, so a board
  cannot mention a user without granting them write access.

---

## 3. Card event — kind `30302`

```jsonc
{
  "kind": 30302,
  "tags": [
    ["d", "<card-d-identifier>"],
    ["title", "Card Title"],
    ["description", "Card Description"],
    ["alt", "A card representing a task"],
    ["s", "To do"],                              // status == column NAME
    ["rank", "10"],                              // ascending sort within column

    ["u", "https://attachment1"],                // attachments

    ["p", "<assignee pubkey>"],                  // assignees

    ["a", "30301:<board-pubkey>:<board-d>", "<relay>"]
  ]
}
```

Required: `d`, `title`, `a`.

**Rules as written.**

- Editing = republish with the same `d`.
- "When a client gets multiple card events with the same `d` tag, it takes the
  latest one by any maintainer or the creator of the board event as the source of
  truth."

### 3.1 The `s`-tag problem — unresolved

`s` holds the column **name**, not the column **id**, even though the board
defines ids. cypherhoodlum raised this on 2025-02-14 while implementing Nestr:

> letting the user rename the kanban board columns results in losing the ability
> to track all cards in the renamed column. Shouldn't the `s` tag of the card
> event be the column uuid instead of title of the column?

The spec was never changed. kanbanstr's answer was to implement column rename as
a **bulk rewrite of every card in that column** (doc 03), which:

- requires write access to every card, including cards authored by other
  maintainers;
- is not atomic — a partial failure strands cards in a column that no longer
  exists;
- is impossible for tracker cards, whose status is derived from the tracked event.

This is the single clearest defect in the current spec.

### 3.2 The multi-author addressing problem — unresolved

A card edited by Alice lives at `30302:alice:card7`. The same card edited by Bob
lives at `30302:bob:card7`. They are **different addressable events**. The spec
papers over this with "takes the latest one by any maintainer", which means every
client must:

1. fetch all `30302` events with `#a` = the board,
2. fetch the board to learn the maintainer set,
3. discard events by non-maintainers,
4. group by `d`, keep newest.

Costs: unbounded fetch (any stranger can publish a `30302` with your board's `a`
tag), a client-side join, and no protection against a removed maintainer's old
edits resurfacing if their event happens to be newest.

vitorpamplona's answer (2025-01-12) was to abandon per-author addressing entirely:

> If you are looking for a way to get rid of the multi-author d-tag shenanigans,
> I recommend [PR #1228]. Basically, boards would be always created by a new nsec
> that is encrypted and shared with all the p-tagged members individually.

The author's reply — "let me explore how to adapt this NIP to use this" — was the
last word. No adaptation was made. Doc 04 evaluates that option properly.

### 3.3 `rank`

A float-ish string; cards sort ascending. kanbanstr computes midpoints between
neighbours on drag (`(prev + next) / 2`). Standard fractional indexing, and it
degrades the standard way: repeated insertion at the same position halves the gap
each time and eventually exhausts float precision. No rebalancing is specified.

---

## 4. Tracker cards

A `30302` whose `k` tag names the kind it mirrors. "Any `30302` event with a `k`
tag will be treated as a tracker card."

| Target | Reference tags |
|---|---|
| Regular event (e.g. kind `1`) | `["k","1"]`, `["e","<id>","<relay>"]` |
| Addressable event | `["k","<kind>"]`, `["a", …]` |
| Another Kanban card | `["k","30302"]`, `["refs/board","30301:pk:d"]`, `["refs/card","<card-d>"]` |

The third form exists because `a` is already taken by the board association, so
NIP-100 borrows NIP-34's `refs/…` convention. Note both `refs/board` and
`refs/card` are multi-character and therefore **not indexed** — you cannot ask a
relay "which tracker cards point at this card", you can only fetch and filter.

### Automatic movement

> In case of tracked card, its status is deemed to be the `s` tag value of the
> event it tracks.

This is the most interesting idea in the NIP. An executive board tracks cards
from team boards, and they move by themselves as the teams work. The author
defends it explicitly (2025-02-15) as the alternative to micromanagement.

It also means a tracker card's status is **not** under the tracking board's
control, so a tracker card cannot be dragged between columns — and if the tracked
board renames a column (§3.1), the tracker card lands in a column the tracking
board may not even have.

---

## 5. Card-to-card links

Present in the kanbanstr repo's `NIP-100.md`, **absent from the PR**.

```jsonc
["i", "kanban:<board-pubkey>:<board-d>:<card-d>", "is blocked by", "blocks"]
```

Forward and reverse labels in one tag, so a single event describes both directions
of the relationship. `i` *is* single-letter, so incoming links are queryable:
`{"kinds":[30302], "#i":["kanban:pk:board:card"]}`. That is a genuinely nice
design and it is not in the proposal under review.

`i` in NIP-73 means "external content id" with a scheme prefix. `kanban:` as a
scheme is consistent with that spirit, though not registered.

---

## 6. Access control, as specified

| Actor | May |
|---|---|
| Board creator | Modify the board event |
| Maintainers | Add cards; edit any card including status |
| Anyone | View, react, comment, zap |

Enforcement is **client-side only**. Relays neither know nor care about the
maintainer list. A malicious client publishes whatever it wants; the protection
is that honest clients ignore non-maintainer events.

Two omissions:

- No way to grant "can add cards" without "can edit everyone's cards".
- No way to remove a maintainer retroactively — their historical card versions
  remain valid-looking, and if one is the newest for a `d` tag it wins.

---

## 7. Open objections

| Raised by | Date | Objection | Status |
|---|---|---|---|
| pablof7z | 2025-01-01 | Columns should be NIP-51 sets, recursively nestable (org → project → board → column), decoupling storage from Kanban presentation | Never addressed |
| dluvian / Silberengel | 2025-01-02 | Arbitrary events (git issues) should be usable as cards directly, without a wrapper | Partly addressed via tracker cards; dluvian maintained wrapping "adds complexity and increases the likelihood of errors" |
| dluvian | 2025-01-02 | Board owner should be able to move a card on their own board without owning it | Never addressed |
| vitorpamplona | 2025-01-05 | Move stringified-JSON content into tags; use `imeta` for attachments (blurhash, size, hash) instead of bare `u` | `u` retained; `imeta` not adopted |
| auggie-lahey | 2025-01-08 | No nested cards (epic → feature → story → task) | Deferred by author; partly served by `i` links, which are not in the PR |
| vitorpamplona | 2025-01-12 | Adopt PR #1228 event-owned keys to eliminate multi-author `d`-tag collisions | Acknowledged enthusiastically, never implemented |
| Silberengel | 2025-01-12 | If adopting shared keys, confirm at least one member received the key before writing — they lost projects to this twice | Moot while #1228 unadopted |
| fiatjaf | 2025-01-12 | This belongs inside NIP-29 or MLS groups, not as a standalone scheme | Never addressed |
| cypherhoodlum | 2025-02-14 | `s` should be column id, not name | Never addressed |
| cypherhoodlum | 2025-02-14 | Wants an `r` tag linking a board to a git repository | Not adopted |

Every architectural objection is open. The PR's 8 commits added tracker cards and
the maintainer list; nothing else moved.

---

## Version drift

Three documents, all called NIP-100:

| Where | Card links |
|---|---|
| PR #1665 head (`100.md`, 196 lines) | **absent** |
| `kanbanstr/old-NIP-100.md` | present, using `refs/link` |
| `kanbanstr/NIP-100.md` (222 lines) | present, using `i` with a `kanban:` prefix |

So the reference implementation ships a feature its own proposal does not
describe, and the proposal under community review is the most out-of-date of the
three. Anyone implementing from the PR alone produces a client that cannot read
kanbanstr's links.

---

## What we take forward

**Keep**: kinds `30301`/`30302` and the public wire format as-is (interop with
kanbanstr.com is worth more than fixing it unilaterally). Keep tracker cards —
auto-movement is the best idea here. Keep `i` links, including the label pair.

**Fix in the private profile** (doc 05): status by column **id**; explicit
membership with separate read and write roles; a board reference that is
queryable without leaking.

**Do not fix unilaterally**: multi-author addressing on public boards. Changing it
means abandoning kanbanstr interop, and the view-key model we chose keeps
per-author signing deliberately. Doc 04 explains the trade.
