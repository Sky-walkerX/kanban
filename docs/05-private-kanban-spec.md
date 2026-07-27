# 05 — NIP-100E: Private Kanban Boards (Draft)

`draft` `optional` — working document, not submitted anywhere.

Extends [NIP-100](02-nip-100-analysis.md) (PR #1665) with a privacy layer, using
the same view-key pattern as
[NIP-52E](../../nostr-calendar/nips/NIP-52E.md). Public boards (`30301`/`30302`)
are unchanged and remain fully interoperable with kanbanstr.com.

Rationale for the key model is in [doc 04](04-key-models-prior-art.md).

---

## Abstract

A private board's title, columns, membership, and every card on it are encrypted
under a per-board **view key** — a freshly generated secret that is not anybody's
identity key. Relays store only a `d` tag and an opaque blob. The view key is
delivered to members by NIP-59 gift wrap and retained by each member in a
self-encrypted board list.

Cards are found by a **blinded board pointer**: a tag whose value is derived from
the board coordinate and its view public key, so relay-side filtering keeps
working while the relay learns nothing about which board a card belongs to.

---

## Event kinds

| Kind | Name | Type |
|---|---|---|
| `32301` | Private Kanban Board | Addressable |
| `32302` | Private Kanban Card | Addressable |
| `32303` | Private Board List | Addressable, self-encrypted |
| `32304` | Private Card Comment | Addressable |
| `1053` | Board Invitation Gift Wrap | Regular (NIP-59) |
| `53` | Board Invitation Rumor | Unsigned, inside the wrap |
| `84` | Membership Removal | Regular |
| `5` | Deletion (NIP-09) | Regular |

None of these are registered in `nips/README.md`. Ranges were checked free as of
2026-07-28: no `32xxx` kind other than `32267` is registered, and `1053` is free
between `1040` and `1059`. See [doc 07](07-gaps-risks.md) for the collision risk
and how it is managed.

`84` and the `1053`/`53` wrap-rumor pairing deliberately mirror NIP-52E so the two
formstr protocols behave identically.

---

## 1. The view key

Identical to NIP-52E §"View Key Pattern".

```
viewSecretKey = generateSecretKey()
viewPublicKey = getPublicKey(viewSecretKey)
ck            = nip44.getConversationKey(viewSecretKey, viewPublicKey)
content       = nip44.encrypt(JSON.stringify(innerTags), ck)
```

Self-encryption under a key nobody owns as an identity. Anyone holding
`viewSecretKey` derives `viewPublicKey`, reconstructs `ck`, and decrypts.

The shareable encoding is bech32 `nsec` (NIP-19), as in NIP-52E. One exception:
the blinded pointer is computed from the **hex** public key (§2).

**One view key per board.** Cards on a board are encrypted under their board's
view key — not their own. This is the deliberate difference from NIP-52E, where
each event has its own key: a board is a unit of access, and per-card keys would
mean the board's card index becomes a key store that must be rewritten on every
card creation.

---

## 2. Blinded board pointer

A private card must be fetchable by board without telling the relay which board.
Per [doc 01](01-nostr-primitives.md), only single-letter tags are queryable, and a
plaintext coordinate in one would expose the collaboration graph — who works with
whom, and how large each board is.

```
coordinate = "32301:<board-author-pubkey>:<board-d-tag>"
b          = hex(sha256(utf8("nip100e:v1:" + viewPublicKey + ":" + coordinate)))
```

`viewPublicKey` is hex, lowercase, and is never published anywhere — it is
derivable only from the view secret. So `b` is computable by every view-key holder
and by nobody else, while remaining a plain indexed tag as far as the relay is
concerned.

```jsonc
// card, as stored on a relay
{ "kind": 32302, "tags": [["d", "<card-d>"], ["b", "<64 hex chars>"]], "content": "<blob>" }

// fetching a board's cards
{ "kinds": [32302], "#b": ["<64 hex chars>"] }
```

Properties:

- The relay sees a 32-byte opaque label. It can count cards under one label and
  observe write timing, nothing more.
- An observer cannot link two boards, or link a card to a board, without the key.
- Rotating the view key changes `b`. This is consistent: rotation re-encrypts
  every card anyway, so every card is republished with the new pointer regardless.
- `b` does **not** authorize anything. It is a lookup handle. Membership is
  enforced by decrypting the board and checking the maintainer set (§7).

`b` is a novel convention; NIP-100 does not use `b` for anything, and no
widely-used NIP claims it. Any client that does not understand `32302` ignores it.

---

## 3. Private board — kind `32301`

```jsonc
{
  "kind": 32301,
  "pubkey": "<board creator hex pubkey>",
  "created_at": 1753600000,
  "tags": [["d", "<board-d-identifier>"]],
  "content": "<NIP-44 blob, encrypted under the board view key>",
  "id": "…",
  "sig": "…"
}
```

Only `d` is public. **No `alt` tag** — NIP-31's plaintext summary would leak
exactly what is being hidden.

### `d` tags MUST be opaque

The `d` tag of a private board or card is public and permanent. It MUST be a
random identifier with no relation to the object's content — a UUID (kanbanstr's
`crypto.randomUUID()`) or 16 random hex characters.

This is stated because NIP-52E's calendar **list** derives its `d` from
`sha256("<title>:<created_at>")`, which is deterministic and therefore
**guessable**: an observer who suspects a list is called "Work" can confirm it by
brute-forcing `created_at` over a plausible window. That derivation is fine for a
list whose name is generic and is retained in §5 for cross-app compatibility, but
it must not be copied to boards or cards, where the title is the secret.

### Encrypted inner tags

```jsonc
[
  ["d", "<board-d-identifier>"],
  ["title", "Q3 Roadmap"],
  ["description", "Markdown allowed"],

  ["col", "col-1", "To Do", "0"],
  ["col", "col-2", "In Progress", "1"],
  ["col", "col-3", "Done", "2"],

  ["maintainer", "<hex pubkey>"],
  ["member", "<hex pubkey>"],

  ["nozap"]
]
```

| Inner tag | Required | Meaning |
|---|---|---|
| `d` | yes | Repeated inside so the payload is self-describing after decryption |
| `title` | yes | Board name |
| `description` | no | Markdown |
| `col` | repeated | `["col", id, name, order]`, as NIP-100 |
| `maintainer` | repeated | May create and edit cards |
| `member` | repeated | Holds the view key, read-only by convention |
| `nozap` | no | Carried over from kanbanstr |

The board creator is implicitly a maintainer and need not be listed.

`maintainer`/`member` replace NIP-100's `p` tags. Two reasons: `p` would have to
be public to be useful and would leak membership, and NIP-100 overloads `p` to
mean "assignee" on cards. Distinct names remove the ambiguity.

`member` is a **client-enforced** role, not a cryptographic one — a member holds
the same key a maintainer does and can publish a card event. Honest clients
ignore cards from non-maintainers (§7). Cryptographic read-only would require a
second keypair; see doc 04 and doc 07.

---

## 4. Private card — kind `32302`

```jsonc
{
  "kind": 32302,
  "pubkey": "<card author hex pubkey>",
  "created_at": 1753600100,
  "tags": [
    ["d", "<card-d-identifier>"],
    ["b", "<blinded board pointer>"]
  ],
  "content": "<NIP-44 blob, encrypted under the BOARD's view key>",
  "id": "…",
  "sig": "…"
}
```

Cards are signed by their real author, which is the point of the view-key model
(doc 04). `pubkey` is visible to relays; content is not.

### Encrypted inner tags

```jsonc
[
  ["d", "<card-d-identifier>"],
  ["a", "32301:<board-author-pubkey>:<board-d>"],

  ["title", "Ship the SDK"],
  ["description", "Markdown allowed"],
  ["s", "col-2"],
  ["rank", "10"],

  ["u", "https://blossom.example/abc.png"],
  ["p", "<assignee hex pubkey>"],
  ["t", "backend"],

  ["i", "kanban:<board-pubkey>:<board-d>:<card-d>", "is blocked by", "blocks"],

  ["k", "1621"],
  ["e", "<tracked event id>", "<relay>"],
  ["binned"]
]
```

| Inner tag | Required | Meaning |
|---|---|---|
| `d` | yes | Repeated inside |
| `a` | yes | Full board coordinate — the authoritative board association |
| `title` | yes | |
| `description` | no | Markdown |
| `s` | no | **Column `id`**, not name. See below |
| `rank` | no | Fractional sort key within the column, ascending |
| `u` | repeated | Attachment URLs |
| `p` | repeated | Assignees |
| `t` | repeated | Labels (kanbanstr convention) |
| `i` | repeated | Card links, `["i", "kanban:pk:board:card", forwardLabel, reverseLabel]` |
| `k`, `e`, `refs/board`, `refs/card` | no | Tracker card references, as NIP-100 |
| `binned` | no | Soft delete (kanbanstr convention) |

### Two deliberate divergences from NIP-100

**`s` holds the column id.** NIP-100 stores the column *name*, so renaming a
column orphans every card in it and forces a bulk rewrite — the defect
cypherhoodlum reported in 2025 (doc 02 §3.1) and the cause of kanbanstr's
tracker-card data loss (doc 03 §6.2). Ids are stable; renaming becomes a
single-event board edit. A card whose `s` matches no column id is **unmapped**
and clients SHOULD surface it rather than hide it.

**`a` moves inside the encrypted payload.** The public `b` pointer replaces it for
lookup. `a` is retained inside so that a decrypted card is self-describing and can
be checked against the board it claims to belong to (§7).

---

## 5. Private board list — kind `32303`

A user's personal index of the private boards they can read. **Self-encrypted** to
the user's own pubkey via their signer, exactly as NIP-52E's calendar list
(`32123`) is.

```jsonc
{
  "kind": 32303,
  "pubkey": "<user hex pubkey>",
  "tags": [["d", "<list-d-tag>"]],
  "content": "<NIP-44 self-encrypted blob>"
}
```

```
encrypted = signer.nip44Encrypt(userPubkey, JSON.stringify(innerTags))
plaintext = signer.nip44Decrypt(event.pubkey, event.content)
```

### Encrypted inner tags

```jsonc
[
  ["title", "Work"],
  ["a", "32301:<board-author>:<board-d>", "wss://relay.example/", "nsec1…", "maintainer"]
]
```

Board reference layout, extending NIP-52E's three-element form with a role:

| Index | Content |
|---|---|
| 0 | `"a"` |
| 1 | Coordinate `32301:<pubkey>:<d>` |
| 2 | Relay hint (empty string if none) |
| 3 | `nsec`-encoded view secret key |
| 4 | Role as last known: `"owner"`, `"maintainer"`, or `"member"` |

`d` tag derivation follows NIP-52E: `sha256("<title>:<created_at>").slice(0, 16)`.

**A private board MUST be linked into a board list.** An unlisted private board is
unreachable the moment its view key leaves memory — there is no way to recover it
from relays. The SDK enforces this by auto-creating a default list (doc 06).

---

## 5b. Private card comment — kind `32304`

Decided 2026-07-28 (doc 07 §C1). A comment is a first-class encrypted event, not a
field inside the card — otherwise every comment republishes the whole card and two
people commenting at once lose one of each other's writes.

```jsonc
{
  "kind": 32304,
  "pubkey": "<comment author hex pubkey>",
  "tags": [
    ["d", "<comment-d-identifier>"],
    ["b", "<blinded board pointer>"]
  ],
  "content": "<NIP-44 blob, encrypted under the BOARD's view key>"
}
```

Same public shape as a card, same pointer, same key — so comments arrive on the
existing card fetch (`{"kinds":[32302,32304], "#b":[b]}`, one round trip) and reuse
the card codec's crypto path wholesale.

### Encrypted inner tags

```jsonc
[
  ["d", "<comment-d-identifier>"],
  ["a", "32301:<board-author-pubkey>:<board-d>"],
  ["e", "<card-d-identifier>"],
  ["content", "Looks good, shipping Friday"],
  ["p", "<mentioned hex pubkey>"],
  ["reply", "<parent-comment-d-identifier>"]
]
```

| Inner tag | Required | Meaning |
|---|---|---|
| `d` | yes | Repeated inside |
| `a` | yes | Board coordinate — validated exactly as for cards (§7) |
| `e` | yes | `d` identifier of the card being commented on |
| `content` | yes | Comment body, markdown allowed |
| `p` | repeated | Mentions |
| `reply` | no | Parent comment's `d`, for one-level threading |

`e` holds the card's **`d` identifier**, not an event id, because a card's event id
changes on every edit while its `d` does not.

Comments are author-signed and resolved by the same rules as cards (§7 steps 1–4),
with one difference: **any board member may comment**, not only maintainers. Step 3
therefore admits `maintainer ∪ member ∪ owner` for kind `32304`.

Editing a comment republishes it under the same `d`. Deleting is NIP-09 (§9).

Public boards use NIP-22 comments (kind `1111`) instead; there is no private
analogue to interoperate with, so `32304` has no interop obligation.

---

## 6. Invitations — kind `1053` / rumor `53`

Standard NIP-59, three layers.

**Rumor (kind `53`, unsigned):**

```jsonc
{
  "kind": 53,
  "pubkey": "<inviter hex pubkey>",
  "created_at": 1753600000,
  "tags": [
    ["a", "32301:<board-author>:<board-d>", "<relay hint>"],
    ["viewKey", "nsec1…"],
    ["role", "maintainer"]
  ],
  "content": "<optional message>"
}
```

**Seal (kind `13`)** — `nip44Encrypt(recipient, rumor)`, signed by the inviter.

**Gift wrap (kind `1053`)** — `nip44Encrypt(recipient, seal)`, signed by a fresh
ephemeral key, tagged `["p", recipient]`.

Fetch with `{"kinds":[1053], "#p":["<my pubkey>"]}`.

> **Known weakness, inherited from NIP-52E.** NIP-59 defines only `1059` and
> `21059` as wrap kinds, and the relay-side protection — "relays SHOULD only serve
> `kind 1059` events intended for the marked recipient", enforced by NIP-42 AUTH —
> is written for `1059` specifically. A custom kind `1053` gets **no such
> protection**: any client can fetch `{"kinds":[1053]}` and enumerate every
> `["p", recipient]` tag, learning who is being invited to private boards and how
> often, without decrypting anything.
>
> Using `1059` instead would fix this but makes Kanban invitations indistinguishable
> from DMs and every other wrapped payload, so clients must unwrap everything to
> find theirs — one signer round trip per DM for NIP-46 users.
>
> **Decided 2026-07-28:** keep `1053` for NIP-52E consistency, because the leak is
> identical in the shipped calendar and fixing it here alone buys nothing. The SDK
> exposes `wrapKind` as configuration so the switch is one line rather than a
> migration, and doc 07 §A6 stays open as a **joint** calendar+Kanban decision.
> `1053` is the consistent choice, not the correct one.

Recipients MUST verify, per NIP-59 and doc 01: the seal's signature verifies, and
the seal's signer equals the rumor's claimed `pubkey`. A wrap failing either check
is discarded. Without this, anyone can forge an invitation that appears to come
from a colleague — and an invitation carries a key the recipient will act on.

The `role` tag is advisory. It tells the client how to present the board; the
authoritative role is whatever the decrypted board's `maintainer`/`member` tags
say.

Accepting appends a board ref to a list (§5). Declining publishes §8.

---

## 7. Access control and conflict resolution

Nothing here is relay-enforced. All of it is client-side, and only decryptable by
key holders.

**Reading a board.** Fetch `32301` by coordinate, decrypt with the view key from
the board list or an invitation.

**Reading cards.** Compute `b` (§2), fetch `{"kinds":[32302], "#b":[b]}`, then:

1. Decrypt each card under the board view key. Discard anything that fails —
   failure means it was not written by a key holder.
2. Discard any card whose inner `a` tag does not equal this board's coordinate.
   Guards against a key holder cross-posting a card into a board it does not
   belong to.
3. Discard cards whose `event.pubkey` is neither the board author nor in the
   decrypted `maintainer` set.
4. Group by inner `d`. Resolve to one version per `d` using NIP-01 rules:
   **newest `created_at`; ties broken by lowest event id.**

Step 4 is where kanbanstr is wrong (doc 03 §6.1) and where an SDK must not be.

**Writing.** Any maintainer publishes a `32302` signed by their own key,
encrypted under the board view key, carrying `b`. Because cards are author-signed,
two maintainers editing one card produce two coordinates sharing a `d` — resolved
by step 4, the same way NIP-100 intends for public boards.

**Editing the board event** is restricted to its author, as in NIP-100. Maintainers
who need a column change ask the owner. This is a limitation, inherited: the board
is an addressable event owned by one pubkey.

**Strict supersession.** Every republish of an addressable object MUST use
`created_at = max(now, previous.created_at + 1)`. Two writes inside one second
otherwise tie and the tie-break may keep the stale version — losing an entire
edit. This applies to boards, cards, and board lists alike.

---

## 8. Membership removal — kind `84`

Published by a member opting out, or by an owner recording a removal.

```jsonc
{
  "kind": 84,
  "pubkey": "<departing or removing hex pubkey>",
  "tags": [
    ["a", "32301:<board-author>:<board-d>"],
    ["e", "<board event id>"],
    ["k", "32301"]
  ],
  "content": "<optional reason>"
}
```

This is a **notification, not a revocation.** It tells other clients to stop
showing that person as a member. It does not take the view key away.

### Real removal: key rotation

To actually cut off a removed member:

1. Generate a new view key.
2. Re-encrypt and republish the board (`32301`) under it, without the removed
   pubkey.
3. Recompute `b` and re-encrypt and republish **every card** under the new key.
4. Gift-wrap the new key to every remaining member.
5. Update the board ref in the owner's board list; every other member updates
   theirs on accepting the new key.

O(n) in cards, and cards authored by *other* maintainers cannot be re-signed by
the rotating owner — they will be republished under the rotator's pubkey, which
changes their coordinate while preserving their `d` tag and content.

That has a consequence worth spelling out. After rotation, Bob's card exists twice:
the original at `32302:bob:card7` under the old key, and the rotator's copy at
`32302:alice:card7` under the new key. When Bob next comes online and edits, he
publishes `32302:bob:card7` again under the new key — and now **two coordinates
share a `d` tag and both authors are maintainers**, so §7 step 4 resolves by
`created_at` alone. Authorship of that card flips to whoever wrote last. The card
content stays correct; the "who wrote this" answer does not.

Alternative, not chosen: leave other authors' cards encrypted under the old key
and unreadable until each author republishes. That preserves attribution and
leaves the board partially broken until every maintainer comes online. Doc 07 §B2
holds this open.

Rotation does not un-read what the removed member already read. Stated plainly:
**there is no retroactive revocation in this model.** Doc 07 tracks this.

---

## 9. Deletion

NIP-09 kind `5`, referencing both the event id (`e`) and the coordinate (`a`),
matching NIP-52E.

Relays honour deletion inconsistently, so every fetch path MUST filter deleted
objects client-side. Deleting a board additionally removes its ref from the board
list — otherwise the client keeps trying to fetch a tombstoned coordinate.

---

## 10. Relay selection

Follows NIP-52E's relay-hint rules, which exist because getting this wrong makes
invitations silently vanish (doc 01, NIP-65).

**Publishing a board or card:** fetch each member's kind `10002`, collect their
**inbox (read)** relays, publish to the union of those and the author's write
relays. The relay hint stored in board lists and invitations MUST be one that
accepted the event.

**Sending an invitation:** publish the `1053` wrap to the **recipient's inbox
relays**. Publishing to your own relays means they may never see it.

**Stale hints:** if a fetch against a stored hint fails, retry across the known
relay set before treating the board as missing, then update the stored hint.

---

## 11. Tracker cards

Unchanged from NIP-100 §"Tracker Card Event", with the reference tags moved inside
the encrypted payload.

- Tracking a **public** event (git issue `1621`/`1617`, a public card, a note) from
  a private board is fine. The tracker card is encrypted; the tracked event is
  public. The relay cannot tell that the private card mirrors it.
- Tracking a **private** card on another board requires that board's view key.
  Clients MAY store it alongside the reference in the tracking card's payload:
  `["refs/viewKey", "nsec1…"]`.

  > **This is a whole-board access grant, not a card-level one.** Because one view
  > key covers a board and all its cards (§1), embedding board B's key in a card
  > on board A gives **every member of A full read access to all of B** — its
  > title, columns, membership, and every card, not just the tracked one. It also
  > persists: revoking A's members later does not un-share B.
  >
  > Clients MUST present this as "share board B with everyone on board A", never
  > as "track this card". A safer default is to refuse cross-board tracking of
  > private cards and offer a manual status mirror instead.
- A tracker card's status derives from the tracked event's `s` tag and MUST NOT be
  changed directly (which is also why the column-rename bulk rewrite of doc 03
  §6.2 must never touch tracker cards).

---

## 12. Public/private conversion

A board is public or private for its lifetime. There is no conversion event.

- **Public → private** would be a lie: the plaintext history stays on relays
  forever. Clients MUST NOT offer it as "make this board private". Creating a new
  private board and copying cards is honest and is what the SDK does.
- **Private → public** is a fresh `30301` plus `30302` cards, with a NIP-09
  deletion of the private originals. The deletion is best-effort.

---

## 13. Security considerations

- **Key scope is the board, not the object.** This is the one place the design
  deviates from NIP-52E, where each event carries its own key, and it is a real
  increase in blast radius: one leaked key exposes a board's title, columns,
  membership list, and every card ever published under it — past and future. In
  NIP-52E a leaked key costs one meeting. The trade was made because per-card keys
  turn the board into a key store that must be rewritten on every card creation,
  serializing all maintainers behind one addressable event.
- **No forward secrecy.** Inherited from NIP-44. A leaked view key exposes every
  past and future version of the board and its cards under that key.
- **Invitation wraps are not relay-protected.** Kind `1053` is outside NIP-59's
  `1059`/`21059`, so the AUTH-gated serving rule does not apply and the `p` tags on
  invitation wraps are publicly enumerable. §6.
- **No retroactive revocation.** §8. Removal is a rotation plus a notification;
  what a member has read, they keep.
- **Onward sharing is unenforceable.** A view key can be forwarded to anyone. The
  protocol has no way to know or prevent it.
- **Metadata that remains visible to relays:** the `d` tag of every board and
  card; the author pubkey of every card; the blinded pointer `b`, allowing a relay
  to count cards under one unnamed board and watch the timing of edits; that a
  user publishes `32303` lists and receives `1053` wraps; event sizes.
- **Activity-pattern inference.** A relay watching one `b` label sees the rhythm
  of a team's work — sprint boundaries, crunch periods, who is active when, from
  the card authors' pubkeys. Encrypting content does not hide this. Clients
  handling genuinely sensitive boards should spread cards across relays or use a
  relay they control.
- **Board-list loss is total.** The board list is self-encrypted to the user's
  identity key. Lose that key, lose every private board — the view keys are stored
  nowhere else.
- **Invitation forgery.** Mitigated only by the NIP-59 seal-signer check in §6.
  Skipping it lets anyone hand a victim a board that appears to come from a
  trusted colleague.
- **Card injection by key holders.** Anyone with the view key can publish a card
  carrying the right `b`. Step 3 of §7 restricts what is *displayed* to the
  maintainer set, but the events exist on relays and count against any quota.

---

## 14. Open points

Tracked in [doc 07](07-gaps-risks.md):

- Kind numbers are unregistered and could collide.
- `b` as a tag letter is unclaimed — verified against every NIP on master — but
  unratified.
- Invitation wrap kind: `1053` (NIP-52E-consistent, unprotected) vs `1059`
  (protected, indistinguishable from DMs). Unsettled — §6.
- No cryptographic read-only role.
- Rotation is O(cards), cannot re-sign other authors' cards, and flips authorship
  on the ones it rewrites.
- Board edits remain single-owner; maintainers cannot add a column.
- Comments, reactions, and zaps on private cards are unspecified — all three are
  public-by-nature Nostr kinds and would leak the card they point at.
