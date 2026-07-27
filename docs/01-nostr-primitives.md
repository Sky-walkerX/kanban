# 01 — Nostr Primitives That Kanban Stands On

Only the rules that actually constrain the Kanban design are here. This is not a
NIP summary; it is a list of the sharp edges.

---

## NIP-01 — events, kinds, tags

### Kind ranges

| Range | Behaviour |
|---|---|
| `1000`–`9999` | Regular. Relays store all of them. |
| `10000`–`19999` | Replaceable. One per `(pubkey, kind)`. |
| `20000`–`29999` | Ephemeral. Not stored. |
| `30000`–`39999` | **Addressable.** One per `(pubkey, kind, d-tag)`. |

Kanban boards (`30301`) and cards (`30302`) are addressable. Consequences that
run through every later doc:

- The identity of a board is `30301:<pubkey>:<d>`. Change the author, and it is a
  different board — which is why "any maintainer can edit a card" is awkward:
  two maintainers editing the same card produce two *different addresses* that
  merely share a `d` tag.
- Relays are permitted to discard older versions. Editing history is not
  recoverable from relays by default; a card's past states are gone.

### Replacement and tie-breaking

> in case of replaceable events with the same timestamp, the event with the
> lowest id (first in lexical order) should be retained

This is the exact rule relays apply. A client that dedupes differently can
disagree with the relay about which version is current. kanbanstr does dedupe
differently — see doc 03.

Corollary for writers: two updates inside the same second **tie**, and the
tie-break is by id, not by intent. `calendar-sdk` handles this with strict
supersession — `created_at = max(now, previous + 1)` — and the Kanban SDK must
do the same.

### Tag indexing

> As a convention, all single-letter (only english alphabet letters: a-z, A-Z)
> key tags are expected to be indexed by relays... **Only the first value in any
> given tag is indexed.**
> — `01.md:84`

Two rules, both load-bearing.

**Only single-letter tags are queryable.** Everything else — `title`,
`description`, `col`, `rank`, `refs/board`, `refs/card` — is invisible to relay
filters and can only be read after the event is fetched.

**Only `tag[1]` is indexed.** Extra elements ride along unindexed. This is what
makes NIP-100's link tag work: `["i", "kanban:pk:board:card", "is blocked by",
"blocks"]` is queryable by the coordinate in position 1 while the label pair in
positions 2–3 is invisible to the relay. Any tag design that needs two queryable
values needs two tags.

This is the load-bearing fact behind the private-card design in doc 05: if a
private card is to be fetched by board, the board reference must live in a
single-letter tag, and therefore is visible to the relay unless it is blinded.

### Filters

Filter fields AND together; values within a field OR together. `since`/`until`
match `created_at` — **publication** time, not any domain notion of time. Relevant
because a Kanban "due date" or a card's activity window can never be a relay-side
filter.

---

## NIP-09 — deletion

A kind `5` event requesting deletion of events referenced by `e` (event id) or
`a` (coordinate) tags. Relays *should* honour it; many do not, and addressable
events keep resolving after deletion because the newest version is still on disk
somewhere.

Two consequences:

- Every fetcher in an SDK must filter deleted objects client-side. `calendar-sdk`
  does this in `discovery/deletions.ts`; the Kanban SDK needs the same.
- "Delete the board" cannot mean "the data is gone". For a private board it means
  "the key index entry is removed and a tombstone is published" — anyone who
  already holds the view key keeps their copy. Doc 07 restates this as a risk.

kanbanstr has no deletion at all. It implements a `binned` tag instead — a
soft-delete flag written into the card — which is a client convention, not a NIP.

---

## NIP-44 — versioned encryption

ChaCha20 + HMAC-SHA256, keyed by an ECDH+HKDF **conversation key** between two
keypairs. From `nostr-tools`:

```ts
getConversationKey(privkeyA: Uint8Array, pubkeyB: string): Uint8Array
encrypt(plaintext: string, conversationKey: Uint8Array): string
decrypt(payload: string, conversationKey: Uint8Array): string   // verifies MAC first
```

Two usage patterns matter here.

**Self-encryption** — `getConversationKey(sk, getPublicKey(sk))`. Encrypting to
your own pubkey. Only the holder of `sk` can read it. Used for personal indexes
(the calendar list, and our board list).

**View-key encryption** — the same self-encryption, but performed under a
*generated* key rather than an identity key. Whoever is handed that generated
secret can reconstruct the conversation key and decrypt. This is what makes an
encrypted object *shareable* without re-encrypting per recipient. It is the
entire basis of doc 05.

What NIP-44 explicitly does **not** provide, from its own Limitations section:

- **No deniability** — "it is possible to prove an event was signed by a
  particular key"
- **No forward secrecy** — a compromised key decrypts all previous conversations
- **No post-compromise security** — and all future ones
- **No post-quantum security**
- **IP address leak** to relays and intermediaries
- **Date leak** — `created_at` is public, being part of the NIP-01 event

The NIP's own advice is blunt: "For high-risk situations, users should chat in
specialized E2EE messaging software and limit use of nostr to exchanging
contacts." Every one of these becomes a documented limitation of private Kanban.

Payload size is not a constraint here: the theoretical maximum plaintext is
2^32 − 1 bytes, and implementations set their own lower bound. A board with
hundreds of members or a card with a long markdown body is nowhere near any
limit.

---

## NIP-59 — gift wrap

Three layers, used to hand a key to a specific person without the relay learning
who is talking to whom:

| Layer | Kind | Signed by | Content |
|---|---|---|---|
| Rumor | app-specific | **unsigned** | the payload |
| Seal | `13` | sender's real key | `nip44Encrypt(recipient, rumor)` |
| Gift wrap | `1059` (or `21059`, ephemeral) | **ephemeral, throwaway key** | `nip44Encrypt(recipient, seal)` |

"Tags MUST always be empty in a `kind:13`. The inner event MUST always be
unsigned." The rumor is unsigned deliberately — an unsigned event cannot be
presented to a third party as proof the sender wrote it.

### The wrap kind is not a free choice

NIP-59 defines exactly two wrap kinds, `1059` and `21059`. It does **not** sanction
app-specific ones. That matters because of the relay rule attached to `1059`:

> To protect recipient metadata, relays SHOULD only serve `kind 1059` events
> intended for the marked recipient.
> — `59.md:122`; NIP-17 restates it as NIP-42 AUTH enforcement

A custom wrap kind gets none of that. `calendar-sdk` uses `1052`, `1055`, `1057`,
`1058`; NIP-52E specifies them. Those wraps are served to **anyone who asks**,
because no relay has a rule for them. The content is still encrypted, but the
`["p", recipient]` tag is public — so an observer can enumerate who is receiving
invitations, and how many, without decrypting anything.

This is inherited, not introduced, by the Kanban work. Doc 05 and doc 07 carry it
as an open risk rather than repeating it silently.

Verification rule that is easy to get wrong and security-critical: after
unwrapping, the **seal's signature must verify, and the seal's signer must equal
the rumor's claimed `pubkey`**. Skip that and anyone can forge an invitation that
appears to come from someone you trust. `calendar-sdk` enforces it; the Kanban
SDK must too.

On timestamps: "The canonical `created_at` time belongs to the `rumor`. All other
timestamps SHOULD be tweaked to thwart" timing analysis (`59.md:111`), and
expiration tags on the two outer layers should use independent random values.
`calendar-sdk` makes this a configurable `wrapTimestamps: "real" | "jittered"`
because byte-parity with the standalone app mattered more than timing privacy —
i.e. it ships the non-recommended default. The same knob carries over, and the
same criticism applies.

---

## NIP-65 — relay lists

Kind `10002`, the user's read (inbox) and write (outbox) relays.

Gift wraps are useless if published where the recipient never looks. Invitations
MUST go to the recipient's inbox relays. A private board must be published to the
union of the author's write relays and every member's inbox relays, or members
simply cannot fetch it.

This is the difference between an invitation flow that works and one that
silently fails for anyone whose relay set doesn't overlap yours.

---

## NIP-51 — lists and sets

Addressable events whose payload is a collection, with private entries supported
by NIP-44-encrypting them into `content`.

Relevant twice:

- pablof7z's review of PR #1665 argues columns should *be* NIP-51 sets rather than
  `col` tags on the board, so that groups can nest arbitrarily (org → project →
  board → column). See doc 02.
- The private **board list** in doc 05 is structurally a NIP-51-style
  self-encrypted set, exactly as `calendar-sdk`'s kind `32123` calendar list is.

---

## NIP-31 — `alt`

A human-readable plaintext summary for clients that don't understand the kind.
NIP-100 uses it on both boards and cards.

Directly hostile to private boards: an `alt` tag on an encrypted board would leak
the very thing being hidden. Private kinds in doc 05 therefore carry **no** `alt`.

---

## NIP-34 — git stuff

Source of two things NIP-100 borrows:

- The `refs/…` tag convention, which NIP-100 reuses as `refs/board` / `refs/card`
  for tracker cards. Note these are multi-character and therefore **not
  queryable** — a tracker card cannot be found by what it tracks.
- Issue kinds `1621`/`1617` and status kinds `1630`–`1633`, which kanbanstr reads
  to auto-move tracker cards. Status mapping is in doc 03.

---

## NIP-EE / MLS — status as of this writing

`EE.md` on nips master carries this header:

> `final` `unrecommended` `optional`
>
> **Warning** `unrecommended`: superseded by the
> [Marmot Protocol](https://github.com/marmot-protocol/marmot)

MLS is the only option surveyed that offers forward secrecy and post-compromise
security. But the NIP itself is deprecated in favour of a protocol maintained
outside the NIPs repo, and MLS state is not addressable-event shaped. Doc 04
records why this was not chosen and what it would buy if revisited.

---

## NIP-29 — relay-based groups

Membership and authorization enforced **by the relay**, with an `h` tag naming the
group. fiatjaf's review comment argues Kanban "should really be done inside the
scope of an existing closed group interface, like NIP-29 or MLS groups."

The objection to it is not technical quality but portability: it requires a
group-capable relay, so a board cannot be moved to arbitrary relays or read by a
client pointed at the general relay set. Doc 04 has the full comparison.

---

## What this leaves us

1. Addressable events give free "latest version wins" but no multi-author story.
2. Only single-letter tags are queryable — everything a client needs to filter on
   must be one, and is therefore visible to relays unless deliberately blinded.
3. NIP-44 gives shareable encryption with no forward secrecy; NIP-59 gives private
   key delivery; NIP-65 decides whether delivery works at all.
4. Deletion is advisory, so "revoke" is always weaker than it sounds.
