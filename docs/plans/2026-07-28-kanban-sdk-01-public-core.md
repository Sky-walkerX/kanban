# Kanban SDK — Plan 1: Public NIP-100 Core

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship `@formstr/kanban-sdk` with full read/write support for public NIP-100 Kanban boards, byte-compatible with kanbanstr.com.

**Architecture:** A new package in the `common-packages` pnpm workspace, shaped exactly like `packages/calendar-sdk`: pure synchronous codecs that convert Nostr events to domain objects and back, services that combine codecs with I/O, and all network access behind one `NostrRuntime` interface with a `SimplePool`-backed default. No NDK. This plan builds only the public path — the private/encrypted path is Plan 2.

**Tech Stack:** TypeScript 5.6 (strict), `nostr-tools` ^2.23.3, `@noble/hashes` ^1.8.0, `tsup` (ESM + CJS + d.ts), `vitest`, pnpm workspace.

## Global Constraints

- Package name: `@formstr/kanban-sdk`, version `0.1.0`, `"type": "module"`, MIT.
- Location: `common-packages/packages/kanban-sdk/`. The workspace already globs `packages/*` — no `pnpm-workspace.yaml` edit needed.
- `tsconfig.json` extends `../../tsconfig.base.json`. Strict mode is on; `noImplicitReturns` and `noFallthroughCasesInSwitch` are on.
- Build target `es2022`. Test environment `node`.
- Runtime dependencies are **only** `nostr-tools` and `@noble/hashes`. No `rrule` (calendar-only). No NDK, ever.
- Everything under `src/codec/` MUST be pure and synchronous: no network, no `Date.now()` in parsers, no signer. Services own all I/O.
- Wire format for public boards/cards MUST match kanbanstr at commit `bf36bd8`, **including** its non-spec conventions: assignees written to both `p` and `zap` tags, plus `binned`, `nozap`, and `t` tags. See `kanban/docs/03-kanbanstr-review.md` §5.
- Every republish of an addressable event MUST use `created_at = max(now, previous + 1)` (strict supersession).
- Every edit path MUST merge into the fetched event's tags, never rebuild from the in-memory model. This is the direct fix for the two data-loss bugs in `kanban/docs/03-kanbanstr-review.md` §6.2 and §6.3.
- Test files live next to their source as `*.test.ts`, except the interop suite which lives in `test/`.
- Every commit message ends with the footer:
  ```
  Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>
  ```
  Commit commands below omit it for brevity; add it to each.
- Run all commands from `common-packages/packages/kanban-sdk/` unless stated otherwise.

## Reference docs

- Protocol being implemented: `kanban/docs/02-nip-100-analysis.md`
- Behaviour to match and bugs to avoid: `kanban/docs/03-kanbanstr-review.md`
- Target architecture: `kanban/docs/06-sdk-architecture.md`
- Template to copy from: `common-packages/packages/calendar-sdk/`

## File structure

| File | Responsibility |
|---|---|
| `src/kinds.ts` | Every kind number the SDK knows, with reasoning inline |
| `src/types.ts` | Domain objects: `KanbanBoard`, `KanbanCard`, `Column`, `CardLink`, drafts |
| `src/contracts.ts` | `KanbanSigner`, `NostrRuntime`, `KanbanCtx`, typed errors |
| `src/runtime/pool.ts` | `SimplePoolRuntime` — the default `NostrRuntime` |
| `src/discovery/dedupe.ts` | NIP-01 resolution: `supersedes`, `newestByDTag`, `nextCreatedAt` |
| `src/discovery/deletions.ts` | NIP-09 client-side deletion filtering |
| `src/discovery/relays.ts` | NIP-65 relay-list resolution and URL normalization |
| `src/codec/rank.ts` | Fractional ordering: `computeRank`, `needsRebalance`, `rebalance` |
| `src/codec/board.ts` | Public board `30301` parse/build/merge, plus v0 legacy read |
| `src/codec/card.ts` | Public card `30302` parse/build/merge |
| `src/services/boards.ts` | Board create/update/fetch orchestration |
| `src/services/cards.ts` | Card create/update/move/fetch orchestration |
| `src/KanbanSDK.ts` | Public facade |
| `src/index.ts` | Barrel export |
| `test/helpers.ts` | `FakeRuntime` in-memory relay, signer helpers |
| `test/interop.test.ts` | Ported kanbanstr parsers, both directions |

---

### Task 1: Package scaffold, kinds, types, contracts, runtime

**Files:**
- Create: `common-packages/packages/kanban-sdk/package.json`
- Create: `common-packages/packages/kanban-sdk/tsconfig.json`
- Create: `common-packages/packages/kanban-sdk/tsup.config.ts`
- Create: `common-packages/packages/kanban-sdk/vitest.config.ts`
- Create: `common-packages/packages/kanban-sdk/src/kinds.ts`
- Create: `common-packages/packages/kanban-sdk/src/types.ts`
- Create: `common-packages/packages/kanban-sdk/src/contracts.ts`
- Create: `common-packages/packages/kanban-sdk/src/runtime/pool.ts`
- Test: `common-packages/packages/kanban-sdk/src/kinds.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `KANBAN_KINDS`; types `Column`, `CardLink`, `KanbanBoard`, `KanbanCard`, `BoardDraft`, `CardDraft`; `KanbanSigner`, `NostrRuntime`, `SubscriptionHandle`, `KanbanCtx`, `SignerRequiredError`, `BoardNotFoundError`, `NotAMaintainerError`; `SimplePoolRuntime`.

- [ ] **Step 1: Create the package manifest**

Create `package.json`:

```json
{
  "name": "@formstr/kanban-sdk",
  "version": "0.1.0",
  "description": "Headless TypeScript SDK for Nostr Kanban boards (NIP-100) — interoperable with kanbanstr.com",
  "license": "MIT",
  "type": "module",
  "publishConfig": { "access": "public" },
  "main": "./dist/index.cjs",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "files": ["dist"],
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "require": "./dist/index.cjs"
    }
  },
  "scripts": {
    "build": "tsup",
    "typecheck": "tsc --noEmit",
    "clean": "rm -rf dist",
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  },
  "dependencies": {
    "@noble/hashes": "^1.8.0",
    "nostr-tools": "^2.23.3"
  },
  "devDependencies": {
    "@vitest/coverage-v8": "^3.2.4",
    "tsup": "^8.5.0",
    "typescript": "^5.6.0",
    "vitest": "^3.2.4"
  }
}
```

- [ ] **Step 2: Create the build and test configs**

Create `tsconfig.json`:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "."
  },
  "include": ["src", "test"]
}
```

Create `tsup.config.ts`:

```ts
import { defineConfig } from "tsup";

export default defineConfig({
  entry: { index: "src/index.ts" },
  format: ["esm", "cjs"],
  dts: true,
  clean: true,
  sourcemap: true,
  target: "es2022",
});
```

Create `vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    environment: "node",
    include: ["src/**/*.test.ts", "test/**/*.test.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "html", "json-summary"],
      include: ["src/**/*.ts"],
      exclude: ["src/**/*.test.ts", "src/**/index.ts", "src/**/types.ts"],
    },
  },
});
```

- [ ] **Step 3: Install dependencies**

Run from `common-packages/`: `pnpm install`
Expected: `packages/kanban-sdk` is linked into the workspace, `node_modules` populated.

- [ ] **Step 4: Write the failing test for the kind registry**

Create `src/kinds.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { KANBAN_KINDS } from "./kinds";

describe("KANBAN_KINDS", () => {
  it("matches the NIP-100 public kinds", () => {
    expect(KANBAN_KINDS.publicBoard).toBe(30301);
    expect(KANBAN_KINDS.publicCard).toBe(30302);
  });

  it("maps git status kinds used by tracker cards", () => {
    expect(KANBAN_KINDS.gitStatusOpen).toBe(1630);
    expect(KANBAN_KINDS.gitStatusClosed).toBe(1632);
  });

  it("has no duplicate kind numbers", () => {
    const values = Object.values(KANBAN_KINDS);
    expect(new Set(values).size).toBe(values.length);
  });
});
```

- [ ] **Step 5: Run the test to verify it fails**

Run: `pnpm vitest run src/kinds.test.ts`
Expected: FAIL — `Failed to resolve import "./kinds"`.

- [ ] **Step 6: Write the kind registry**

Create `src/kinds.ts`:

```ts
/**
 * Kanban event-kind registry.
 *
 * Public kinds (30301/30302) come from NIP PR #1665. They are NOT registered in
 * nips/README.md — see kanban/docs/07-gaps-risks.md §A1. Keep every number here
 * so a forced renumber is a one-file change.
 *
 * Private kinds (32301-32304, 1053/53) are specified in
 * kanban/docs/05-private-kanban-spec.md and land in Plan 2. They are declared
 * now so the registry is complete and the duplicate check is meaningful.
 */
export const KANBAN_KINDS = {
  // NIP-100 public
  publicBoard: 30301,
  publicCard: 30302,

  // NIP-100E private (Plan 2)
  privateBoard: 32301,
  privateCard: 32302,
  boardList: 32303,
  privateComment: 32304,
  inviteGiftWrap: 1053,
  inviteRumor: 53,
  membershipRemoval: 84,

  // Borrowed from other NIPs
  deletion: 5,
  seal: 13,
  relayList: 10002,
  publicComment: 1111,

  // Tracker-card targets (NIP-34)
  gitIssue: 1621,
  gitPatch: 1617,
  gitStatusOpen: 1630,
  gitStatusApplied: 1631,
  gitStatusClosed: 1632,
  gitStatusDraft: 1633,
} as const;

export type KanbanKind = (typeof KANBAN_KINDS)[keyof typeof KANBAN_KINDS];
```

- [ ] **Step 7: Run the test to verify it passes**

Run: `pnpm vitest run src/kinds.test.ts`
Expected: PASS, 3 tests.

- [ ] **Step 8: Write the domain types**

Create `src/types.ts`:

```ts
export interface Column {
  id: string;
  name: string;
  order: number;
}

export interface CardLink {
  boardPubkey: string;
  boardDTag: string;
  cardDTag: string;
  forwardLabel: string;
  reverseLabel: string;
}

export interface TrackedRef {
  eventId?: string;
  boardCoordinate?: string;
  cardDTag?: string;
}

export interface KanbanBoard {
  /** The `d` tag. Board identity is `kind:pubkey:id`. */
  id: string;
  pubkey: string;
  eventId: string;
  title: string;
  description: string;
  columns: Column[];
  maintainers: string[];
  noZap: boolean;
  createdAt: number;
  isPrivate: boolean;
  /** True for v0 boards: columns in JSON content, cards listed by board `a` tags. */
  legacy: boolean;
  /** Original tags, retained so edits can merge rather than rebuild. */
  rawTags: string[][];
}

export interface KanbanCard {
  /** The `d` tag. */
  id: string;
  pubkey: string;
  eventId: string;
  /** `30301:<pubkey>:<d>` of the owning board. */
  boardCoordinate: string;
  title: string;
  description: string;
  /** Column name on public boards; column id on private ones (Plan 2). */
  status?: string;
  rank: number;
  attachments: string[];
  assignees: string[];
  labels: string[];
  links: CardLink[];
  binned: boolean;
  createdAt: number;
  trackedKind?: number;
  trackedRef?: TrackedRef;
  rawTags: string[][];
}

export interface BoardDraft {
  title: string;
  description?: string;
  columns: Column[];
  maintainers?: string[];
  noZap?: boolean;
}

export interface CardDraft {
  title: string;
  description?: string;
  status?: string;
  rank?: number;
  attachments?: string[];
  assignees?: string[];
  labels?: string[];
  links?: CardLink[];
}
```

- [ ] **Step 9: Write the contracts**

Create `src/contracts.ts`:

```ts
import type { Event, EventTemplate, Filter } from "nostr-tools";

/**
 * Structural signer contract, deliberately identical to `@formstr/calendar-sdk`'s
 * `CalendarSigner` so one signer object satisfies every formstr SDK.
 * nip44 methods are unused on the public path but required so a single signer
 * carries through to Plan 2 without a second contract.
 */
export interface KanbanSigner {
  getPublicKey(): Promise<string>;
  signEvent(event: EventTemplate): Promise<Event>;
  nip44Encrypt(pubkey: string, plaintext: string): Promise<string>;
  nip44Decrypt(pubkey: string, ciphertext: string): Promise<string>;
}

export interface SubscriptionHandle {
  unsub(): void;
}

/** Every byte of network I/O in the SDK goes through this. */
export interface NostrRuntime {
  querySync(relays: string[], filter: Filter, timeoutMs?: number): Promise<Event[]>;
  subscribe(
    relays: string[],
    filters: Filter[],
    options?: { onEvent?: (event: Event) => void; onEose?: () => void },
  ): SubscriptionHandle;
  publish(relays: string[], event: Event, timeoutMs?: number): Promise<void>;
  dispose?(): void;
}

/** Per-instance context threaded through every service call. */
export interface KanbanCtx {
  /** Resolves the configured signer or throws `SignerRequiredError`. */
  getSigner(): Promise<KanbanSigner>;
  runtime: NostrRuntime;
  relays: string[];
}

export class SignerRequiredError extends Error {
  constructor(operation: string) {
    super(`${operation} requires a signer — construct the SDK with one: new KanbanSDK({ signer })`);
    this.name = "SignerRequiredError";
  }
}

export class BoardNotFoundError extends Error {
  constructor(coordinate: string) {
    super(`Board not found: ${coordinate}`);
    this.name = "BoardNotFoundError";
  }
}

export class NotAMaintainerError extends Error {
  constructor(pubkey: string, coordinate: string) {
    super(`${pubkey} is not a maintainer of ${coordinate}`);
    this.name = "NotAMaintainerError";
  }
}
```

- [ ] **Step 10: Write the default runtime**

Create `src/runtime/pool.ts`:

```ts
import { SimplePool, type Event, type Filter } from "nostr-tools";

import type { NostrRuntime, SubscriptionHandle } from "../contracts";

const DEFAULT_TIMEOUT_MS = 5000;

/** Zero-config `NostrRuntime` over nostr-tools' SimplePool. */
export class SimplePoolRuntime implements NostrRuntime {
  private readonly pool = new SimplePool();
  /** Every relay we have ever touched, so `dispose` can actually close them. */
  private readonly touched = new Set<string>();

  private track(relays: string[]): string[] {
    for (const relay of relays) this.touched.add(relay);
    return relays;
  }

  async querySync(relays: string[], filter: Filter, timeoutMs = DEFAULT_TIMEOUT_MS): Promise<Event[]> {
    this.track(relays);
    const seen = new Map<string, Event>();
    return new Promise((resolve) => {
      const finish = () => {
        clearTimeout(timer);
        sub.close();
        resolve([...seen.values()]);
      };
      const timer = setTimeout(finish, timeoutMs);
      const sub = this.pool.subscribeMany(relays, [filter], {
        onevent: (event) => seen.set(event.id, event),
        oneose: finish,
      });
    });
  }

  subscribe(
    relays: string[],
    filters: Filter[],
    options: { onEvent?: (event: Event) => void; onEose?: () => void } = {},
  ): SubscriptionHandle {
    const sub = this.pool.subscribeMany(this.track(relays), filters, {
      onevent: (event) => options.onEvent?.(event),
      oneose: () => options.onEose?.(),
    });
    return { unsub: () => sub.close() };
  }

  async publish(relays: string[], event: Event, timeoutMs = DEFAULT_TIMEOUT_MS): Promise<void> {
    const publishes = this.pool.publish(this.track(relays), event);
    await Promise.race([
      Promise.allSettled(publishes),
      new Promise((resolve) => setTimeout(resolve, timeoutMs)),
    ]);
  }

  /** `pool.close([])` closes nothing — it takes the relays to close, not a flag. */
  dispose(): void {
    this.pool.close([...this.touched]);
    this.touched.clear();
  }
}
```

- [ ] **Step 11: Verify the package typechecks**

Run: `pnpm typecheck`
Expected: no errors.

- [ ] **Step 12: Commit**

```bash
git add packages/kanban-sdk
git commit -m "feat(kanban-sdk): scaffold package with kinds, types, contracts, pool runtime"
```

---

### Task 2: NIP-01 event resolution

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/discovery/dedupe.ts`
- Test: `common-packages/packages/kanban-sdk/src/discovery/dedupe.test.ts`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces: `supersedes(candidate: Event, incumbent: Event): boolean`, `newestByDTag(events: Iterable<Event>): Map<string, Event>`, `nextCreatedAt(previousCreatedAt?: number): number`.

**Why this exists:** kanbanstr breaks ties by relay response order (`stores/kanban.ts:1246`), so two maintainers editing within one second make clients disagree with the relay non-deterministically. See `kanban/docs/03-kanbanstr-review.md` §6.1.

- [ ] **Step 1: Write the failing tests**

Create `src/discovery/dedupe.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import { newestByDTag, nextCreatedAt, supersedes } from "./dedupe";

function evt(id: string, createdAt: number, dTag = "card1"): Event {
  return {
    id,
    created_at: createdAt,
    kind: 30302,
    pubkey: "a".repeat(64),
    tags: [["d", dTag]],
    content: "",
    sig: "",
  } as Event;
}

describe("supersedes", () => {
  it("prefers the newer created_at", () => {
    expect(supersedes(evt("bbb", 200), evt("aaa", 100))).toBe(true);
    expect(supersedes(evt("aaa", 100), evt("bbb", 200))).toBe(false);
  });

  it("breaks created_at ties by LOWEST id, as relays do", () => {
    expect(supersedes(evt("aaa", 100), evt("bbb", 100))).toBe(true);
    expect(supersedes(evt("bbb", 100), evt("aaa", 100))).toBe(false);
  });

  it("does not supersede itself", () => {
    expect(supersedes(evt("aaa", 100), evt("aaa", 100))).toBe(false);
  });
});

describe("newestByDTag", () => {
  it("keeps one event per d tag", () => {
    const resolved = newestByDTag([
      evt("aaa", 100, "card1"),
      evt("bbb", 200, "card1"),
      evt("ccc", 150, "card2"),
    ]);
    expect(resolved.size).toBe(2);
    expect(resolved.get("card1")?.id).toBe("bbb");
    expect(resolved.get("card2")?.id).toBe("ccc");
  });

  it("resolves ties independently of iteration order", () => {
    const forward = newestByDTag([evt("bbb", 100), evt("aaa", 100)]);
    const reverse = newestByDTag([evt("aaa", 100), evt("bbb", 100)]);
    expect(forward.get("card1")?.id).toBe("aaa");
    expect(reverse.get("card1")?.id).toBe("aaa");
  });
});

describe("nextCreatedAt", () => {
  it("returns now when there is no previous version", () => {
    const now = Math.floor(Date.now() / 1000);
    expect(nextCreatedAt()).toBeGreaterThanOrEqual(now);
  });

  it("forces strictly-increasing timestamps within the same second", () => {
    const now = Math.floor(Date.now() / 1000);
    expect(nextCreatedAt(now)).toBe(now + 1);
    expect(nextCreatedAt(now + 500)).toBe(now + 501);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/discovery/dedupe.test.ts`
Expected: FAIL — `Failed to resolve import "./dedupe"`.

- [ ] **Step 3: Write the implementation**

Create `src/discovery/dedupe.ts`:

```ts
import type { Event } from "nostr-tools";

/**
 * NIP-01 replaceable-event resolution. From 01.md:101 — "In case of replaceable
 * events with the same timestamp, the event with the lowest id (first in lexical
 * order) should be retained, and the other discarded."
 *
 * Relays apply exactly this. A client that ties any other way (e.g. keep-first,
 * which depends on relay response order) can disagree with the relay about which
 * version is current.
 */
export function supersedes(candidate: Event, incumbent: Event): boolean {
  if (candidate.created_at !== incumbent.created_at) {
    return candidate.created_at > incumbent.created_at;
  }
  return candidate.id < incumbent.id;
}

/** Resolve many versions of addressable events to the current one per `d` tag. */
export function newestByDTag(events: Iterable<Event>): Map<string, Event> {
  const newest = new Map<string, Event>();
  for (const event of events) {
    const dTag = event.tags.find((t) => t[0] === "d")?.[1] ?? "";
    const previous = newest.get(dTag);
    if (!previous || supersedes(event, previous)) newest.set(dTag, event);
  }
  return newest;
}

/**
 * Strict supersession. Two writes inside the same second otherwise TIE, and the
 * NIP-01 tie-break can keep the stale version — silently losing an entire edit.
 */
export function nextCreatedAt(previousCreatedAt?: number): number {
  const now = Math.floor(Date.now() / 1000);
  if (previousCreatedAt === undefined) return now;
  return Math.max(now, previousCreatedAt + 1);
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run src/discovery/dedupe.test.ts`
Expected: PASS, 7 tests.

- [ ] **Step 5: Commit**

```bash
git add packages/kanban-sdk/src/discovery
git commit -m "feat(kanban-sdk): NIP-01 event resolution with correct tie-breaking"
```

---

### Task 3: Fractional rank ordering

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/codec/rank.ts`
- Test: `common-packages/packages/kanban-sdk/src/codec/rank.test.ts`

**Interfaces:**
- Consumes: nothing.
- Produces: `RANK_STEP: number`, `computeRank(sortedRanks: number[], targetIndex: number): number`, `needsRebalance(sortedRanks: number[]): boolean`, `rebalance(count: number): number[]`.

**Why `rebalance` exists:** NIP-100 specifies no rebalancing, and repeated insertion at the same position halves the gap each time until float precision runs out. See `kanban/docs/07-gaps-risks.md` §A5.

- [ ] **Step 1: Write the failing tests**

Create `src/codec/rank.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { RANK_STEP, computeRank, needsRebalance, rebalance } from "./rank";

describe("computeRank", () => {
  it("returns the base step for an empty column", () => {
    expect(computeRank([], 0)).toBe(RANK_STEP);
  });

  it("places a card before the first by stepping down", () => {
    expect(computeRank([10, 20, 30], 0)).toBe(0);
  });

  it("places a card after the last by stepping up", () => {
    expect(computeRank([10, 20, 30], 3)).toBe(40);
  });

  it("places a card between two neighbours at the midpoint", () => {
    expect(computeRank([10, 20, 30], 1)).toBe(15);
    expect(computeRank([10, 20, 30], 2)).toBe(25);
  });

  it("clamps an out-of-range index to the end", () => {
    expect(computeRank([10], 99)).toBe(20);
  });

  it("treats a negative index as the start", () => {
    expect(computeRank([10], -3)).toBe(0);
  });
});

describe("needsRebalance", () => {
  it("is false for well-spaced ranks", () => {
    expect(needsRebalance([10, 20, 30])).toBe(false);
  });

  it("is true when neighbours are closer than the precision floor", () => {
    expect(needsRebalance([10, 10 + 1e-10, 20])).toBe(true);
  });

  it("is true for exactly equal neighbours", () => {
    expect(needsRebalance([10, 10, 20])).toBe(true);
  });

  it("is false for fewer than two cards", () => {
    expect(needsRebalance([10])).toBe(false);
    expect(needsRebalance([])).toBe(false);
  });
});

describe("rebalance", () => {
  it("produces evenly spaced ranks", () => {
    expect(rebalance(3)).toEqual([10, 20, 30]);
  });

  it("produces an empty array for zero cards", () => {
    expect(rebalance(0)).toEqual([]);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/codec/rank.test.ts`
Expected: FAIL — `Failed to resolve import "./rank"`.

- [ ] **Step 3: Write the implementation**

Create `src/codec/rank.ts`:

```ts
/**
 * Fractional indexing for card order within a column. Cards sort by ascending
 * `rank`. kanbanstr uses the same midpoint scheme (stores/kanban.ts:790) but
 * never rebalances; `needsRebalance` + `rebalance` close that gap.
 */

export const RANK_STEP = 10;

/** Smallest gap we tolerate before rebalancing. Well above float epsilon. */
const PRECISION_FLOOR = 1e-6;

/**
 * Rank for a card landing at `targetIndex` within a column whose existing cards
 * (excluding the one being moved) have `sortedRanks` in ascending order.
 */
export function computeRank(sortedRanks: number[], targetIndex: number): number {
  if (sortedRanks.length === 0) return RANK_STEP;

  if (targetIndex <= 0) return sortedRanks[0] - RANK_STEP;
  if (targetIndex >= sortedRanks.length) {
    return sortedRanks[sortedRanks.length - 1] + RANK_STEP;
  }

  const before = sortedRanks[targetIndex - 1];
  const after = sortedRanks[targetIndex];
  return before + (after - before) / 2;
}

/** True when any adjacent pair has collapsed and the column should be rewritten. */
export function needsRebalance(sortedRanks: number[]): boolean {
  for (let i = 1; i < sortedRanks.length; i += 1) {
    if (sortedRanks[i] - sortedRanks[i - 1] < PRECISION_FLOOR) return true;
  }
  return false;
}

/** Evenly spaced ranks for `count` cards, in display order. */
export function rebalance(count: number): number[] {
  return Array.from({ length: count }, (_, i) => (i + 1) * RANK_STEP);
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run src/codec/rank.test.ts`
Expected: PASS, 12 tests.

- [ ] **Step 5: Commit**

```bash
git add packages/kanban-sdk/src/codec
git commit -m "feat(kanban-sdk): fractional rank ordering with rebalance detection"
```

---

### Task 4: Public board codec

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/codec/board.ts`
- Test: `common-packages/packages/kanban-sdk/src/codec/board.test.ts`

**Interfaces:**
- Consumes: `KANBAN_KINDS` (Task 1), `KanbanBoard`, `Column`, `BoardDraft` (Task 1).
- Produces:
  - `BOARD_MANAGED_TAGS: readonly string[]`
  - `buildPublicBoardTags(draft: BoardDraft, dTag: string): string[][]`
  - `parsePublicBoard(event: Event): KanbanBoard | null`
  - `isLegacyBoard(event: Event): boolean`
  - `mergeTags(existing: string[][], next: string[][], managed: readonly string[]): string[][]`
  - `boardCoordinate(board: Pick<KanbanBoard, "pubkey" | "id">): string`

- [ ] **Step 1: Write the failing tests**

Create `src/codec/board.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import {
  boardCoordinate,
  buildPublicBoardTags,
  isLegacyBoard,
  mergeTags,
  parsePublicBoard,
  BOARD_MANAGED_TAGS,
} from "./board";

const PUBKEY = "a".repeat(64);

function boardEvent(tags: string[][], content = ""): Event {
  return {
    id: "e".repeat(64),
    pubkey: PUBKEY,
    created_at: 1700000000,
    kind: 30301,
    tags,
    content,
    sig: "",
  } as Event;
}

describe("buildPublicBoardTags", () => {
  it("emits d, title, description, alt, col and p tags", () => {
    const tags = buildPublicBoardTags(
      {
        title: "Roadmap",
        description: "Q3 work",
        columns: [
          { id: "c1", name: "To Do", order: 0 },
          { id: "c2", name: "Done", order: 1 },
        ],
        maintainers: ["b".repeat(64)],
      },
      "board7",
    );

    expect(tags).toContainEqual(["d", "board7"]);
    expect(tags).toContainEqual(["title", "Roadmap"]);
    expect(tags).toContainEqual(["description", "Q3 work"]);
    expect(tags).toContainEqual(["alt", "A board titled Roadmap"]);
    expect(tags).toContainEqual(["col", "c1", "To Do", "0"]);
    expect(tags).toContainEqual(["col", "c2", "Done", "1"]);
    expect(tags).toContainEqual(["p", "b".repeat(64)]);
  });

  it("emits nozap only when requested", () => {
    const without = buildPublicBoardTags({ title: "X", columns: [] }, "d1");
    expect(without.some((t) => t[0] === "nozap")).toBe(false);

    const with_ = buildPublicBoardTags({ title: "X", columns: [], noZap: true }, "d1");
    expect(with_).toContainEqual(["nozap"]);
  });
});

describe("parsePublicBoard", () => {
  it("reads a well-formed board", () => {
    const board = parsePublicBoard(
      boardEvent([
        ["d", "board7"],
        ["title", "Roadmap"],
        ["description", "Q3 work"],
        ["col", "c1", "To Do", "0"],
        ["col", "c2", "Done", "1"],
        ["p", "b".repeat(64)],
      ]),
    );

    expect(board).not.toBeNull();
    expect(board!.id).toBe("board7");
    expect(board!.title).toBe("Roadmap");
    expect(board!.columns).toEqual([
      { id: "c1", name: "To Do", order: 0 },
      { id: "c2", name: "Done", order: 1 },
    ]);
    expect(board!.maintainers).toEqual(["b".repeat(64)]);
    expect(board!.noZap).toBe(false);
    expect(board!.legacy).toBe(false);
    expect(board!.isPrivate).toBe(false);
  });

  it("sorts columns by order regardless of tag sequence", () => {
    const board = parsePublicBoard(
      boardEvent([
        ["d", "b1"],
        ["title", "X"],
        ["col", "c2", "Done", "1"],
        ["col", "c1", "To Do", "0"],
      ]),
    );
    expect(board!.columns.map((c) => c.id)).toEqual(["c1", "c2"]);
  });

  it("returns null when the d tag is missing", () => {
    expect(parsePublicBoard(boardEvent([["title", "X"]]))).toBeNull();
  });

  it("falls back to a placeholder title", () => {
    const board = parsePublicBoard(boardEvent([["d", "b1"]]));
    expect(board!.title).toBe("Untitled Board");
  });

  it("reads the nozap flag", () => {
    const board = parsePublicBoard(boardEvent([["d", "b1"], ["title", "X"], ["nozap"]]));
    expect(board!.noZap).toBe(true);
  });

  it("retains rawTags so edits can merge", () => {
    const event = boardEvent([["d", "b1"], ["title", "X"], ["unknown", "keep me"]]);
    expect(parsePublicBoard(event)!.rawTags).toContainEqual(["unknown", "keep me"]);
  });
});

describe("isLegacyBoard", () => {
  it("detects v0 boards that list cards with a tags", () => {
    expect(isLegacyBoard(boardEvent([["d", "b1"], ["a", "30302:x:card1"]]))).toBe(true);
  });

  it("detects v0 boards that keep columns in JSON content", () => {
    const event = boardEvent([["d", "b1"]], JSON.stringify({ columns: [], description: "old" }));
    expect(isLegacyBoard(event)).toBe(true);
  });

  it("is false for current-format boards", () => {
    expect(isLegacyBoard(boardEvent([["d", "b1"], ["col", "c1", "To Do", "0"]]))).toBe(false);
  });

  it("does not throw on non-JSON content", () => {
    expect(isLegacyBoard(boardEvent([["d", "b1"]], "not json"))).toBe(false);
  });
});

describe("mergeTags", () => {
  it("replaces managed tags and preserves unknown ones", () => {
    const merged = mergeTags(
      [["d", "b1"], ["title", "Old"], ["nozap"], ["weird", "keep"]],
      [["d", "b1"], ["title", "New"]],
      BOARD_MANAGED_TAGS,
    );

    expect(merged).toContainEqual(["title", "New"]);
    expect(merged).not.toContainEqual(["title", "Old"]);
    expect(merged).toContainEqual(["weird", "keep"]);
  });

  it("drops a managed tag that the new set omits", () => {
    const merged = mergeTags([["d", "b1"], ["nozap"]], [["d", "b1"]], BOARD_MANAGED_TAGS);
    expect(merged.some((t) => t[0] === "nozap")).toBe(false);
  });
});

describe("boardCoordinate", () => {
  it("builds a NIP-01 address", () => {
    expect(boardCoordinate({ pubkey: PUBKEY, id: "board7" })).toBe(`30301:${PUBKEY}:board7`);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/codec/board.test.ts`
Expected: FAIL — `Failed to resolve import "./board"`.

- [ ] **Step 3: Write the implementation**

Create `src/codec/board.ts`:

```ts
import type { Event } from "nostr-tools";

import { KANBAN_KINDS } from "../kinds";
import type { BoardDraft, Column, KanbanBoard } from "../types";

/**
 * Tags this codec owns. `mergeTags` replaces exactly these and leaves everything
 * else alone, so an edit cannot silently drop a tag the model does not know about
 * — which is how kanbanstr loses `nozap` on every board update
 * (kanban/docs/03-kanbanstr-review.md §6.3).
 */
export const BOARD_MANAGED_TAGS = ["d", "title", "description", "alt", "col", "p", "nozap"] as const;

export function boardCoordinate(board: Pick<KanbanBoard, "pubkey" | "id">): string {
  return `${KANBAN_KINDS.publicBoard}:${board.pubkey}:${board.id}`;
}

export function buildPublicBoardTags(draft: BoardDraft, dTag: string): string[][] {
  const tags: string[][] = [
    ["d", dTag],
    ["title", draft.title],
    ["description", draft.description ?? ""],
    ["alt", `A board titled ${draft.title}`],
  ];

  for (const column of draft.columns) {
    tags.push(["col", column.id, column.name, String(column.order)]);
  }
  for (const maintainer of draft.maintainers ?? []) {
    tags.push(["p", maintainer]);
  }
  if (draft.noZap) tags.push(["nozap"]);

  return tags;
}

/**
 * v0 boards kept columns and description in stringified JSON `content` and listed
 * their cards with board-side `a` tags. They still exist on relays; we read them
 * but never write them.
 */
export function isLegacyBoard(event: Event): boolean {
  if (event.tags.some((t) => t[0] === "a")) return true;
  if (!event.content) return false;
  try {
    const parsed = JSON.parse(event.content) as { columns?: unknown };
    return Array.isArray(parsed.columns);
  } catch {
    return false;
  }
}

export function parsePublicBoard(event: Event): KanbanBoard | null {
  const id = event.tags.find((t) => t[0] === "d")?.[1];
  if (!id) return null;

  const legacy = isLegacyBoard(event);
  let columns: Column[] = [];
  let description = event.tags.find((t) => t[0] === "description")?.[1] ?? "";

  if (legacy) {
    try {
      const parsed = JSON.parse(event.content) as {
        columns?: Column[];
        description?: string;
      };
      columns = parsed.columns ?? [];
      description = parsed.description ?? description;
    } catch {
      columns = [];
    }
  } else {
    columns = event.tags
      .filter((t) => t[0] === "col")
      .map((t) => ({ id: t[1], name: t[2], order: Number.parseInt(t[3] ?? "0", 10) }));
  }

  columns = [...columns].sort((a, b) => a.order - b.order);

  return {
    id,
    pubkey: event.pubkey,
    eventId: event.id,
    title: event.tags.find((t) => t[0] === "title")?.[1] ?? "Untitled Board",
    description,
    columns,
    maintainers: event.tags.filter((t) => t[0] === "p").map((t) => t[1]),
    noZap: event.tags.some((t) => t[0] === "nozap"),
    createdAt: event.created_at,
    isPrivate: false,
    legacy,
    rawTags: event.tags,
  };
}

/**
 * Merge freshly built tags into an existing tag array, replacing only the managed
 * names. Unknown tags survive the round trip.
 */
export function mergeTags(
  existing: string[][],
  next: string[][],
  managed: readonly string[],
): string[][] {
  const managedSet = new Set(managed);
  const preserved = existing.filter((tag) => !managedSet.has(tag[0]));
  return [...next, ...preserved];
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run src/codec/board.test.ts`
Expected: PASS, 15 tests.

- [ ] **Step 5: Commit**

```bash
git add packages/kanban-sdk/src/codec/board.ts packages/kanban-sdk/src/codec/board.test.ts
git commit -m "feat(kanban-sdk): public board codec with legacy read and tag-preserving merge"
```

---

### Task 5: Public card codec

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/codec/card.ts`
- Test: `common-packages/packages/kanban-sdk/src/codec/card.test.ts`

**Interfaces:**
- Consumes: `KanbanCard`, `CardDraft`, `CardLink`, `TrackedRef` (Task 1). The codec itself does **not** use `mergeTags` — merging happens in the card service (Task 8).
- Produces:
  - `CARD_MANAGED_TAGS: readonly string[]` — note it excludes `k`, `e`, `refs/board`, `refs/card`, so tracker references survive every edit
  - `buildPublicCardTags(draft: CardDraft, dTag: string, boardCoordinate: string, rank: number): string[][]`
  - `parsePublicCard(event: Event): KanbanCard | null`
  - `parseCardLink(tag: string[]): CardLink | null`
  - `buildCardLinkTag(link: CardLink): string[]`

**Compatibility notes that drive the tests:** kanbanstr writes each assignee as **both** `["zap", pk]` and `["p", pk]` (`stores/kanban.ts:737`) and reads either (`:521`). Links use `["i", "kanban:<pk>:<board>:<card>", forwardLabel, reverseLabel]` — the `refs/link` form in `old-NIP-100.md` is dead and must not be written.

- [ ] **Step 1: Write the failing tests**

Create `src/codec/card.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import { buildCardLinkTag, buildPublicCardTags, parseCardLink, parsePublicCard } from "./card";

const PUBKEY = "a".repeat(64);
const BOARD_PUBKEY = "b".repeat(64);
const COORD = `30301:${BOARD_PUBKEY}:board7`;

function cardEvent(tags: string[][]): Event {
  return {
    id: "e".repeat(64),
    pubkey: PUBKEY,
    created_at: 1700000000,
    kind: 30302,
    tags,
    content: "",
    sig: "",
  } as Event;
}

describe("buildPublicCardTags", () => {
  it("emits the core card tags", () => {
    const tags = buildPublicCardTags(
      { title: "Ship it", description: "body", status: "To Do" },
      "card1",
      COORD,
      15,
    );

    expect(tags).toContainEqual(["d", "card1"]);
    expect(tags).toContainEqual(["title", "Ship it"]);
    expect(tags).toContainEqual(["description", "body"]);
    expect(tags).toContainEqual(["alt", "A card titled Ship it"]);
    expect(tags).toContainEqual(["s", "To Do"]);
    expect(tags).toContainEqual(["rank", "15"]);
    expect(tags).toContainEqual(["a", COORD]);
  });

  it("writes each assignee as BOTH p and zap, matching kanbanstr", () => {
    const tags = buildPublicCardTags({ title: "X", assignees: [PUBKEY] }, "c1", COORD, 10);
    expect(tags).toContainEqual(["p", PUBKEY]);
    expect(tags).toContainEqual(["zap", PUBKEY]);
  });

  it("omits the s tag when there is no status", () => {
    const tags = buildPublicCardTags({ title: "X" }, "c1", COORD, 10);
    expect(tags.some((t) => t[0] === "s")).toBe(false);
  });

  it("emits attachments, labels and links", () => {
    const tags = buildPublicCardTags(
      {
        title: "X",
        attachments: ["https://example.com/a.png"],
        labels: ["backend"],
        links: [
          {
            boardPubkey: BOARD_PUBKEY,
            boardDTag: "board9",
            cardDTag: "card9",
            forwardLabel: "is blocked by",
            reverseLabel: "blocks",
          },
        ],
      },
      "c1",
      COORD,
      10,
    );

    expect(tags).toContainEqual(["u", "https://example.com/a.png"]);
    expect(tags).toContainEqual(["t", "backend"]);
    expect(tags).toContainEqual([
      "i",
      `kanban:${BOARD_PUBKEY}:board9:card9`,
      "is blocked by",
      "blocks",
    ]);
  });
});

describe("parsePublicCard", () => {
  it("reads a well-formed card", () => {
    const card = parsePublicCard(
      cardEvent([
        ["d", "card1"],
        ["title", "Ship it"],
        ["description", "body"],
        ["s", "To Do"],
        ["rank", "15"],
        ["a", COORD],
        ["u", "https://example.com/a.png"],
        ["t", "backend"],
      ]),
    );

    expect(card).not.toBeNull();
    expect(card!.id).toBe("card1");
    expect(card!.title).toBe("Ship it");
    expect(card!.status).toBe("To Do");
    expect(card!.rank).toBe(15);
    expect(card!.boardCoordinate).toBe(COORD);
    expect(card!.attachments).toEqual(["https://example.com/a.png"]);
    expect(card!.labels).toEqual(["backend"]);
    expect(card!.binned).toBe(false);
  });

  it("deduplicates assignees written as both p and zap", () => {
    const card = parsePublicCard(
      cardEvent([["d", "c1"], ["a", COORD], ["p", PUBKEY], ["zap", PUBKEY]]),
    );
    expect(card!.assignees).toEqual([PUBKEY]);
  });

  it("reads an assignee written only as zap", () => {
    const card = parsePublicCard(cardEvent([["d", "c1"], ["a", COORD], ["zap", PUBKEY]]));
    expect(card!.assignees).toEqual([PUBKEY]);
  });

  it("reads the binned flag", () => {
    const card = parsePublicCard(cardEvent([["d", "c1"], ["a", COORD], ["binned"]]));
    expect(card!.binned).toBe(true);
  });

  it("defaults rank to 0 when absent or unparseable", () => {
    expect(parsePublicCard(cardEvent([["d", "c1"], ["a", COORD]]))!.rank).toBe(0);
    expect(parsePublicCard(cardEvent([["d", "c1"], ["a", COORD], ["rank", "x"]]))!.rank).toBe(0);
  });

  it("returns null without a d tag", () => {
    expect(parsePublicCard(cardEvent([["a", COORD]]))).toBeNull();
  });

  it("captures tracker references", () => {
    const card = parsePublicCard(
      cardEvent([["d", "c1"], ["a", COORD], ["k", "1621"], ["e", "f".repeat(64), "wss://r"]]),
    );
    expect(card!.trackedKind).toBe(1621);
    expect(card!.trackedRef).toEqual({ eventId: "f".repeat(64) });
  });

  it("captures kanban-card tracker references", () => {
    const card = parsePublicCard(
      cardEvent([
        ["d", "c1"],
        ["a", COORD],
        ["k", "30302"],
        ["refs/board", `30301:${BOARD_PUBKEY}:board9`],
        ["refs/card", "card9"],
      ]),
    );
    expect(card!.trackedKind).toBe(30302);
    expect(card!.trackedRef).toEqual({
      boardCoordinate: `30301:${BOARD_PUBKEY}:board9`,
      cardDTag: "card9",
    });
  });
});

describe("card links", () => {
  it("round-trips a link through build and parse", () => {
    const link = {
      boardPubkey: BOARD_PUBKEY,
      boardDTag: "board9",
      cardDTag: "card9",
      forwardLabel: "is a child of",
      reverseLabel: "is a parent of",
    };
    expect(parseCardLink(buildCardLinkTag(link))).toEqual(link);
  });

  it("ignores i tags that are not kanban links", () => {
    expect(parseCardLink(["i", "podcast:guid:abc"])).toBeNull();
  });

  it("ignores malformed kanban links", () => {
    expect(parseCardLink(["i", "kanban:only:two"])).toBeNull();
  });

  it("defaults missing labels to empty strings", () => {
    const parsed = parseCardLink(["i", `kanban:${BOARD_PUBKEY}:board9:card9`]);
    expect(parsed!.forwardLabel).toBe("");
    expect(parsed!.reverseLabel).toBe("");
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/codec/card.test.ts`
Expected: FAIL — `Failed to resolve import "./card"`.

- [ ] **Step 3: Write the implementation**

Create `src/codec/card.ts`:

```ts
import type { Event } from "nostr-tools";

import type { CardDraft, CardLink, KanbanCard, TrackedRef } from "../types";

/** Tags this codec owns; see BOARD_MANAGED_TAGS for why merging matters. */
export const CARD_MANAGED_TAGS = [
  "d",
  "title",
  "description",
  "alt",
  "s",
  "rank",
  "a",
  "u",
  "t",
  "p",
  "zap",
  "i",
  "binned",
] as const;

export function buildCardLinkTag(link: CardLink): string[] {
  return [
    "i",
    `kanban:${link.boardPubkey}:${link.boardDTag}:${link.cardDTag}`,
    link.forwardLabel,
    link.reverseLabel,
  ];
}

export function parseCardLink(tag: string[]): CardLink | null {
  if (tag[0] !== "i" || !tag[1]?.startsWith("kanban:")) return null;
  const parts = tag[1].split(":");
  if (parts.length !== 4) return null;
  const [, boardPubkey, boardDTag, cardDTag] = parts;
  return {
    boardPubkey,
    boardDTag,
    cardDTag,
    forwardLabel: tag[2] ?? "",
    reverseLabel: tag[3] ?? "",
  };
}

export function buildPublicCardTags(
  draft: CardDraft,
  dTag: string,
  boardCoordinate: string,
  rank: number,
): string[][] {
  const tags: string[][] = [
    ["d", dTag],
    ["title", draft.title],
    ["description", draft.description ?? ""],
    ["alt", `A card titled ${draft.title}`],
    ["rank", String(rank)],
    ["a", boardCoordinate],
  ];

  if (draft.status) tags.push(["s", draft.status]);
  for (const url of draft.attachments ?? []) tags.push(["u", url]);
  for (const label of draft.labels ?? []) tags.push(["t", label]);

  // kanbanstr writes assignees twice so zaps route to them. Match it exactly.
  for (const assignee of draft.assignees ?? []) {
    tags.push(["zap", assignee]);
    tags.push(["p", assignee]);
  }

  for (const link of draft.links ?? []) tags.push(buildCardLinkTag(link));

  return tags;
}

function parseTrackedRef(event: Event, trackedKind: number): TrackedRef {
  if (trackedKind === 30302) {
    return {
      boardCoordinate: event.tags.find((t) => t[0] === "refs/board")?.[1],
      cardDTag: event.tags.find((t) => t[0] === "refs/card")?.[1],
    };
  }
  return { eventId: event.tags.find((t) => t[0] === "e")?.[1] };
}

export function parsePublicCard(event: Event): KanbanCard | null {
  const id = event.tags.find((t) => t[0] === "d")?.[1];
  if (!id) return null;

  const rawRank = event.tags.find((t) => t[0] === "rank")?.[1];
  const parsedRank = rawRank === undefined ? Number.NaN : Number.parseFloat(rawRank);

  const assignees = [
    ...new Set(event.tags.filter((t) => t[0] === "p" || t[0] === "zap").map((t) => t[1])),
  ];

  const rawTrackedKind = event.tags.find((t) => t[0] === "k")?.[1];
  const trackedKind = rawTrackedKind ? Number.parseInt(rawTrackedKind, 10) : undefined;

  return {
    id,
    pubkey: event.pubkey,
    eventId: event.id,
    boardCoordinate: event.tags.find((t) => t[0] === "a")?.[1] ?? "",
    title: event.tags.find((t) => t[0] === "title")?.[1] ?? "Untitled Card",
    description: event.tags.find((t) => t[0] === "description")?.[1] ?? "",
    status: event.tags.find((t) => t[0] === "s")?.[1],
    rank: Number.isNaN(parsedRank) ? 0 : parsedRank,
    attachments: event.tags.filter((t) => t[0] === "u").map((t) => t[1]),
    assignees,
    labels: event.tags.filter((t) => t[0] === "t").map((t) => t[1]),
    links: event.tags
      .map(parseCardLink)
      .filter((link): link is CardLink => link !== null),
    binned: event.tags.some((t) => t[0] === "binned"),
    createdAt: event.created_at,
    trackedKind,
    trackedRef: trackedKind === undefined ? undefined : parseTrackedRef(event, trackedKind),
    rawTags: event.tags,
  };
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run src/codec/card.test.ts`
Expected: PASS, 16 tests.

- [ ] **Step 5: Commit**

```bash
git add packages/kanban-sdk/src/codec/card.ts packages/kanban-sdk/src/codec/card.test.ts
git commit -m "feat(kanban-sdk): public card codec matching kanbanstr wire format"
```

---

### Task 6: Deletion filtering and relay resolution

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/discovery/deletions.ts`
- Create: `common-packages/packages/kanban-sdk/src/discovery/relays.ts`
- Test: `common-packages/packages/kanban-sdk/src/discovery/deletions.test.ts`
- Test: `common-packages/packages/kanban-sdk/src/discovery/relays.test.ts`

**Interfaces:**
- Consumes: nothing from earlier tasks.
- Produces:
  - `DeletedSet` — `{ ids: Set<string>; coordinates: Set<string> }`
  - `collectDeleted(deletionEvents: Event[]): DeletedSet`
  - `isDeleted(event: Event, deleted: DeletedSet, coordinateOf: (e: Event) => string): boolean`
  - `normalizeRelayUrl(url: string): string`
  - `normalizeRelayList(urls: string[]): string[]`
  - `parseRelayList(event: Event): { read: string[]; write: string[] }`

**Why deletion filtering is client-side:** relays honour NIP-09 inconsistently and addressable events keep resolving after a delete. See `kanban/docs/01-nostr-primitives.md`.

- [ ] **Step 1: Write the failing deletion tests**

Create `src/discovery/deletions.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import { collectDeleted, isDeleted } from "./deletions";

const PUBKEY = "a".repeat(64);

function deletion(tags: string[][]): Event {
  return { id: "d1", pubkey: PUBKEY, created_at: 1, kind: 5, tags, content: "", sig: "" } as Event;
}

function board(id: string, dTag: string): Event {
  return {
    id,
    pubkey: PUBKEY,
    created_at: 1,
    kind: 30301,
    tags: [["d", dTag]],
    content: "",
    sig: "",
  } as Event;
}

const coordinateOf = (e: Event) =>
  `${e.kind}:${e.pubkey}:${e.tags.find((t) => t[0] === "d")?.[1] ?? ""}`;

describe("collectDeleted", () => {
  it("collects both e and a targets", () => {
    const deleted = collectDeleted([
      deletion([
        ["e", "abc"],
        ["a", `30301:${PUBKEY}:board7`],
      ]),
    ]);
    expect(deleted.ids.has("abc")).toBe(true);
    expect(deleted.coordinates.has(`30301:${PUBKEY}:board7`)).toBe(true);
  });

  it("returns empty sets for no deletions", () => {
    const deleted = collectDeleted([]);
    expect(deleted.ids.size).toBe(0);
    expect(deleted.coordinates.size).toBe(0);
  });
});

describe("isDeleted", () => {
  it("matches by event id", () => {
    const deleted = collectDeleted([deletion([["e", "abc"]])]);
    expect(isDeleted(board("abc", "board7"), deleted, coordinateOf)).toBe(true);
  });

  it("matches by coordinate", () => {
    const deleted = collectDeleted([deletion([["a", `30301:${PUBKEY}:board7`]])]);
    expect(isDeleted(board("xyz", "board7"), deleted, coordinateOf)).toBe(true);
  });

  it("leaves unrelated events alone", () => {
    const deleted = collectDeleted([deletion([["e", "other"]])]);
    expect(isDeleted(board("abc", "board7"), deleted, coordinateOf)).toBe(false);
  });
});
```

- [ ] **Step 2: Run the deletion tests to verify they fail**

Run: `pnpm vitest run src/discovery/deletions.test.ts`
Expected: FAIL — `Failed to resolve import "./deletions"`.

- [ ] **Step 3: Write the deletion filter**

Create `src/discovery/deletions.ts`:

```ts
import type { Event } from "nostr-tools";

export interface DeletedSet {
  ids: Set<string>;
  coordinates: Set<string>;
}

/** Index kind-5 events by what they claim to delete (NIP-09 `e` and `a` tags). */
export function collectDeleted(deletionEvents: Event[]): DeletedSet {
  const ids = new Set<string>();
  const coordinates = new Set<string>();
  for (const event of deletionEvents) {
    for (const tag of event.tags) {
      if (tag[0] === "e" && tag[1]) ids.add(tag[1]);
      if (tag[0] === "a" && tag[1]) coordinates.add(tag[1]);
    }
  }
  return { ids, coordinates };
}

/**
 * Relays honour NIP-09 inconsistently, and addressable events keep resolving
 * after deletion, so every fetch path filters client-side.
 */
export function isDeleted(
  event: Event,
  deleted: DeletedSet,
  coordinateOf: (event: Event) => string,
): boolean {
  if (deleted.ids.has(event.id)) return true;
  return deleted.coordinates.has(coordinateOf(event));
}
```

- [ ] **Step 4: Run the deletion tests to verify they pass**

Run: `pnpm vitest run src/discovery/deletions.test.ts`
Expected: PASS, 5 tests.

- [ ] **Step 5: Write the failing relay tests**

Create `src/discovery/relays.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import { normalizeRelayList, normalizeRelayUrl, parseRelayList } from "./relays";

describe("normalizeRelayUrl", () => {
  it("appends a trailing slash", () => {
    expect(normalizeRelayUrl("wss://relay.example")).toBe("wss://relay.example/");
  });

  it("lowercases the host and keeps the path", () => {
    expect(normalizeRelayUrl("wss://Relay.Example/Inbox")).toBe("wss://relay.example/Inbox");
  });

  it("leaves an already-normal url alone", () => {
    expect(normalizeRelayUrl("wss://relay.example/")).toBe("wss://relay.example/");
  });

  it("returns the input unchanged when it is not a url", () => {
    expect(normalizeRelayUrl("not a url")).toBe("not a url");
  });
});

describe("normalizeRelayList", () => {
  it("dedupes after normalization and preserves order", () => {
    expect(normalizeRelayList(["wss://a.example", "wss://A.example/", "wss://b.example"])).toEqual([
      "wss://a.example/",
      "wss://b.example/",
    ]);
  });
});

describe("parseRelayList", () => {
  function relayList(tags: string[][]): Event {
    return {
      id: "r1",
      pubkey: "a".repeat(64),
      created_at: 1,
      kind: 10002,
      tags,
      content: "",
      sig: "",
    } as Event;
  }

  it("splits read and write markers", () => {
    const parsed = parseRelayList(
      relayList([
        ["r", "wss://in.example", "read"],
        ["r", "wss://out.example", "write"],
      ]),
    );
    expect(parsed.read).toEqual(["wss://in.example/"]);
    expect(parsed.write).toEqual(["wss://out.example/"]);
  });

  it("treats an unmarked relay as both read and write", () => {
    const parsed = parseRelayList(relayList([["r", "wss://both.example"]]));
    expect(parsed.read).toEqual(["wss://both.example/"]);
    expect(parsed.write).toEqual(["wss://both.example/"]);
  });

  it("returns empty lists for an empty event", () => {
    const parsed = parseRelayList(relayList([]));
    expect(parsed.read).toEqual([]);
    expect(parsed.write).toEqual([]);
  });
});
```

- [ ] **Step 6: Run the relay tests to verify they fail**

Run: `pnpm vitest run src/discovery/relays.test.ts`
Expected: FAIL — `Failed to resolve import "./relays"`.

- [ ] **Step 7: Write the relay helpers**

Create `src/discovery/relays.ts`:

```ts
import type { Event } from "nostr-tools";

/**
 * NIP-65 relay lists (kind 10002). Publishing to the wrong relays is the single
 * most common cause of "my collaborator can't see the board" — kanbanstr does no
 * NIP-65 handling at all (kanban/docs/03-kanbanstr-review.md §3).
 */

export function normalizeRelayUrl(url: string): string {
  try {
    const parsed = new URL(url);
    parsed.hostname = parsed.hostname.toLowerCase();
    if (parsed.pathname === "") parsed.pathname = "/";
    return parsed.toString();
  } catch {
    return url;
  }
}

export function normalizeRelayList(urls: string[]): string[] {
  const seen = new Set<string>();
  const result: string[] = [];
  for (const url of urls) {
    const normalized = normalizeRelayUrl(url);
    if (seen.has(normalized)) continue;
    seen.add(normalized);
    result.push(normalized);
  }
  return result;
}

export function parseRelayList(event: Event): { read: string[]; write: string[] } {
  const read: string[] = [];
  const write: string[] = [];

  for (const tag of event.tags) {
    if (tag[0] !== "r" || !tag[1]) continue;
    const marker = tag[2];
    if (marker === "read") read.push(tag[1]);
    else if (marker === "write") write.push(tag[1]);
    else {
      read.push(tag[1]);
      write.push(tag[1]);
    }
  }

  return { read: normalizeRelayList(read), write: normalizeRelayList(write) };
}
```

- [ ] **Step 8: Run the relay tests to verify they pass**

Run: `pnpm vitest run src/discovery/relays.test.ts`
Expected: PASS, 8 tests.

- [ ] **Step 9: Commit**

```bash
git add packages/kanban-sdk/src/discovery
git commit -m "feat(kanban-sdk): NIP-09 deletion filtering and NIP-65 relay resolution"
```

---

### Task 7: Board service

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/services/boards.ts`
- Create: `common-packages/packages/kanban-sdk/test/helpers.ts`
- Test: `common-packages/packages/kanban-sdk/src/services/boards.test.ts`

**Interfaces:**
- Consumes: `KanbanCtx`, `BoardNotFoundError` (Task 1); `newestByDTag`, `nextCreatedAt` (Task 2); `parsePublicBoard`, `buildPublicBoardTags`, `mergeTags`, `boardCoordinate`, `BOARD_MANAGED_TAGS` (Task 4); `collectDeleted`, `isDeleted` (Task 6).
- Produces:
  - `createBoard(ctx: KanbanCtx, draft: BoardDraft): Promise<KanbanBoard>`
  - `updateBoard(ctx: KanbanCtx, board: KanbanBoard, changes: Partial<BoardDraft>): Promise<KanbanBoard>`
  - `fetchBoards(ctx: KanbanCtx, params: { authors?: string[]; maintainedBy?: string }): Promise<KanbanBoard[]>`
  - `fetchBoardByCoordinate(ctx: KanbanCtx, coordinate: string): Promise<KanbanBoard | null>`
- Test helper produced: `FakeRuntime` (implements `NostrRuntime` over an in-memory store), `fakeSigner(secretKey?: Uint8Array): KanbanSigner`, `makeCtx(overrides?): KanbanCtx`.

- [ ] **Step 1: Write the test helpers**

Create `test/helpers.ts`:

```ts
import {
  finalizeEvent,
  generateSecretKey,
  getPublicKey,
  nip44,
  type Event,
  type EventTemplate,
  type Filter,
} from "nostr-tools";

import type { KanbanCtx, KanbanSigner, NostrRuntime, SubscriptionHandle } from "../src/contracts";

/** In-memory relay. Applies NIP-01 addressable replacement so tests see real behaviour. */
export class FakeRuntime implements NostrRuntime {
  readonly published: Event[] = [];
  private readonly store = new Map<string, Event>();

  private keyOf(event: Event): string {
    if (event.kind >= 30000 && event.kind < 40000) {
      const dTag = event.tags.find((t) => t[0] === "d")?.[1] ?? "";
      return `${event.kind}:${event.pubkey}:${dTag}`;
    }
    return event.id;
  }

  seed(event: Event): void {
    this.store.set(this.keyOf(event), event);
  }

  async publish(_relays: string[], event: Event): Promise<void> {
    this.published.push(event);
    const key = this.keyOf(event);
    const existing = this.store.get(key);
    if (
      !existing ||
      event.created_at > existing.created_at ||
      (event.created_at === existing.created_at && event.id < existing.id)
    ) {
      this.store.set(key, event);
    }
  }

  async querySync(_relays: string[], filter: Filter): Promise<Event[]> {
    return [...this.store.values()].filter((event) => {
      if (filter.kinds && !filter.kinds.includes(event.kind)) return false;
      if (filter.authors && !filter.authors.includes(event.pubkey)) return false;
      if (filter.ids && !filter.ids.includes(event.id)) return false;
      for (const [key, values] of Object.entries(filter)) {
        if (!key.startsWith("#")) continue;
        const letter = key.slice(1);
        const tagValues = event.tags.filter((t) => t[0] === letter).map((t) => t[1]);
        if (!(values as string[]).some((v) => tagValues.includes(v))) return false;
      }
      return true;
    });
  }

  subscribe(): SubscriptionHandle {
    return { unsub: () => {} };
  }
}

export function fakeSigner(secretKey = generateSecretKey()): KanbanSigner {
  const pubkey = getPublicKey(secretKey);
  return {
    getPublicKey: async () => pubkey,
    signEvent: async (template: EventTemplate) => finalizeEvent(template, secretKey) as Event,
    nip44Encrypt: async (peer: string, plaintext: string) =>
      nip44.v2.encrypt(plaintext, nip44.v2.utils.getConversationKey(secretKey, peer)),
    nip44Decrypt: async (peer: string, ciphertext: string) =>
      nip44.v2.decrypt(ciphertext, nip44.v2.utils.getConversationKey(secretKey, peer)),
  };
}

export function makeCtx(
  overrides: { signer?: KanbanSigner; runtime?: NostrRuntime } = {},
): KanbanCtx & { runtime: FakeRuntime } {
  const runtime = (overrides.runtime as FakeRuntime) ?? new FakeRuntime();
  const signer = overrides.signer ?? fakeSigner();
  return {
    getSigner: async () => signer,
    runtime,
    relays: ["wss://test.relay/"],
  } as KanbanCtx & { runtime: FakeRuntime };
}
```

- [ ] **Step 2: Write the failing board-service tests**

Create `src/services/boards.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { FakeRuntime, fakeSigner, makeCtx } from "../../test/helpers";
import { createBoard, fetchBoardByCoordinate, fetchBoards, updateBoard } from "./boards";

describe("createBoard", () => {
  it("publishes a 30301 with a random d tag and returns the board", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, {
      title: "Roadmap",
      columns: [{ id: "c1", name: "To Do", order: 0 }],
    });

    expect(board.title).toBe("Roadmap");
    expect(board.id).toMatch(/^[0-9a-f-]{36}$/);
    expect(ctx.runtime.published).toHaveLength(1);
    expect(ctx.runtime.published[0].kind).toBe(30301);
  });

  it("round-trips through fetchBoardByCoordinate", async () => {
    const ctx = makeCtx();
    const created = await createBoard(ctx, { title: "Roadmap", columns: [] });
    const fetched = await fetchBoardByCoordinate(ctx, `30301:${created.pubkey}:${created.id}`);
    expect(fetched?.title).toBe("Roadmap");
  });
});

describe("updateBoard", () => {
  it("preserves tags the model does not know about", async () => {
    const ctx = makeCtx();
    const created = await createBoard(ctx, { title: "Roadmap", columns: [] });

    // Simulate a tag written by some other client.
    const stored = ctx.runtime.published[0];
    ctx.runtime.seed({ ...stored, tags: [...stored.tags, ["experimental", "keep me"]] });

    const reloaded = await fetchBoardByCoordinate(ctx, `30301:${created.pubkey}:${created.id}`);
    const updated = await updateBoard(ctx, reloaded!, { title: "Roadmap v2" });

    const published = ctx.runtime.published.at(-1)!;
    expect(published.tags).toContainEqual(["title", "Roadmap v2"]);
    expect(published.tags).toContainEqual(["experimental", "keep me"]);
    expect(updated.title).toBe("Roadmap v2");
  });

  it("preserves the nozap flag across an unrelated edit", async () => {
    const ctx = makeCtx();
    const created = await createBoard(ctx, { title: "X", columns: [], noZap: true });
    const updated = await updateBoard(ctx, created, { title: "Y" });
    expect(updated.noZap).toBe(true);
    expect(ctx.runtime.published.at(-1)!.tags).toContainEqual(["nozap"]);
  });

  it("forces a strictly newer created_at", async () => {
    const ctx = makeCtx();
    const created = await createBoard(ctx, { title: "X", columns: [] });
    const updated = await updateBoard(ctx, created, { title: "Y" });
    expect(updated.createdAt).toBeGreaterThan(created.createdAt);
  });
});

describe("fetchBoards", () => {
  it("filters by author", async () => {
    const alice = fakeSigner();
    const runtime = new FakeRuntime();
    const aliceCtx = makeCtx({ signer: alice, runtime });
    const bobCtx = makeCtx({ runtime });

    await createBoard(aliceCtx, { title: "Alice board", columns: [] });
    await createBoard(bobCtx, { title: "Bob board", columns: [] });

    const boards = await fetchBoards(aliceCtx, { authors: [await alice.getPublicKey()] });
    expect(boards.map((b) => b.title)).toEqual(["Alice board"]);
  });

  it("filters by maintainer", async () => {
    const runtime = new FakeRuntime();
    const ctx = makeCtx({ runtime });
    const maintainer = "c".repeat(64);

    await createBoard(ctx, { title: "Shared", columns: [], maintainers: [maintainer] });
    await createBoard(ctx, { title: "Solo", columns: [] });

    const boards = await fetchBoards(ctx, { maintainedBy: maintainer });
    expect(boards.map((b) => b.title)).toEqual(["Shared"]);
  });

  it("returns one entry per board even with several versions on the relay", async () => {
    const ctx = makeCtx();
    const created = await createBoard(ctx, { title: "V1", columns: [] });
    await updateBoard(ctx, created, { title: "V2" });

    const boards = await fetchBoards(ctx, { authors: [created.pubkey] });
    expect(boards).toHaveLength(1);
    expect(boards[0].title).toBe("V2");
  });
});
```

- [ ] **Step 3: Run the tests to verify they fail**

Run: `pnpm vitest run src/services/boards.test.ts`
Expected: FAIL — `Failed to resolve import "./boards"`.

- [ ] **Step 4: Write the board service**

Create `src/services/boards.ts`:

```ts
import type { Event, Filter } from "nostr-tools";

import {
  BOARD_MANAGED_TAGS,
  boardCoordinate,
  buildPublicBoardTags,
  mergeTags,
  parsePublicBoard,
} from "../codec/board";
import { BoardNotFoundError, type KanbanCtx } from "../contracts";
import { collectDeleted, isDeleted } from "../discovery/deletions";
import { newestByDTag, nextCreatedAt } from "../discovery/dedupe";
import { KANBAN_KINDS } from "../kinds";
import type { BoardDraft, KanbanBoard } from "../types";

const coordinateOf = (event: Event): string =>
  `${event.kind}:${event.pubkey}:${event.tags.find((t) => t[0] === "d")?.[1] ?? ""}`;

async function publishBoard(
  ctx: KanbanCtx,
  tags: string[][],
  createdAt: number,
): Promise<KanbanBoard> {
  const signer = await ctx.getSigner();
  const signed = await signer.signEvent({
    kind: KANBAN_KINDS.publicBoard,
    created_at: createdAt,
    tags,
    content: "",
  });
  await ctx.runtime.publish(ctx.relays, signed);

  const board = parsePublicBoard(signed);
  if (!board) throw new Error("Built an unparseable board event");
  return board;
}

export async function createBoard(ctx: KanbanCtx, draft: BoardDraft): Promise<KanbanBoard> {
  const dTag = crypto.randomUUID();
  return publishBoard(ctx, buildPublicBoardTags(draft, dTag), nextCreatedAt());
}

export async function updateBoard(
  ctx: KanbanCtx,
  board: KanbanBoard,
  changes: Partial<BoardDraft>,
): Promise<KanbanBoard> {
  const draft: BoardDraft = {
    title: changes.title ?? board.title,
    description: changes.description ?? board.description,
    columns: changes.columns ?? board.columns,
    maintainers: changes.maintainers ?? board.maintainers,
    noZap: changes.noZap ?? board.noZap,
  };

  // Merge, never rebuild: unknown tags written by other clients must survive.
  const tags = mergeTags(board.rawTags, buildPublicBoardTags(draft, board.id), BOARD_MANAGED_TAGS);
  return publishBoard(ctx, tags, nextCreatedAt(board.createdAt));
}

async function resolveBoards(ctx: KanbanCtx, filter: Filter): Promise<KanbanBoard[]> {
  const events = await ctx.runtime.querySync(ctx.relays, filter);
  if (events.length === 0) return [];

  const deletions = await ctx.runtime.querySync(ctx.relays, {
    kinds: [KANBAN_KINDS.deletion],
    authors: [...new Set(events.map((e) => e.pubkey))],
  });
  const deleted = collectDeleted(deletions);

  // Group by author too: `d` alone is not unique across authors.
  const byAuthor = new Map<string, Event[]>();
  for (const event of events) {
    if (isDeleted(event, deleted, coordinateOf)) continue;
    const bucket = byAuthor.get(event.pubkey) ?? [];
    bucket.push(event);
    byAuthor.set(event.pubkey, bucket);
  }

  const boards: KanbanBoard[] = [];
  for (const bucket of byAuthor.values()) {
    for (const event of newestByDTag(bucket).values()) {
      const board = parsePublicBoard(event);
      if (board) boards.push(board);
    }
  }
  return boards;
}

export async function fetchBoards(
  ctx: KanbanCtx,
  params: { authors?: string[]; maintainedBy?: string } = {},
): Promise<KanbanBoard[]> {
  const filter: Filter = { kinds: [KANBAN_KINDS.publicBoard] };
  if (params.authors) filter.authors = params.authors;
  if (params.maintainedBy) filter["#p"] = [params.maintainedBy];
  return resolveBoards(ctx, filter);
}

export async function fetchBoardByCoordinate(
  ctx: KanbanCtx,
  coordinate: string,
): Promise<KanbanBoard | null> {
  const [kind, pubkey, dTag] = coordinate.split(":");
  if (Number.parseInt(kind, 10) !== KANBAN_KINDS.publicBoard || !pubkey || !dTag) {
    throw new BoardNotFoundError(coordinate);
  }
  const boards = await resolveBoards(ctx, {
    kinds: [KANBAN_KINDS.publicBoard],
    authors: [pubkey],
    "#d": [dTag],
  });
  return boards[0] ?? null;
}

export { boardCoordinate };
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run src/services/boards.test.ts`
Expected: PASS, 7 tests.

- [ ] **Step 6: Commit**

```bash
git add packages/kanban-sdk/src/services packages/kanban-sdk/test/helpers.ts
git commit -m "feat(kanban-sdk): board service with merge-preserving updates"
```

---

### Task 8: Card service

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/services/cards.ts`
- Test: `common-packages/packages/kanban-sdk/src/services/cards.test.ts`

**Interfaces:**
- Consumes: `KanbanCtx`, `NotAMaintainerError` (Task 1); `newestByDTag`, `nextCreatedAt` (Task 2); `computeRank` (Task 3); `boardCoordinate`, `mergeTags` (Task 4); `buildPublicCardTags`, `parsePublicCard`, `CARD_MANAGED_TAGS` (Task 5); `collectDeleted`, `isDeleted` (Task 6).
- Produces:
  - `createCard(ctx: KanbanCtx, board: KanbanBoard, draft: CardDraft): Promise<KanbanCard>`
  - `updateCard(ctx: KanbanCtx, board: KanbanBoard, card: KanbanCard, changes: Partial<CardDraft>): Promise<KanbanCard>`
  - `moveCard(ctx, board, cards: KanbanCard[], cardId: string, targetStatus: string, targetIndex: number): Promise<KanbanCard>`
  - `fetchCards(ctx: KanbanCtx, board: KanbanBoard): Promise<KanbanCard[]>`
  - `canEditCards(board: KanbanBoard, pubkey: string): boolean`

- [ ] **Step 1: Write the failing tests**

Create `src/services/cards.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { FakeRuntime, fakeSigner, makeCtx } from "../../test/helpers";
import { createBoard } from "./boards";
import { canEditCards, createCard, fetchCards, moveCard, updateCard } from "./cards";

describe("canEditCards", () => {
  it("allows the board owner", () => {
    const board = { pubkey: "a".repeat(64), maintainers: [] } as never;
    expect(canEditCards(board, "a".repeat(64))).toBe(true);
  });

  it("allows a listed maintainer", () => {
    const board = { pubkey: "a".repeat(64), maintainers: ["b".repeat(64)] } as never;
    expect(canEditCards(board, "b".repeat(64))).toBe(true);
  });

  it("rejects a stranger", () => {
    const board = { pubkey: "a".repeat(64), maintainers: [] } as never;
    expect(canEditCards(board, "c".repeat(64))).toBe(false);
  });
});

describe("createCard", () => {
  it("publishes a 30302 pointing at the board", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, {
      title: "B",
      columns: [{ id: "c1", name: "To Do", order: 0 }],
    });
    const card = await createCard(ctx, board, { title: "Task", status: "To Do" });

    expect(card.title).toBe("Task");
    expect(card.boardCoordinate).toBe(`30301:${board.pubkey}:${board.id}`);
    expect(ctx.runtime.published.at(-1)!.kind).toBe(30302);
  });

  it("rejects a non-maintainer", async () => {
    const runtime = new FakeRuntime();
    const ownerCtx = makeCtx({ runtime });
    const board = await createBoard(ownerCtx, { title: "B", columns: [] });

    const strangerCtx = makeCtx({ signer: fakeSigner(), runtime });
    await expect(createCard(strangerCtx, board, { title: "Nope" })).rejects.toThrow(
      /not a maintainer/i,
    );
  });

  it("assigns an increasing rank within a column", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, { title: "B", columns: [] });
    const first = await createCard(ctx, board, { title: "A", status: "To Do" });
    const second = await createCard(ctx, board, { title: "B", status: "To Do" });
    expect(second.rank).toBeGreaterThan(first.rank);
  });
});

describe("updateCard", () => {
  it("preserves tracker tags across an edit", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, { title: "B", columns: [] });
    const card = await createCard(ctx, board, { title: "Tracker", status: "To Do" });

    // Simulate a tracker card: add k/refs tags outside the model.
    const stored = ctx.runtime.published.at(-1)!;
    ctx.runtime.seed({
      ...stored,
      tags: [...stored.tags, ["k", "1621"], ["e", "f".repeat(64)]],
    });
    const reloaded = (await fetchCards(ctx, board)).find((c) => c.id === card.id)!;

    const updated = await updateCard(ctx, board, reloaded, { status: "Done" });

    const published = ctx.runtime.published.at(-1)!;
    expect(published.tags).toContainEqual(["k", "1621"]);
    expect(published.tags).toContainEqual(["e", "f".repeat(64)]);
    expect(updated.status).toBe("Done");
    expect(updated.trackedKind).toBe(1621);
  });

  it("forces a strictly newer created_at", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, { title: "B", columns: [] });
    const card = await createCard(ctx, board, { title: "T" });
    const updated = await updateCard(ctx, board, card, { title: "T2" });
    expect(updated.createdAt).toBeGreaterThan(card.createdAt);
  });
});

describe("moveCard", () => {
  it("changes status and ranks the card between its new neighbours", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, { title: "B", columns: [] });
    const a = await createCard(ctx, board, { title: "A", status: "Done" });
    const b = await createCard(ctx, board, { title: "B", status: "Done" });
    const mover = await createCard(ctx, board, { title: "M", status: "To Do" });

    const moved = await moveCard(ctx, board, [a, b, mover], mover.id, "Done", 1);

    expect(moved.status).toBe("Done");
    expect(moved.rank).toBeGreaterThan(a.rank);
    expect(moved.rank).toBeLessThan(b.rank);
  });
});

describe("fetchCards", () => {
  it("drops cards published by non-maintainers", async () => {
    const runtime = new FakeRuntime();
    const ownerCtx = makeCtx({ runtime });
    const board = await createBoard(ownerCtx, { title: "B", columns: [] });
    await createCard(ownerCtx, board, { title: "Legit" });

    // A stranger publishes a card carrying the board's `a` tag directly.
    const stranger = fakeSigner();
    const strangerEvent = await stranger.signEvent({
      kind: 30302,
      created_at: Math.floor(Date.now() / 1000),
      tags: [
        ["d", "intruder"],
        ["title", "Spam"],
        ["a", `30301:${board.pubkey}:${board.id}`],
      ],
      content: "",
    });
    await runtime.publish([], strangerEvent);

    const cards = await fetchCards(ownerCtx, board);
    expect(cards.map((c) => c.title)).toEqual(["Legit"]);
  });

  it("returns cards sorted by rank", async () => {
    const ctx = makeCtx();
    const board = await createBoard(ctx, { title: "B", columns: [] });
    await createCard(ctx, board, { title: "First", status: "To Do" });
    await createCard(ctx, board, { title: "Second", status: "To Do" });

    const cards = await fetchCards(ctx, board);
    expect(cards.map((c) => c.title)).toEqual(["First", "Second"]);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/services/cards.test.ts`
Expected: FAIL — `Failed to resolve import "./cards"`.

- [ ] **Step 3: Write the card service**

Create `src/services/cards.ts`:

```ts
import type { Event } from "nostr-tools";

import { boardCoordinate, mergeTags } from "../codec/board";
import { CARD_MANAGED_TAGS, buildPublicCardTags, parsePublicCard } from "../codec/card";
import { computeRank } from "../codec/rank";
import { NotAMaintainerError, type KanbanCtx } from "../contracts";
import { collectDeleted, isDeleted } from "../discovery/deletions";
import { newestByDTag, nextCreatedAt } from "../discovery/dedupe";
import { KANBAN_KINDS } from "../kinds";
import type { CardDraft, KanbanBoard, KanbanCard } from "../types";

const coordinateOf = (event: Event): string =>
  `${event.kind}:${event.pubkey}:${event.tags.find((t) => t[0] === "d")?.[1] ?? ""}`;

/** NIP-100: the board author plus every `p`-tagged maintainer may write cards. */
export function canEditCards(board: KanbanBoard, pubkey: string): boolean {
  if (!pubkey) return false;
  if (board.pubkey === pubkey) return true;
  return board.maintainers.includes(pubkey);
}

async function assertMaintainer(ctx: KanbanCtx, board: KanbanBoard): Promise<string> {
  const signer = await ctx.getSigner();
  const pubkey = await signer.getPublicKey();
  if (!canEditCards(board, pubkey)) {
    throw new NotAMaintainerError(pubkey, boardCoordinate(board));
  }
  return pubkey;
}

async function publishCard(
  ctx: KanbanCtx,
  tags: string[][],
  createdAt: number,
): Promise<KanbanCard> {
  const signer = await ctx.getSigner();
  const signed = await signer.signEvent({
    kind: KANBAN_KINDS.publicCard,
    created_at: createdAt,
    tags,
    content: "",
  });
  await ctx.runtime.publish(ctx.relays, signed);

  const card = parsePublicCard(signed);
  if (!card) throw new Error("Built an unparseable card event");
  return card;
}

export async function createCard(
  ctx: KanbanCtx,
  board: KanbanBoard,
  draft: CardDraft,
): Promise<KanbanCard> {
  await assertMaintainer(ctx, board);

  let rank = draft.rank;
  if (rank === undefined) {
    const existing = await fetchCards(ctx, board);
    const columnRanks = existing
      .filter((c) => c.status === draft.status)
      .map((c) => c.rank)
      .sort((a, b) => a - b);
    rank = computeRank(columnRanks, columnRanks.length);
  }

  const tags = buildPublicCardTags(draft, crypto.randomUUID(), boardCoordinate(board), rank);
  return publishCard(ctx, tags, nextCreatedAt());
}

export async function updateCard(
  ctx: KanbanCtx,
  board: KanbanBoard,
  card: KanbanCard,
  changes: Partial<CardDraft>,
): Promise<KanbanCard> {
  await assertMaintainer(ctx, board);

  const draft: CardDraft = {
    title: changes.title ?? card.title,
    description: changes.description ?? card.description,
    status: changes.status ?? card.status,
    attachments: changes.attachments ?? card.attachments,
    assignees: changes.assignees ?? card.assignees,
    labels: changes.labels ?? card.labels,
    links: changes.links ?? card.links,
  };
  const rank = changes.rank ?? card.rank;

  // Merge, never rebuild. This is what stops an edit from stripping a tracker
  // card's k/refs tags — kanbanstr's data-loss bug §6.2.
  const tags = mergeTags(
    card.rawTags,
    buildPublicCardTags(draft, card.id, card.boardCoordinate, rank),
    CARD_MANAGED_TAGS,
  );
  return publishCard(ctx, tags, nextCreatedAt(card.createdAt));
}

export async function moveCard(
  ctx: KanbanCtx,
  board: KanbanBoard,
  cards: KanbanCard[],
  cardId: string,
  targetStatus: string,
  targetIndex: number,
): Promise<KanbanCard> {
  const card = cards.find((c) => c.id === cardId);
  if (!card) throw new Error(`Card not found in the supplied list: ${cardId}`);

  const columnRanks = cards
    .filter((c) => c.status === targetStatus && c.id !== cardId)
    .map((c) => c.rank)
    .sort((a, b) => a - b);

  return updateCard(ctx, board, card, {
    status: targetStatus,
    rank: computeRank(columnRanks, targetIndex),
  });
}

export async function fetchCards(ctx: KanbanCtx, board: KanbanBoard): Promise<KanbanCard[]> {
  const coordinate = boardCoordinate(board);
  const events = await ctx.runtime.querySync(ctx.relays, {
    kinds: [KANBAN_KINDS.publicCard],
    "#a": [coordinate],
  });
  if (events.length === 0) return [];

  const allowed = new Set([board.pubkey, ...board.maintainers]);
  const authored = events.filter((event) => allowed.has(event.pubkey));

  const deletions = await ctx.runtime.querySync(ctx.relays, {
    kinds: [KANBAN_KINDS.deletion],
    authors: [...allowed],
  });
  const deleted = collectDeleted(deletions);

  const live = authored.filter((event) => !isDeleted(event, deleted, coordinateOf));

  const cards: KanbanCard[] = [];
  for (const event of newestByDTag(live).values()) {
    const card = parsePublicCard(event);
    if (card) cards.push(card);
  }
  return cards.sort((a, b) => a.rank - b.rank);
}
```

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pnpm vitest run src/services/cards.test.ts`
Expected: PASS, 10 tests.

- [ ] **Step 5: Commit**

```bash
git add packages/kanban-sdk/src/services/cards.ts packages/kanban-sdk/src/services/cards.test.ts
git commit -m "feat(kanban-sdk): card service with maintainer filtering and rank-aware moves"
```

---

### Task 9: Facade and barrel export

**Files:**
- Create: `common-packages/packages/kanban-sdk/src/KanbanSDK.ts`
- Create: `common-packages/packages/kanban-sdk/src/index.ts`
- Test: `common-packages/packages/kanban-sdk/src/KanbanSDK.test.ts`

**Interfaces:**
- Consumes: everything from Tasks 1–8.
- Produces: `KanbanSDK` class with `createBoard`, `updateBoard`, `fetchBoards`, `fetchBoardByCoordinate`, `createCard`, `updateCard`, `moveCard`, `fetchCards`, `relays` getter, `dispose()`; `KanbanSDKOptions`; the barrel re-exporting every public building block.

- [ ] **Step 1: Write the failing facade tests**

Create `src/KanbanSDK.test.ts`:

```ts
import { describe, expect, it } from "vitest";

import { KanbanSDK } from "./KanbanSDK";
import { SignerRequiredError } from "./contracts";
import { FakeRuntime, fakeSigner } from "../test/helpers";

describe("KanbanSDK", () => {
  it("throws SignerRequiredError for writes without a signer", async () => {
    const sdk = new KanbanSDK({ runtime: new FakeRuntime() });
    await expect(sdk.createBoard({ title: "X", columns: [] })).rejects.toBeInstanceOf(
      SignerRequiredError,
    );
  });

  it("reads public boards without a signer", async () => {
    const runtime = new FakeRuntime();
    const writer = new KanbanSDK({ signer: fakeSigner(), runtime });
    const board = await writer.createBoard({ title: "Public", columns: [] });

    const reader = new KanbanSDK({ runtime });
    const boards = await reader.fetchBoards({ authors: [board.pubkey] });
    expect(boards.map((b) => b.title)).toEqual(["Public"]);
  });

  it("exposes its relay set", () => {
    const sdk = new KanbanSDK({ runtime: new FakeRuntime(), relays: ["wss://a.example/"] });
    expect(sdk.relays).toEqual(["wss://a.example/"]);
  });

  it("creates a board and a card end to end", async () => {
    const sdk = new KanbanSDK({ signer: fakeSigner(), runtime: new FakeRuntime() });
    const board = await sdk.createBoard({
      title: "Roadmap",
      columns: [{ id: "c1", name: "To Do", order: 0 }],
    });
    await sdk.createCard(board, { title: "Ship it", status: "To Do" });

    const cards = await sdk.fetchCards(board);
    expect(cards.map((c) => c.title)).toEqual(["Ship it"]);
  });
});
```

- [ ] **Step 2: Run the tests to verify they fail**

Run: `pnpm vitest run src/KanbanSDK.test.ts`
Expected: FAIL — `Failed to resolve import "./KanbanSDK"`.

- [ ] **Step 3: Write the facade**

Create `src/KanbanSDK.ts`:

```ts
import { SignerRequiredError, type KanbanCtx, type KanbanSigner, type NostrRuntime } from "./contracts";
import { normalizeRelayList } from "./discovery/relays";
import { SimplePoolRuntime } from "./runtime/pool";
import * as boards from "./services/boards";
import * as cards from "./services/cards";
import type { BoardDraft, CardDraft, KanbanBoard, KanbanCard } from "./types";

/** Cross-app default relay set. Keep any override a superset or boards stop syncing. */
export const DEFAULT_RELAYS = [
  "wss://relay.damus.io/",
  "wss://nos.lol/",
  "wss://relay.primal.net/",
];

export interface KanbanSDKOptions {
  /** Without one, reads work and every write throws SignerRequiredError. */
  signer?: KanbanSigner;
  relays?: string[];
  runtime?: NostrRuntime;
}

export class KanbanSDK {
  private readonly ctx: KanbanCtx;
  private readonly ownsRuntime: boolean;

  constructor(options: KanbanSDKOptions = {}) {
    const runtime = options.runtime ?? new SimplePoolRuntime();
    this.ownsRuntime = options.runtime === undefined;
    this.ctx = {
      getSigner: async () => {
        if (!options.signer) throw new SignerRequiredError("This operation");
        return options.signer;
      },
      runtime,
      relays: normalizeRelayList(options.relays ?? DEFAULT_RELAYS),
    };
  }

  get relays(): readonly string[] {
    return this.ctx.relays;
  }

  dispose(): void {
    if (this.ownsRuntime) this.ctx.runtime.dispose?.();
  }

  createBoard(draft: BoardDraft): Promise<KanbanBoard> {
    return boards.createBoard(this.ctx, draft);
  }

  updateBoard(board: KanbanBoard, changes: Partial<BoardDraft>): Promise<KanbanBoard> {
    return boards.updateBoard(this.ctx, board, changes);
  }

  fetchBoards(params: { authors?: string[]; maintainedBy?: string } = {}): Promise<KanbanBoard[]> {
    return boards.fetchBoards(this.ctx, params);
  }

  fetchBoardByCoordinate(coordinate: string): Promise<KanbanBoard | null> {
    return boards.fetchBoardByCoordinate(this.ctx, coordinate);
  }

  createCard(board: KanbanBoard, draft: CardDraft): Promise<KanbanCard> {
    return cards.createCard(this.ctx, board, draft);
  }

  updateCard(
    board: KanbanBoard,
    card: KanbanCard,
    changes: Partial<CardDraft>,
  ): Promise<KanbanCard> {
    return cards.updateCard(this.ctx, board, card, changes);
  }

  moveCard(
    board: KanbanBoard,
    allCards: KanbanCard[],
    cardId: string,
    targetStatus: string,
    targetIndex: number,
  ): Promise<KanbanCard> {
    return cards.moveCard(this.ctx, board, allCards, cardId, targetStatus, targetIndex);
  }

  fetchCards(board: KanbanBoard): Promise<KanbanCard[]> {
    return cards.fetchCards(this.ctx, board);
  }
}
```

- [ ] **Step 4: Write the barrel export**

Create `src/index.ts`:

```ts
export { KanbanSDK, DEFAULT_RELAYS, type KanbanSDKOptions } from "./KanbanSDK";
export { KANBAN_KINDS, type KanbanKind } from "./kinds";
export * from "./types";
export {
  BoardNotFoundError,
  NotAMaintainerError,
  SignerRequiredError,
  type KanbanCtx,
  type KanbanSigner,
  type NostrRuntime,
  type SubscriptionHandle,
} from "./contracts";
export { SimplePoolRuntime } from "./runtime/pool";

// Pure building blocks, for hosts and tests working below the facade.
export {
  BOARD_MANAGED_TAGS,
  boardCoordinate,
  buildPublicBoardTags,
  isLegacyBoard,
  mergeTags,
  parsePublicBoard,
} from "./codec/board";
export {
  CARD_MANAGED_TAGS,
  buildCardLinkTag,
  buildPublicCardTags,
  parseCardLink,
  parsePublicCard,
} from "./codec/card";
export { RANK_STEP, computeRank, needsRebalance, rebalance } from "./codec/rank";
export { newestByDTag, nextCreatedAt, supersedes } from "./discovery/dedupe";
export { collectDeleted, isDeleted, type DeletedSet } from "./discovery/deletions";
export { normalizeRelayList, normalizeRelayUrl, parseRelayList } from "./discovery/relays";
export { canEditCards } from "./services/cards";
```

- [ ] **Step 5: Run the tests to verify they pass**

Run: `pnpm vitest run src/KanbanSDK.test.ts`
Expected: PASS, 4 tests.

- [ ] **Step 6: Verify the whole package builds and typechecks**

Run: `pnpm typecheck && pnpm build && pnpm test`
Expected: no type errors; `dist/index.js`, `dist/index.cjs`, `dist/index.d.ts` emitted; all tests pass.

- [ ] **Step 7: Commit**

```bash
git add packages/kanban-sdk/src/KanbanSDK.ts packages/kanban-sdk/src/KanbanSDK.test.ts packages/kanban-sdk/src/index.ts
git commit -m "feat(kanban-sdk): public facade and barrel export"
```

---

### Task 10: kanbanstr interop suite

**Files:**
- Create: `common-packages/packages/kanban-sdk/test/kanbanstr-parsers.ts`
- Test: `common-packages/packages/kanban-sdk/test/interop.test.ts`
- Modify: `common-packages/packages/kanban-sdk/README.md` (create)

**Interfaces:**
- Consumes: `buildPublicBoardTags`, `buildPublicCardTags`, `parsePublicBoard`, `parsePublicCard` (Tasks 4–5).
- Produces: `parseBoardLikeKanbanstr(event)`, `parseCardLikeKanbanstr(event)` — ports of kanbanstr's own parsers, used as the contract.

**Why ports and not paraphrases:** kanbanstr's quirks *are* the wire contract. If this suite is written from the spec instead of from their code, it will pass while the two clients disagree. Ported from `vivganes/kanbanstr@bf36bd8`, `src/lib/stores/kanban.ts` lines 131–174 (board) and 506–561 (card).

- [ ] **Step 1: Port kanbanstr's parsers verbatim**

Create `test/kanbanstr-parsers.ts`:

```ts
import type { Event } from "nostr-tools";

/**
 * Ports of kanbanstr's own parsers at commit bf36bd8. Copied in structure, not
 * paraphrased: their behaviour is the interop contract. Do not "improve" these —
 * if they look wrong, that is the point.
 *
 * Board: src/lib/stores/kanban.ts:131-174
 * Card:  src/lib/stores/kanban.ts:506-561
 */

export function parseBoardLikeKanbanstr(event: Event) {
  const titleTag = event.tags.find((t) => t[0] === "title");
  const descTag = event.tags.find((t) => t[0] === "description");
  const dTag = event.tags.find((t) => t[0] === "d");

  const columns = event.tags
    .filter((t) => t[0] === "col")
    .map((t) => ({ id: t[1], name: t[2], order: parseInt(t[3]) }));

  const hasNoZapTag = event.tags.some((t) => t[0] === "nozap");
  const maintainers = event.tags.filter((t) => t[0] === "p").map((t) => t[1]);

  return {
    id: dTag ? dTag[1] : event.id,
    pubkey: event.pubkey,
    title: titleTag ? titleTag[1] : "Untitled Board",
    description: descTag ? descTag[1] : "",
    columns,
    isNoZapBoard: hasNoZapTag,
    maintainers,
  };
}

export function parseCardLikeKanbanstr(event: Event) {
  const titleTag = event.tags.find((t) => t[0] === "title");
  const descTag = event.tags.find((t) => t[0] === "description");
  const statusTag = event.tags.find((t) => t[0] === "s");
  const rankTag = event.tags.find((t) => t[0] === "rank");

  const attachments = event.tags.filter((t) => t[0] === "u").map((t) => t[1]);
  // Note: kanbanstr does NOT dedupe p/zap, so an assignee appears twice.
  const assignees = event.tags.filter((t) => t[0] === "p" || t[0] === "zap").map((t) => t[1]);
  const aTags = event.tags.filter((t) => t[0] === "a").map((t) => t[1]);
  const tTags = event.tags.filter((t) => t[0] === "t").map((t) => t[1]);
  const iTags = event.tags.filter((t) => t[0] === "i");
  const binnedTag = event.tags.find((t) => t[0] === "binned");

  return {
    dTag: event.tags.find((t) => t[0] === "d")?.[1] ?? event.id,
    pubkey: event.pubkey,
    title: titleTag ? titleTag[1] : "Untitled Card",
    description: descTag ? descTag[1] : "",
    status: statusTag ? statusTag[1] : "To Do",
    order: rankTag ? parseInt(rankTag[1]) : 0,
    attachments,
    assignees,
    aTags,
    tTags,
    iTags,
    binned: binnedTag ? true : false,
  };
}
```

- [ ] **Step 2: Write the failing interop tests**

Create `test/interop.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import type { Event } from "nostr-tools";

import { buildPublicBoardTags, parsePublicBoard } from "../src/codec/board";
import { buildPublicCardTags, parsePublicCard } from "../src/codec/card";
import { parseBoardLikeKanbanstr, parseCardLikeKanbanstr } from "./kanbanstr-parsers";

const PUBKEY = "a".repeat(64);
const ASSIGNEE = "b".repeat(64);
const COORD = `30301:${PUBKEY}:board7`;

function asEvent(kind: number, tags: string[][], content = ""): Event {
  return {
    id: "e".repeat(64),
    pubkey: PUBKEY,
    created_at: 1700000000,
    kind,
    tags,
    content,
    sig: "",
  } as Event;
}

describe("boards we write are readable by kanbanstr", () => {
  it("survives kanbanstr's board parser intact", () => {
    const tags = buildPublicBoardTags(
      {
        title: "Roadmap",
        description: "Q3",
        columns: [
          { id: "c1", name: "To Do", order: 0 },
          { id: "c2", name: "Done", order: 1 },
        ],
        maintainers: [ASSIGNEE],
        noZap: true,
      },
      "board7",
    );

    const theirs = parseBoardLikeKanbanstr(asEvent(30301, tags));

    expect(theirs.id).toBe("board7");
    expect(theirs.title).toBe("Roadmap");
    expect(theirs.description).toBe("Q3");
    expect(theirs.columns).toEqual([
      { id: "c1", name: "To Do", order: 0 },
      { id: "c2", name: "Done", order: 1 },
    ]);
    expect(theirs.maintainers).toEqual([ASSIGNEE]);
    expect(theirs.isNoZapBoard).toBe(true);
  });
});

describe("cards we write are readable by kanbanstr", () => {
  it("survives kanbanstr's card parser intact", () => {
    const tags = buildPublicCardTags(
      {
        title: "Ship it",
        description: "body",
        status: "To Do",
        attachments: ["https://example.com/a.png"],
        assignees: [ASSIGNEE],
        labels: ["backend"],
        links: [
          {
            boardPubkey: PUBKEY,
            boardDTag: "board9",
            cardDTag: "card9",
            forwardLabel: "is blocked by",
            reverseLabel: "blocks",
          },
        ],
      },
      "card1",
      COORD,
      15,
    );

    const theirs = parseCardLikeKanbanstr(asEvent(30302, tags));

    expect(theirs.title).toBe("Ship it");
    expect(theirs.status).toBe("To Do");
    expect(theirs.order).toBe(15);
    expect(theirs.aTags).toEqual([COORD]);
    expect(theirs.attachments).toEqual(["https://example.com/a.png"]);
    expect(theirs.tTags).toEqual(["backend"]);
    expect(theirs.iTags[0][1]).toBe(`kanban:${PUBKEY}:board9:card9`);
  });

  it("writes assignees so kanbanstr's zap routing works", () => {
    const tags = buildPublicCardTags({ title: "X", assignees: [ASSIGNEE] }, "c1", COORD, 10);
    const theirs = parseCardLikeKanbanstr(asEvent(30302, tags));
    // Their parser reads p AND zap without deduping, so both must be present.
    expect(theirs.assignees).toEqual([ASSIGNEE, ASSIGNEE]);
  });
});

describe("boards and cards kanbanstr writes are readable by us", () => {
  it("parses a kanbanstr-shaped board", () => {
    const ours = parsePublicBoard(
      asEvent(30301, [
        ["d", "board7"],
        ["title", "Roadmap"],
        ["description", "Q3"],
        ["alt", "A board titled Roadmap"],
        ["col", "c1", "To Do", "0"],
        ["p", ASSIGNEE],
        ["nozap"],
      ]),
    );

    expect(ours!.title).toBe("Roadmap");
    expect(ours!.columns).toEqual([{ id: "c1", name: "To Do", order: 0 }]);
    expect(ours!.maintainers).toEqual([ASSIGNEE]);
    expect(ours!.noZap).toBe(true);
  });

  it("parses a kanbanstr-shaped card, deduping the doubled assignee", () => {
    const ours = parsePublicCard(
      asEvent(30302, [
        ["d", "card1"],
        ["title", "Ship it"],
        ["s", "To Do"],
        ["rank", "15"],
        ["a", COORD],
        ["zap", ASSIGNEE],
        ["p", ASSIGNEE],
        ["binned"],
      ]),
    );

    expect(ours!.status).toBe("To Do");
    expect(ours!.rank).toBe(15);
    expect(ours!.assignees).toEqual([ASSIGNEE]);
    expect(ours!.binned).toBe(true);
  });

  it("parses a v0 legacy board", () => {
    const ours = parsePublicBoard(
      asEvent(
        30301,
        [
          ["d", "board0"],
          ["title", "Old"],
          ["a", "30302:x:card1"],
        ],
        JSON.stringify({
          description: "from content",
          columns: [{ id: "c1", name: "To Do", order: 0 }],
        }),
      ),
    );

    expect(ours!.legacy).toBe(true);
    expect(ours!.description).toBe("from content");
    expect(ours!.columns).toEqual([{ id: "c1", name: "To Do", order: 0 }]);
  });
});
```

- [ ] **Step 3: Run the interop tests**

Run: `pnpm vitest run test/interop.test.ts`
Expected: PASS, 6 tests. If any fail, the codec is wrong — fix the codec, never the ported parser.

- [ ] **Step 4: Write the package README**

Create `README.md`:

````markdown
# @formstr/kanban-sdk

Headless TypeScript SDK for Nostr Kanban boards ([NIP-100](https://github.com/nostr-protocol/nips/pull/1665)), byte-compatible with [kanbanstr.com](https://kanbanstr.com).

Ships no UI and owns no storage or keys — you bring a signer, call methods, and get plain objects back.

```bash
npm install @formstr/kanban-sdk
```

## Quick start

```ts
import { KanbanSDK } from "@formstr/kanban-sdk";

const sdk = new KanbanSDK({ signer });

const board = await sdk.createBoard({
  title: "Q3 Roadmap",
  columns: [
    { id: "todo", name: "To Do", order: 0 },
    { id: "doing", name: "In Progress", order: 1 },
    { id: "done", name: "Done", order: 2 },
  ],
  maintainers: [colleagueHexPubkey],
});

await sdk.createCard(board, { title: "Ship the SDK", status: "To Do" });

const cards = await sdk.fetchCards(board);
const moved = await sdk.moveCard(board, cards, cards[0].id, "In Progress", 0);

sdk.dispose();
```

Without a signer the SDK still reads public boards; writes throw `SignerRequiredError`.

## Status

Public NIP-100 boards only. Private encrypted boards (NIP-100E) land in a later release — see `kanban/docs/05-private-kanban-spec.md`.

## Notes on compatibility

- Assignees are written to **both** `p` and `zap` tags, because kanbanstr reads either and routes zaps via `zap`.
- Non-spec tags `binned`, `nozap`, and `t` are preserved and understood.
- v0 legacy boards (columns in JSON `content`) are readable but never written.
- Every edit merges into the fetched event rather than rebuilding from the model, so tags written by other clients survive a round trip.

## Development

```bash
pnpm build      # tsup → dist (ESM + CJS + d.ts)
pnpm typecheck
pnpm test
```

`test/interop.test.ts` runs this SDK's output through **ports of kanbanstr's actual parsers**, copied rather than paraphrased. If you change a wire shape, that suite is what tells you whether you just desynced the two clients.
````

- [ ] **Step 5: Run the whole suite one final time**

Run: `pnpm typecheck && pnpm build && pnpm test`
Expected: all green; 90+ tests passing across 10 files.

- [ ] **Step 6: Commit**

```bash
git add packages/kanban-sdk/test packages/kanban-sdk/README.md
git commit -m "test(kanban-sdk): kanbanstr interop suite via ported parsers"
```

---

## Done when

- `pnpm typecheck && pnpm build && pnpm test` is green in `packages/kanban-sdk`.
- A board and cards created through `KanbanSDK` render correctly in kanbanstr.com against a shared relay (manual check, worth doing once before starting Plan 2).
- The three kanbanstr defects from `kanban/docs/03-kanbanstr-review.md` are provably absent: dedupe tie-breaking is covered in Task 2, tracker-tag preservation in Task 8, `nozap` preservation in Task 7.

## Deliberately not in this plan

| Deferred to | What |
|---|---|
| **Plan 2 — private core** | View-key crypto, blinded pointer, kinds `32301`/`32302`/`32303`, board lists, invitations (`1053`/`53`), membership, `wrapKind` config |
| **Plan 3 — extensions** | Encrypted comments (`32304`), tracker-card status resolution against live git-issue events, card link fetching in both directions, `rotateBoardKey`, `LocalRelayRuntime` subpath, rank rebalancing on write |

Tracker and link **tags** are parsed and preserved by this plan's codecs — resolving them against the network is Plan 3.
