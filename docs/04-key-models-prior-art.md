# 04 — Key Models for Private Multi-Party State: Prior Art and Decision

Making a Kanban board private is not primarily an encryption problem. NIP-44
solves encryption. The problem is **who holds which key, how it reaches them, and
what happens when they leave** — for an object that several people write to and
that lives at an addressable coordinate.

Five approaches exist in or around Nostr. Two of them are already in production in
this repo group.

---

## Option A — Per-object view key (NIP-52E)

**Where:** `nostr-calendar/nips/NIP-52E.md`, implemented in
`common-packages/packages/calendar-sdk`. Shipping at calendar.formstr.app.

Each private object gets a freshly generated secret key. Content is encrypted
under that key to *its own* public key — self-encryption performed by a key nobody
owns as an identity:

```ts
// calendar-sdk/src/crypto/viewKey.ts
const secret = generateSecretKey();
const ck = nip44.getConversationKey(secret, getPublicKey(secret));
content = nip44.encrypt(JSON.stringify(innerTags), ck);
```

Anyone handed `secret` reconstructs `ck` and decrypts. That is what makes it
shareable without re-encrypting per recipient.

The key travels two ways:

1. **A self-encrypted personal index** (calendar list, kind `32123`) storing
   `[coordinate, relayHint, nsecViewKey]` per object — so your own objects survive
   a page refresh.
2. **NIP-59 gift wraps** to each participant, carrying coordinate + view key.

The event itself keeps only a public `d` tag. Everything else is inside the blob.

| | |
|---|---|
| Attribution | **Preserved** — each event signed by its real author |
| Editing | Author-signed, so multi-author `d`-tag collision remains |
| Read sharing | Cheap — hand over the key, re-encrypt nothing |
| Revocation | **Expensive** — rotate key, re-encrypt and republish every object |
| Relay sees | `d` tag, author pubkey, timing |
| Maturity here | Shipped; spec written; SDK tested against a second implementation |

---

## Option B — Event-owned shared key (NIP PR #1228)

**Where:** [nostr-protocol/nips#1228](https://github.com/nostr-protocol/nips/pull/1228),
"Shared replaceables via Event-owned keys", by vitorpamplona. Open, unmerged.
Used by sheetstr.amethyst.social. Recommended *for this exact NIP* in
[PR #1665 review](https://github.com/nostr-protocol/nips/pull/1665) on 2025-01-12.

The event owns a keypair and signs itself. Editors receive the secret key,
NIP-44-encrypted to them, as the 4th element of their `p` tag:

```js
{
  pubkey: editingKeyPair.publicKey,
  kind: 3xxxx,
  tags: [
    ["d", "<identifier>"],
    ["p", "<pubkey1>", "<relay>", nip44Encrypt(editingSk, editingSk, "<pubkey1>")],
    ["p", "<pubkey2>", "<relay>", nip44Encrypt(editingSk, editingSk, "<pubkey2>")]
  ],
  content: "",
  sig: signWith(editingKeyPair.privateKey)
}
```

Optionally a **second** viewing keypair separates read from write: `content` is
encrypted from the editing key to the viewing pubkey, and view-only members get
the viewing secret in their `p` tag instead.

The proposal is candid about its failure mode:

> If any of the event's private keys are lost due to an encrypting bug or if there
> is a failure to add the ciphertext in the p-tags before signing... the event
> might become permanently unmodifiable and undecryptable

Silberengel's review adds the operational version of the same warning: confirm at
least one other member actually received the key before writing, "This failed for
us, *twice*, in the past."

| | |
|---|---|
| Attribution | **Lost** — every edit is signed by the board key. Needs an inner author field, which is unauthenticated |
| Editing | **One coordinate.** Multi-author collision gone entirely |
| Read sharing | Cheap; separate view key gives real read-only roles |
| Revocation | Viewing key rotatable; **editing key is not** ("The shared viewing key can be rotated at will but not the editing key") |
| Relay sees | `d`, all member pubkeys in `p` tags, timing |
| Maturity | Unmerged draft; would need renumbering (NIP-68 is taken by picture-first feeds) |

Note the membership leak: `p` tags list every member in plaintext. For a *private*
board that is the collaboration graph, exposed.

---

## Option C — formstr access control (A + B hybrid, in production)

**Where:** `nostr-forms/packages/formstr-app/src/nostr/accessControl.ts`.

Formstr already runs both mechanisms together. A form owns a `signingKey`
(event-owned, option B) and optionally a `viewKey` (option A), and both are
delivered by **NIP-59 gift wrap** rather than by `p` tags:

```ts
// accessControl.ts — createTag()
if (signingKey) tags.push(["EditAccess",   bytesToHex(signingKey)]);
if (viewKey)    tags.push(["ViewAccess",   bytesToHex(viewKey)]);
if (voterKey)   tags.push(["SubmitAccess", bytesToHex(voterKey)]);

// grantAccess() — those tags become an unsigned kind:18 rumor,
// sealed with the FORM's signing key, wrapped to the recipient.
const rumor = createRumor({ kind: 18, pubkey: issuerPubkey, tags: [...] }, signingKey);
const seal  = createSeal(rumor, signingKey, pubkey);
const receiverWrap = createWrap(seal, pubkey,       issuerPubkey, formId);
const senderWrap   = createWrap(seal, issuerPubkey, issuerPubkey, formId);
```

Three roles, not two — `SubmitAccess` grants response submission without read or
edit. Note also that the seal is signed by the **form's** signing key rather than
the granting human's, so a recipient authenticates the grant as coming from the
form, and that a self-addressed copy of every wrap is published so the issuer can
recover its own grant list.

```ts
// formstr-sdk/src/sdk/FormstrSDK.ts:370
const signingKey = generateSecretKey();
const formPubkey  = getPublicKey(signingKey);
const viewKey     = generateSecretKey();
const ck = nip44.v2.utils.getConversationKey(signingKey, getPublicKey(viewKey));
content = nip44.v2.encrypt(JSON.stringify([["name", name], ...rawFields]), ck);
const signed = finalizeEvent(event, signingKey);
```

Gift-wrapping instead of `p`-tagging is the significant improvement over option B
as written: **membership stops being public**. Nobody can enumerate a form's
editors from the relay.

This is the strongest option on paper, and it is worth understanding precisely why
we did not choose it below.

---

## Option D — NIP-29 relay-based groups

**Where:** `29.md` on nips master. Suggested for this NIP by fiatjaf on 2025-01-12:

> I think can't be done otherwise and should really should be done inside the
> scope of an existing closed group interface, like NIP-29 or MLS groups.

The relay enforces membership; events carry an `h` tag naming the group.

Genuinely better at the thing key-sharing schemes are worst at: **removal actually
removes**. The relay stops serving the group to an ex-member. No re-encryption,
no key rotation, no O(n) republish.

The cost is portability. A board lives on a group-capable relay and cannot be
moved elsewhere or read by a client pointed at the general relay set. The relay
becomes trusted infrastructure — it sees all content unless separately encrypted,
and it can censor. For a protocol whose stated motivation is "decentralized
project management while maintaining the platform's permissionless and
censorship-resistant nature," handing membership to one relay is a real tension.

---

## Option E — MLS (NIP-EE / Marmot)

**Where:** `EE.md` on nips master, header:

> `final` `unrecommended` `optional`
>
> **Warning** `unrecommended`: superseded by the
> [Marmot Protocol](https://github.com/marmot-protocol/marmot)

The only option here with **forward secrecy** and **post-compromise security**.
Epoch-based membership: removing a member advances the epoch and they genuinely
cannot read what follows.

Against it: the NIP is deprecated in the NIPs repo in favour of a protocol
maintained outside it; MLS group state is not addressable-event shaped, so the
whole "board is an addressable event you can link to with an `a` tag" model
disappears; and the implementation cost is an order of magnitude above the
others. It also forecloses gradual adoption — there is no "public board, plus a
private one" continuum.

---

## Comparison

| | A: view key | B: shared key | C: formstr hybrid | D: NIP-29 | E: MLS |
|---|---|---|---|---|---|
| Author attribution | ✅ real | ❌ board key | ❌ board key | ✅ real | ✅ real |
| Multi-author `d` collision | ❌ remains | ✅ solved | ✅ solved | ❌ remains | n/a |
| Membership hidden from relay | ✅ | ❌ `p` tags | ✅ gift wrap | ❌ relay knows | ✅ |
| Read-only role | ⚠️ same key | ✅ two keys | ✅ two keys | ✅ | ✅ |
| Removal effective | ❌ rotate + republish all | ❌ editing key unrotatable | ⚠️ view key only | ✅ | ✅ |
| Forward secrecy | ❌ | ❌ | ❌ | ❌ | ✅ |
| Relay requirements | none | none | none | **group-capable** | none |
| Spec status | our own draft | unmerged PR | in production, unspecced | merged NIP | deprecated |
| Code we already own | **calendar-sdk** | — | formstr-app | — | — |

---

## Decision

**Option A — per-board view key, NIP-52E parity.**

Reasons, in order of weight:

1. **It is already specified and implemented here.** `NIP-52E.md` is a written
   spec, and `calendar-sdk` implements it. `crypto/viewKey.ts`, `crypto/nip44.ts`,
   and `crypto/nip59.ts` — all three committed on `origin/calendar-sdk` and
   exercised by the committed `test/sdk.test.ts`, which drives two SDK users
   against an in-memory relay with real NIP-44/NIP-59 and no mocked crypto — move
   over essentially unchanged. Every other option means writing and debugging new
   crypto plumbing.

   Scoped honestly: `test/interop.test.ts`, the cross-app guard, was **not
   committed** when this was written (doc 08 has the full committed/uncommitted
   split), so "cross-validated against a second implementation" is a claim about
   local work, not about anything in the repository's history.

2. **Attribution matters more for Kanban than for spreadsheets.** "Who moved this
   card, who wrote this comment, who is this assigned to" is the substance of the
   artifact. Options B and C sign every edit with the board key, so authorship
   becomes an unauthenticated field inside the payload — anyone with edit access
   can write any name into it. For a spreadsheet cell that is acceptable; for a
   task board audit trail it is not.

3. **Consistency across formstr apps.** Calendar, and now Kanban, share one
   private-object model. A user's private objects all work the same way, one
   crypto path is maintained, and a future drive/docs SDK inherits it.

4. **No new relay requirements.** A private board is publishable to any relay,
   which keeps the NIP-100 property that boards are portable.

### What we accept by choosing A

- **Multi-author `d`-tag collision remains.** Cards stay author-signed. Resolution
  is NIP-100's rule, done properly: newest `created_at`, ties by lowest id
  (NIP-01), restricted to the maintainer set decrypted from the board. Doc 03 §6.1
  is exactly the bug we avoid.
- **Removal is not retroactive.** Removing a member means rotating the view key
  and re-encrypting the board and every card — O(n) republish — and the ex-member
  keeps everything they already read. The SDK exposes `rotateBoardKey()` and the
  spec states the limitation plainly rather than implying revocation works.
- **No forward secrecy.** A leaked view key exposes all past *and* future versions
  under that key. Inherited from NIP-44 and stated in doc 07.
- **Read-only vs read-write is a client convention**, not a cryptographic
  boundary: one key, and the writer set is a list inside the encrypted payload.
  Cryptographic read-only would need option B's second keypair.

### What we borrow from the options not chosen

- **From C:** deliver keys by **NIP-59 gift wrap, never by plaintext `p` tags**, so
  membership is invisible to relays. Formstr proved this out; option B as written
  leaks the member list.
- **From B:** the two-key separation is the right long-term answer for
  cryptographic read-only roles. Recorded in doc 07 as a possible v2, not built
  now.
- **From D:** relay-enforced membership is the only thing that makes removal real.
  If a formstr relay ever exists, private boards could opt into it as a second
  layer without changing the event format.

### Revisit this if

- PR #1228 merges and Amethyst/sheetstr adoption makes shared keys the ecosystem
  norm for collaborative replaceables;
- Marmot stabilizes and a board's contents are sensitive enough that forward
  secrecy outranks addressability;
- member churn turns out to be frequent in practice — rotation cost is the weak
  point of this decision, and it is the one to measure first.
