# 07 — Gaps, Risks, and Open Questions

Everything known to be unresolved. Grouped by what it threatens. Each entry says
what would settle it.

---

## A. Protocol-level, inherited from NIP-100

### A1. Kinds are unregistered — collision risk

`30301` and `30302` appear nowhere in the kind registry on `nips` master. PR #1665
has been open since 2025-01-01 with every architectural objection unresolved
(doc 02 §7). `30311`–`30315` next door belong to NIP-53, so the neighbourhood is
live.

Our private kinds `32301`/`32302`/`32303`/`1053`/`53` are our own invention.
Ranges were verified free on 2026-07-28 — `32267` is the only registered `32xxx`,
and `1053` sits in a gap between `1040` and `1059` — but "free today" is not
"reserved".

**Settle by:** keeping every kind in one `kinds.ts` with the reasoning inline, so
a renumber is a one-file change; opening a PR for the private profile once it
runs.

### A2. `s` stores column name, not id

Doc 02 §3.1, doc 03 §6.2. Causes real data loss in the only shipping client.
Fixed in the private profile (doc 05 §4); **unfixed for public boards**, where we
match kanbanstr rather than diverge.

**Settle by:** a comment on PR #1665 with the tracker-card data-loss trace from
doc 03 §6.2 as evidence. This is the cheapest concrete contribution available to
us.

### A3. Multi-author `d`-tag collision

Two maintainers editing one card produce two addressable coordinates sharing a
`d` tag. We keep it deliberately (doc 04) and resolve properly with NIP-01
tie-breaking. Consequences we accept:

- A removed maintainer's old card version can still win a resolution if it happens
  to be the newest for that `d`.
- Card fetch requires a client-side join against the board's maintainer set.
- On **public** boards anyone can publish a `30302` carrying your board's `a` tag,
  so the fetch is unbounded before filtering. On private boards the blinded
  pointer means only key holders can do this.

### A4. Board edits are single-owner

Only the board author can change `32301`/`30301`, so maintainers cannot add or
rename a column. Inherited from addressable-event ownership; the only clean fix is
the shared-signing-key model we declined (doc 04). dluvian raised the same shape
of complaint in 2025 and it was never answered.

**Settle by:** measuring whether it hurts in practice. If it does, a
maintainer-proposes/owner-applies flow, or reconsider option B.

### A6. Custom gift-wrap kinds lose relay protection — **inherited defect**

NIP-59 defines `1059` and `21059`. NIP-59 §"Recipient metadata" and NIP-17 both
say relays SHOULD serve `kind 1059` **only to the p-tagged recipient**, enforced
by NIP-42 AUTH.

NIP-52E and `calendar-sdk` use `1052`, `1055`, `1057`, `1058`. Doc 05 proposes
`1053`. None of these get the rule. Anyone can subscribe `{"kinds":[1052]}` today
and enumerate every pubkey receiving a private calendar invitation on any relay
serving them — count, timing, recipient set — without decrypting anything.

**This is a live issue in a shipped formstr product, not just a Kanban design
question.** Whatever is decided for Kanban should probably be decided for the
calendar at the same time.

Options: switch to `1059` and unwrap-everything on the client; keep custom kinds
and accept the leak; keep custom kinds and lobby for relays to generalize the
rule.

**Decided 2026-07-28 — partially.** Kanban ships `1053` with a `wrapKind` config
option, so switching is one line. The underlying question is **not** closed: it
should be answered for `calendar-sdk` and `kanban-sdk` together, because the leak
is live in the calendar today and a Kanban-only fix changes nothing for users.
Still question 0 in §E.

### A5. `rank` has no rebalancing

Fractional indexing with midpoint insertion degrades on repeated insertion at the
same position until float precision runs out. Not specified anywhere; kanbanstr
does not handle it.

**Settle by:** detecting gap exhaustion in `codec/rank.ts` and republishing the
column's cards with integer-spaced ranks. Cheap; just needs writing.

---

## B. Private-model limitations

These follow from choosing view keys (doc 04). None is a bug; all must be stated
in user-facing terms, because a user who thinks "private" means "revocable" will
be wrong in a way that matters.

### B1. No retroactive revocation

Removing a member rotates the key going forward. What they already read, they
keep — including any local copy. Doc 05 §8.

### B2. Rotation is O(cards), and cross-author

Rotating re-encrypts the board and **every card**. Cards authored by other
maintainers cannot be re-signed by the rotating owner, so they get republished
under the rotator's pubkey — same `d`, same content, different coordinate.
Attribution of those cards is quietly rewritten by the rotation.

**Open question:** should rotation instead leave other authors' cards unreadable
under the new key until each author republishes their own? That preserves
attribution and breaks the board until everyone comes online. Neither answer is
good. Needs a decision before `rotateBoardKey` ships.

### B3. No cryptographic read-only

`member` vs `maintainer` is client-enforced; both hold the same key. A "read-only"
member can publish a card that honest clients will not display but that exists on
relays. Option B's second keypair (doc 04) is the real fix, and is a plausible v2
that would not change the board or card format — only the key distribution.

### B4. No forward secrecy

NIP-44 property. A leaked view key exposes all past and future versions under that
key. Only MLS/Marmot fixes this, at the cost of the entire addressable-event
model.

### B5. Board-list loss is total

View keys live only in the self-encrypted `32303` list. Lose the identity key,
lose every private board irrecoverably. Same failure mode as NIP-52E calendars, so
at least it is a familiar one.

**Mitigation to consider:** an explicit "export board keys" path so a user can
back up `nsec` view keys out of band.

### B6. Metadata that stays visible

Doc 05 §13. The relay sees `d` tags, card author pubkeys, blinded pointers, event
sizes, and timing. A relay watching one `b` label learns a team's working rhythm
and roster of active authors without decrypting anything.

**Open question:** is per-card `b` randomization worth it — e.g. `b` derived per
epoch so the label rotates weekly? It breaks the single-filter fetch and adds a
key schedule. Probably not worth it; recorded so it is a decision, not an
oversight.

---

## C. Unspecified surface

### C1. Comments, reactions, zaps on private cards — **partly resolved**

**Decided 2026-07-28:** comments get their own encrypted kind `32304`, specified in
doc 05 §5b — same blinded pointer, same board key, fetched on the same round trip
as cards. Any board member may comment, not only maintainers.

**Still open: reactions and zaps.** Both are public kinds (`7`, `9735`) carrying an
`e`/`a` tag at their target, so either one on a private card announces the card's
existence and the reactor's interest. Zaps additionally need a public recipient. No
good answer yet; the likely outcome is that the SDK refuses both on private boards
and the UI says why.

The original framing follows.

#### Original analysis

NIP-100 says "any user can react, comment, zap on board and cards". Every one of
those is a public kind carrying an `e` or `a` tag pointing at its target. On a
private card that **leaks the card's existence and the reactor's interest in it**,
which is most of what encryption was protecting.

Nothing in doc 05 addresses this. Options, none chosen:

- Encrypted comment kind under the board view key, `b`-pointered like cards.
- Comments as inner tags on the card, rewritten by whoever comments — inherits
  A3's collision problem per comment.
- Simply forbid reactions and zaps on private cards, and say so in the UI.

**This is the largest hole in doc 05.** Kanban without comments is not a
finished product.

### C2. Attachments

`u` tags are bare URLs. On a private board the URL is inside the encrypted
payload, but the **file at that URL is not encrypted** and typically sits on a
public Blossom/NIP-96 server. vitorpamplona's `imeta` suggestion (doc 02 §7) is
also unadopted, so there is no hash, size, or blurhash.

**Settle by:** deciding whether the SDK encrypts uploads under the board view key
before upload. Probably yes for private boards; it is a real gap today.

### C3. Notifications

An assignee learns they were assigned only by polling the board. There is no
notification event. For public boards the `p` tag makes a generic client show a
mention; for private boards the `p` tag is encrypted, so nothing does.

### C4. Board discovery for public boards

kanbanstr's `{kinds:[30301], limit:500}` does not scale (doc 03 §6.5). We inherit
the problem for any "browse public boards" feature. A NIP-51 board set per user,
or a `t`-tag convention, would fix it. Out of scope for v1.

---

## D. Implementation risks

### D1. Duplicated crypto between SDKs

`crypto/*`, `discovery/*`, `runtime/*` are copied from `calendar-sdk` rather than
shared (doc 06 §2). Two copies of security-critical code drift. Accepted
deliberately to avoid refactoring a shipped SDK before the new one exists.

**Settle by:** extracting into `@formstr/core` once `kanban-sdk` is working, in
one PR that moves both consumers at once.

### D2. Interop is a moving target

kanbanstr is actively developed and its wire format has already drifted twice
(doc 02 "Version drift"). The ported-parser interop suite (doc 06 §7) detects
drift only when we re-port.

**Settle by:** pinning the kanbanstr commit the fixtures came from in the test
file, and re-checking it deliberately rather than incidentally.

### D3. NIP-46 signing volume

The view-key model helps here, but less than a first reading suggests. Signer
operations are needed for:

| Operation | Signer calls |
|---|---|
| Decrypt board list | 1 per list |
| Decrypt board + all its cards | **0** — view key is local |
| Sign a card edit | 1 |
| Unwrap invitations | **1 per gift wrap**, unavoidable — the seal is encrypted to the identity key |

So loading 50 cards is zero signer round trips, which is the real win over
per-recipient encryption and must not be lost by accidentally routing card
decryption through the signer. But a user with 40 pending invitation wraps still
pays 40 remote `nip44Decrypt` calls at startup behind a NIP-46 bunker, and there
is no way around that — it is how NIP-59 works.

**Mitigation:** cache unwrap results by wrap id, and never re-unwrap a wrap that
was already accepted or dismissed.

### D5. Board key scope is a blast-radius increase over NIP-52E

Doc 05 §13. One key per **board** rather than per event means a single leak costs
the whole board's history. The calendar's per-event keys contain damage to one
meeting. Deliberate, but it is the design's largest single security regression
against its own template, and it should be named as such in any user-facing
security note.

### D4. `b` tag letter is unclaimed but unratified

No widely-used NIP claims `b`. If one does later, private cards need a new letter
and every card must be republished. Low probability, high blast radius, zero cost
to note.

---

## E. Questions for the next session

0. **A6 — invitation wrap kind.** `1053` (NIP-52E-consistent, publicly
   enumerable `p` tags) vs `1059` (relay-AUTH-protected, unwrap-everything on the
   client). Affects the shipped calendar too, so it is the one question here with
   consequences outside this project.
1. **C1 — comments on private cards.** The biggest unspecified surface. Needs a
   decision before the SDK's API is considered stable.
2. **B2 — rotation semantics.** Rewrite other authors' cards, or leave them broken
   until they republish?
3. **C2 — attachment encryption.** Does the SDK encrypt uploads for private
   boards, or is that the host app's job?
4. **Public-board scope.** Is public-board support a first-class SDK goal, or only
   a compatibility read path? Doc 06 §9 assumes first-class; it is the largest
   single lever on scope.
5. **Upstream contribution.** Do we comment on PR #1665 with the A2/A3 findings,
   or ship first and propose from a working implementation? The second is more
   persuasive and slower.
