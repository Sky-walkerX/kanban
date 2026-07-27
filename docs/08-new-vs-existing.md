# 08 — What Is New vs What Already Exists in formstr

The private-Kanban design was chosen partly *because* it reuses proven formstr
code (doc 04). This doc draws the line precisely: what is a copy, what is an
extension, and what is genuinely novel — because the novel parts carry all the
risk and none of the production mileage.

Reference points: `common-packages/packages/calendar-sdk` (+
`nostr-calendar/nips/NIP-52E.md`), `nostr-forms/packages/formstr-sdk` (+
`formstr-app/src/nostr/accessControl.ts`), `common-packages/packages/{signer,
local-relay}`.

---

## 1. Reused essentially unchanged

Battle-tested, cross-validated against a second implementation, shipping today.

| Thing | Lives in | Notes |
|---|---|---|
| View-key generate / encrypt / decrypt | `calendar-sdk/src/crypto/viewKey.ts` | Same self-encryption-under-a-generated-key construction |
| NIP-44 self-encrypt helpers | `calendar-sdk/src/crypto/nip44.ts` | |
| NIP-59 seal / wrap / unwrap **+ seal-signer verification** | `calendar-sdk/src/crypto/nip59.ts` | The verification step is the security-critical part |
| Self-encrypted personal index | `calendar-sdk` calendar list `32123` | Our board list `32303` is the same object with a different name |
| `[coordinate, relayHint, nsecViewKey]` ref layout | NIP-52E §4 | We append one element (§2 below) |
| Invitation gift wrap → accept → link into list | `calendar-sdk/src/services/` | Same three-step flow |
| Opt-out kind `84` | NIP-52E §6 | Same shape, different coordinate kind |
| Strict supersession (`created_at = max(now, prev+1)`) | `calendar-sdk` | |
| NIP-01 tie-break dedupe (`supersedes`, `newestByDTag`) | `calendar-sdk/src/discovery/dedupe.ts` | |
| NIP-09 client-side deletion filtering | `calendar-sdk/src/discovery/deletions.ts` | |
| NIP-65 outbox / inbox relay resolution | `calendar-sdk/src/discovery/relays.ts` | |
| `NostrRuntime` + `SimplePoolRuntime` + `LocalRelayRuntime` | `calendar-sdk/src/runtime/` | |
| Signer contract + `toXSigner()` prototype-binding adapter | `calendar-sdk/src/contracts.ts`, `adapters/signer.ts` | One signer object satisfies all formstr SDKs |
| Codec-is-pure / services-do-I/O layering | `calendar-sdk` | |
| Interop-by-ported-parsers test strategy | `calendar-sdk/test/interop.test.ts` | ⚠️ uncommitted — see caveat below |
| `tsup` + `vitest` + optional-peer subpath packaging | `calendar-sdk` | |

Also already proven, in **formstr-forms** rather than the calendar: delivering
access keys by **gift wrap instead of plaintext `p` tags** (`accessControl.ts`),
which is what keeps membership invisible. Doc 05 §6 inherits that idea, not the
code.

> **Caveat on "proven" — checked 2026-07-28.** Not all of the calendar-sdk files
> cited above are committed. On `origin/calendar-sdk` the committed set is
> `crypto/viewKey.ts`, `crypto/nip44.ts`, `crypto/nip59.ts`, `discovery/dedupe.ts`,
> `runtime/pool.ts`, `test/helpers.ts`, `test/sdk.test.ts`. Uncommitted local work
> at the time of writing: `test/interop.test.ts`, `test/booking.test.ts`,
> `src/runtime/localRelay.ts`, `src/local-relay.ts`, `src/codec/schedulingPage.ts`,
> `src/codec/slots.ts`. `packages/calendar-sdk` does not exist on `origin/main` at
> all.
>
> The doc 04 decision rests entirely on the committed set, so it stands. But the
> interop-by-ported-parsers **strategy** is a good idea we are adopting, not a
> technique with production mileage — Plan 1 Task 10 is its first committed use.

---

## 2. Extensions of existing patterns

Recognisable, but changed enough to need their own tests.

| Extension | Base | What changed |
|---|---|---|
| **Board-scoped key** | NIP-52E: one key per event | One key now covers a board **and every card on it**. Changes the unit of access from object to collection — and the blast radius with it (doc 07 §D5) |
| **Role in the list ref** | 3-element ref | 4th index carries `owner`/`maintainer`/`member` |
| **Membership inside the payload** | Calendar `p` tags inside encrypted content | Split into `maintainer` vs `member` with distinct permissions; formstr has roles (`EditAccess`/`ViewAccess`/`SubmitAccess`) but enforces them by *which key you hold*, not by a list |
| **Opaque `d` mandate** | NIP-52E derives list `d` from `sha256(title:created_at)` | Deliberately **not** copied for boards/cards — that derivation is brute-forceable when the title is the secret (doc 05 §3) |
| **Merge-into-fetched-event on edit** | calendar-sdk preserves unknown fields in places | Promoted to a hard invariant, because kanbanstr's two data-loss bugs are exactly this (doc 03 §6.2, §6.3) |

---

## 3. Genuinely new — nothing like it in any formstr repo

This is the list that needs the most scrutiny and the most tests.

### 3.1 Blinded board pointer (`b` tag) — **new crypto convention**

```
b = hex(sha256("nip100e:v1:" + viewPublicKey + ":" + coordinate))
```

No formstr protocol has needed a *relay-queryable pointer to an encrypted parent*.
The calendar never does parent-scoped queries: private events are found by
coordinate, one at a time, from refs the user already holds. Kanban must ask a
relay "all cards on this board" because cards are written by several people, so a
personal index cannot be authoritative.

Nothing in Nostr or formstr does this. Verified `b` is claimed by no NIP on
master. **Highest-novelty, highest-risk item in the design.**

### 3.2 Multi-author writes under one shared key — **new access pattern**

Every private object in formstr today has exactly one writer:

- Private calendar event → only the organizer edits; participants only RSVP.
- Formstr form → one signing key; "multi-editor" means *sharing that key*, so all
  edits are signed by the form.

Private Kanban is the first formstr object where **N distinct identities each sign
their own events, encrypted under one shared key**, and a reader must reconcile
them. Everything downstream — maintainer filtering, `d`-tag collision, rotation
attribution flipping (doc 05 §8) — follows from this and has no precedent to copy.

### 3.3 Key rotation across a collection

`calendar-sdk` can rotate one event's key (`updateEvent` updates the list ref).
Rotating a board means re-encrypting the board **and every card**, re-deriving `b`,
re-wrapping to every remaining member, and confronting cards signed by people who
are not you. No formstr code does any of that. Doc 07 §B2 still holds an open
question about its semantics.

### 3.4 Cross-object key embedding (`refs/viewKey`)

Storing board B's key inside a card on board A — a transitive, non-revocable grant
of a whole board to another board's membership. No formstr protocol shares keys
*between* encrypted objects. Doc 05 §11 recommends clients refuse it by default.

### 3.5 Domain mechanics with no formstr analogue

Not privacy-related, but all new code:

| Mechanic | Why it is new |
|---|---|
| Fractional `rank` ordering + rebalancing | No formstr object is user-orderable |
| Tracker cards | Deriving one object's state from a *foreign* event, including git-issue kinds `1621`/`1617` and status kinds `1630`–`1633` |
| Card link graph | Bidirectional labelled edges (`i` tag), queryable in the incoming direction |
| Status as column id | Fix to NIP-100's name-based status (doc 02 §3.1) |
| Columns | No formstr object has an owner-defined ordered category set |

### 3.6 Third-party interop obligation

Every existing formstr SDK is interop-tested against **another formstr client**.
`kanban-sdk` must stay byte-compatible with **kanbanstr**, a client we don't
control, that has already changed its wire format twice (doc 02 "Version drift"),
and whose quirks include writing assignees to both `p` and `zap` tags. Plus
read-tolerance for its v0 legacy format. Organizationally new.

---

## 4. Weight

Rough split of the SDK by origin:

| | Share | Risk |
|---|---|---|
| Copied from `calendar-sdk` (crypto, discovery, runtime, adapters, packaging) | ~40% | Low — shipping, cross-validated |
| Extended patterns (§2) | ~15% | Medium |
| New (§3) | ~45% | **High — no precedent, no production mileage** |

The security-critical core is in the low-risk band, which is the point of doc 04's
decision. But two of the new items are security-relevant, not merely functional:
the **blinded pointer** (§3.1) and **multi-author reconciliation** (§3.2). Those two
deserve their own threat review before implementation, not just unit tests.

Everything in §3.5 is ordinary application logic and can be built with normal care.
