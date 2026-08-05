---
layout: default
title: My Odyssey
permalink: /my-odyssey/
nav: my-odyssey
---

# My Odyssey

**A GPS-verified US travel completion tracker. The differentiator is not the map — it is that only evidenced visits count.**

`Kotlin Multiplatform` · `Event sourcing` · `Compose Multiplatform` · `iOS / CoreLocation` · `JVM test harness`

[View source on GitHub](https://github.com/chanukyagattu/my-odyssey)

---

## The idea

Every travel app on the store is a checklist with a map behind it. You tap a
pin, the pin turns green, a counter goes up. The number means nothing, because
the number is a stored boolean and you are the one who set it.

My Odyssey inverts that. There is no boolean anywhere in the system that says
`Utah: complete`. Every percentage the app displays is a **pure fold over an
append-only visit ledger** against a **versioned canon** of must-go places:

```
fold(events, canon, user) -> Completion
```

That single function is the entire product. The UI is a view over it. Change
the canon, and the same untouched ledger reports different numbers — no
migration, because there is no derived state to migrate.

### The evidence ladder

A visit is always recorded. Whether it *counts* depends on what you can prove.

```
SELF_REPORTED  <  IMPORT_VERIFIED  <  PHOTO_VERIFIED  <  GPS_VERIFIED
     ✗                  ✓                  ✓                 ✓
             only the top three move a percentage
```

Evidence is **upgrade-only**, enforced at ingest, so a user's trust level can
never silently regress. Standing outside a geofence still writes a visit — it
just writes a self-reported one, which shows in your history and stays out of
your numbers.

### What the system refuses to accept

| Guard | Rule |
| --- | --- |
| Teleport rejection | Two visits implying travel faster than Mach 1 between centroids are rejected on ingest |
| Overlap rejection | Simultaneous visits to different places are rejected |
| Idempotency | `(userId, deviceId, sourceSeq)` alongside `eventId` — a retried offline batch with fresh event ids is still a no-op |
| Fail-closed EXIF | Missing photo metadata means no evidence upgrade, never a default-allow |
| Compensation, not deletion | Revoking a visit appends a compensating event below the original; nothing is ever overwritten |

The ledger therefore cannot contain a physically impossible history. A
300-event fuzz test asserts this over whatever survives ingest.

### Why media never leaves the device

Photos and videos are copied into the app's own container and content-addressed
by SHA-256. `MediaStore` has **no method that could send bytes to a network** —
the privacy position is structural, not a sentence in a policy document. The
ledger stores only the content address, so the log stays small and replayable
while blobs stay reclaimable: detaching frees the file and leaves the event, and
a memory whose bytes were evicted renders as a tombstone without moving a
percentage.

---

## Wireframes

Five screens. Each one exists to make a different part of the architecture
visible.

### 1 · Home — the two headline metrics

Selection context is **owned here and read everywhere else**, never written
elsewhere. The scope control is the only thing on the screen that writes it.

```
┌──────────────────────────────────────────────┐
│  My Odyssey                            ⚙︎     │
├──────────────────────────────────────────────┤
│                                              │
│   ┌─ World ─┬─ United States ─┬─ State ─┐    │
│   │         │      ▓▓▓▓▓      │         │    │   ← scope control
│   └─────────┴─────────────────┴─────────┘    │      (the one writer)
│                                              │
│        ╭───────────╮      ╭───────────╮      │
│        │   ◜◝      │      │   ◜◝      │      │
│        │    38%    │      │    24 /50 │      │
│        │   ◟◞      │      │   ◟◞      │      │
│        ╰───────────╯      ╰───────────╯      │
│        places visited     states covered     │
│                                              │
│   ── derived from 147 ledger events ──       │
│                                              │
│   Nearest uncredited                         │
│   ┌────────────────────────────────────┐     │
│   │ Arches National Park      12.4 km  │     │
│   │ Utah · dwell 20m · radius 800m     │     │
│   └────────────────────────────────────┘     │
│                                              │
├──────────────────────────────────────────────┤
│   ◉ Home   ○ Tracker   ○ Timeline   ○ Ledger │
└──────────────────────────────────────────────┘
```

### 2 · Tracker — canon places for the selected state

Dwell floors, geofence radii, live distance from your current fix, and credit
status. This is the screen that tells you what the rules actually are before
you drive four hours.

```
┌──────────────────────────────────────────────┐
│  ← Utah                          2 of 2      │
├──────────────────────────────────────────────┤
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ ✓ Delicate Arch            CREDITED    │  │
│  │   GPS_VERIFIED · 14 Jun 2025           │  │
│  │   dwell 22m / 20m required             │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ ○ Bonneville Salt Flats    UNCREDITED  │  │
│  │   SELF_REPORTED · 03 Mar 2024          │  │
│  │   you are 312 km away                  │  │
│  │   ┌──────────────────────────────────┐ │  │
│  │   │ needs GPS or photo to count      │ │  │
│  │   └──────────────────────────────────┘ │  │
│  └────────────────────────────────────────┘  │
│                                              │
└──────────────────────────────────────────────┘
```

### 3 · Capture — say what will happen before it happens

Before anything is appended the screen states the event type, the evidence tier
you have earned, and whether it will move a percentage. No surprises after the
write.

```
┌──────────────────────────────────────────────┐
│  Capture visit                          ✕    │
├──────────────────────────────────────────────┤
│                                              │
│   Delicate Arch · Utah                       │
│                                              │
│   ┌── current fix ───────────────────────┐   │
│   │  38.7436 N, 109.4993 W   ±8 m        │   │
│   │  inside geofence  ✓  (412 m / 800 m) │   │
│   │  dwell so far     22m ✓  (min 20m)   │   │
│   └──────────────────────────────────────┘   │
│                                              │
│   THIS WILL APPEND                           │
│   ┌──────────────────────────────────────┐   │
│   │ event    VISIT_RECORDED              │   │
│   │ evidence GPS_VERIFIED                │   │
│   │ effect   Utah 50% → 100%             │   │
│   └──────────────────────────────────────┘   │
│                                              │
│   [ 📷 attach photo ]   [   Append   ]       │
│                                              │
│   ─ simulator only ─                         │
│   [ stand at the place ] [ stand 40 km away ]│
└──────────────────────────────────────────────┘
```

When you are outside the geofence the same panel reads
`evidence SELF_REPORTED` / `effect none — recorded, not counted`, and the
Append button stays enabled. The app never blocks the write; it only tells the
truth about what the write is worth.

### 4 · Timeline — Memories and Explore partition the canon

W/C/S pills switch scope. **Memories** is your visits newest first, flat at
every level, including uncredited ones. **Explore** is a read-only reference
list of what remains — nothing in it is tappable, because tapping would mean
writing selection, and selection has exactly one owner.

```
┌──────────────────────────────────────────────┐
│  Timeline                     ( W ) C   S    │
├──────────────────────────────────────────────┤
│  ┌─ Memories ─┬─ Explore ─┐                  │
│  │  ▓▓▓▓▓▓▓▓  │           │                  │
│  └────────────┴───────────┘                  │
│                                              │
│  14 Jun 2025                                 │
│  ┌────────────────────────────────────────┐  │
│  │ ▣  Delicate Arch                       │  │
│  │    Utah · GPS_VERIFIED · counted       │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  03 Mar 2024                                 │
│  ┌────────────────────────────────────────┐  │
│  │ ▢  Bonneville Salt Flats               │  │
│  │    Utah · SELF_REPORTED · not counted  │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  ┌────────────────────────────────────────┐  │
│  │ ⌧  (media evicted)                     │  │
│  │    tombstone — percentage unchanged    │  │
│  └────────────────────────────────────────┘  │
└──────────────────────────────────────────────┘
```

Explore, at world scope, groups by country; at country scope, by state; at
state scope it is flat — always ordered nearest-first from your current fix.

### 5 · Ledger — the architecture, falsifiable by hand

The raw encoded log beside the fold output. Revoke a visit and watch a
compensating event get appended *below* the original rather than replacing it.
Upgrade evidence and watch a previously uncredited visit start counting.

```
┌──────────────────────────────────────────────┐
│  Ledger inspector              147 events    │
├──────────────────────────────────────────────┤
│  RAW LOG                                     │
│  ┌────────────────────────────────────────┐  │
│  │ 0141 VISIT  ut-delicate-arch  GPS  ... │  │
│  │ 0142 MEDIA  sha256:9f3c…      ATTACH   │  │
│  │ 0143 VISIT  ut-salt-flats     SELF ... │  │
│  │ 0144 REVOKE ref:0143          ← comp.  │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  FOLD OUTPUT                                 │
│  ┌────────────────────────────────────────┐  │
│  │ world          38%                     │  │
│  │ united states  24 / 50 states          │  │
│  │ utah           1 / 2 places            │  │
│  └────────────────────────────────────────┘  │
│                                              │
│  [ revoke last ]  [ upgrade evidence ]       │
│         ↑ both re-fold in place              │
└──────────────────────────────────────────────┘
```

---

## Architecture

```
engine/            Kotlin Multiplatform · no UI, no framework dependencies
  Types.kt           canon entries, lifecycle, ledger events, evidence tiers
  Ledger.kt          append-only store: idempotency, anti-cheat, plausibility
  Fold.kt            the derivation — the whole app is a view over this
  Codec.kt           line-oriented on-disk format for the log
  Repository.kt      the only write path: append, persist, re-fold
  Explore.kt         grouping, distance ordering, completes-state flag
  Exif.kt            JPEG APP1 reader — what makes PHOTO_VERIFIED real
  Media.kt           attach/detach events, the photo corroboration rule
  CanonV1.kt         100 places · 2 per state · all 50 states
  InvariantsTest     the product's promises, as executable statements

composeApp/        Compose Multiplatform UI · iOS targets only
  AppModel.kt        plain state holder — no framework ViewModel, no async
  ui/                Home, Tracker, Timeline, Capture, Ledger inspector
  iosMain/           CoreLocation delegate + one Swift-facing entry point

iosApp/            SwiftUI host: ~30 lines of Swift
```

The split is deliberate. `engine` has no Compose and no iOS dependency, so the
invariant suite runs on the JVM in about a second — no simulator, no Xcode.
Adding Android later is one line per `build.gradle.kts` and zero changes to any
logic.

### Decisions worth defending

**Completion is derived, never stored.** The suite proves it: take a ledger
where Utah is complete, publish a canon release adding a third Utah place, and
the same untouched ledger now reports Utah incomplete.

**Canon versioning over mutation.** Closures and seasonal shutdowns are
lifecycle transitions in a new immutable release. A fully suspended state is
*frozen* — removed from both sides of the state-coverage ratio, so nothing
completes for free. Reactivation restores the fold exactly, and that round trip
is a test.

**A line-oriented log instead of a serialization framework.** The on-disk format
is the system of record, so it is readable by eye, appendable without
rewriting, and free of any library that could change its encoding between
releases. A torn tail from a crash mid-write drops one line and folds the rest.

---

## Roadmap

| Milestone | Scope |
| --- | --- |
| **M0** ✓ | The engine — types, ledger, fold, codec, invariant suite |
| **M0.5** ✓ | Same engine on a phone: selection → tracker → GPS capture → recomputed percentages |
| **M1** | Segmentation: raw GPS streams → stay-points → attributed visits, with event-time watermarks and late offline dumps |
| **M2** | Deterministic simulation harness: trace replay with injected clock skew, duplication floods, spoofed teleports; byte-identical folds |
| **M3** | Real canon tooling: quarterly releases with index-rebalance discipline, place boundaries as polygons instead of centroid plus radius |
| **M4** | Planner: cheapest next increment of completion under seasonal windows |
| **M5** | Sync: the ledger is already the wire format, so the server is a merge of two append-only logs |

### Known scaffolding

`PHPickerViewController` is written but ships uncompiled at
`docs/IosPhotoPicker.kt.txt` — its Kotlin/ObjC interop signatures could not be
verified without a macOS toolchain, and an unreferenced Kotlin file still breaks
the build if it does not compile. Until then capture stages photos from
`SyntheticMediaSource`, which fabricates a real EXIF container around the
current fix and then goes through the identical hash, parser and evidence rules.
It gets no shortcuts.

`KeyValueStore` on iOS is backed by `NSUserDefaults` — correct for a log
measured in kilobytes and deliberately wrong at scale. It is a two-method
interface precisely so SQLDelight can replace it without touching a caller.