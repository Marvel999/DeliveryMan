# Rider Delivery Architecture — Conversation Changelog

> Complete record of every design decision made across all sessions.  
> Each entry maps: **what was asked → what was decided → where it lives in the master doc.**

---

## How to Read This Document

```
SESSION     Date and transcript file it came from
REQUEST     What the engineer asked
DECISION    What was designed and why
DOC §       Which section(s) of the master doc contain the output
STATUS      ✅ Done | ⚠️ Open question | 🔄 Superseded by later decision
```

---

## Session 1 — 28 Feb 2026
**Transcript:** `2026-03-01-10-37-26-rider-delivery-offline-architecture.txt`  
**Starting point:** Two uploaded PDFs — problem statement + edge case requirements

---

### [S1-01] Initial HLD + LLD

**Request:**
> "Here we have two documents. One is for the problem statement, and the second is the suggestion and edge-case need to be covered. Let's start designing the HLD and LLD of the System. Context: This needs to build on Android native."

**Decisions made:**
- Platform: Android Native, Kotlin, Jetpack Compose
- Architecture: MVI + Clean Architecture (chosen over MVVM because offline-first state complexity needs single immutable UiState)
- Local DB: Room with WAL mode (compile-time query checks, Flow-based reactive UI)
- Sync: WorkManager (survives OEM kills, constraint-aware)
- Action journal pattern established: every rider action writes state + journal entry in one atomic Room transaction
- State machine: ASSIGNED → PICKED_UP → OUT_FOR_DELIVERY → DELIVERED/FAILED/CANCELLED
- NTP offset: fetch server time once per session, apply offset to all timestamps — device clock never trusted
- Sync triggers: T1 (foreground), T2 (network), T3 (after action, ≥5 pending), T4 (15-min periodic fallback)
- Batch sync: 50 actions per API call, per-record failure handling
- 5-action restriction: show alert after 5 unsynced actions
- Image handling: WebP compression, local storage, upload on sync, delete after 2-day retention

**Output:** First architecture doc created (`rider_delivery_offline_architecture.md` v1.0)

**Master doc:** §1, §2, §3, §4, §5, §6, §10.1, §10.3, §10.4, §11, §13

**Status:** ✅ Done (merged into master doc in Session 3)

---

### [S1-02] Data Flow, Sync Mechanism & DB Cleanup Policy

**Request:**
> "Clarify the Data flow (for offline first) and sync mechanism. Also, when will we clean the database?
> 1. On UI will show only 2 days of data, which is synced, and shipment is delivered or failed or cancelled.
> 2. Ensure that when data is pending for sync or not completed in the State Machine, we never delete that data."

**Decisions made:**
- Golden rule formalised: "App never reads from the network directly. UI always reads from Room DB."
- Atomic transaction: shipment state UPDATE + journal INSERT must succeed together or both roll back
- Conflict merge on app launch: if server returns new state for a shipment with PENDING journal entries, do NOT overwrite local state — park server state in `server_pending_state` column, let sync resolve
- Cleanup rule (all three conditions required):
  1. `current_state` IN (DELIVERED, FAILED, CANCELLED)
  2. ALL journal entries for shipment = SYNCED
  3. `Max(synced_at)` older than 2 days
- Never delete if any journal entry is PENDING, IN_PROGRESS, or FAILED
- `DbCleanupWorker`: runs once daily via `PeriodicWorkRequest`
- UI shows: active tasks always; completed+synced tasks for 2 days; completed+unsynced tasks until synced

**Output:** Second doc (`offline_data_flow_sync_cleanup.md`)

**Master doc:** §5, §6.2, §6.3, §9

**Status:** ✅ Done (merged into master doc in Session 3)

---

## Session 2 — 1 Mar 2026 (morning)
**Transcript:** `2026-03-01-13-43-21-rider-delivery-offline-architecture.txt`

---

### [S2-01] List Management, Pagination & Rider-Created Pickup Flow

**Request:**
> "How will we do the list management if we have 1000 of shipments?
> 1. We can use Paging 3 for that loading list.
> 2. Use a paginated library — offset-based or cursor-based?
> Also task can be added by the delivery partner, so add that flow too."

**Decisions made:**

**Pagination:**
- Cursor-based chosen over offset-based: offset-based breaks under live DB mutations (new shipments inserted mid-scroll shift pages), cursor is stable
- Paging 3 integration with Room: `PagingSource` keyed on `shipment_id` cursor
- Three-section list structure:
  - Section 1: Sync Pending (unsynced, non-paginated, always fully loaded, max 5)
  - Section 2: Active Tasks (paginated, cursor-based)
  - Section 3: Completed Tasks (paginated, cursor-based, 2-day window)

**Rider-Created Pickup Flow:**
- Two input paths: camera barcode scan OR manual shipment ID entry
- Online path: `POST /v1/shipments/validate` → backend validates + claims → INSERT as ASSIGNED
- Offline path: INSERT locally as `UNVERIFIED_PICKUP`, journal PICKUP_CREATED as PENDING → sync resolves
- Duplicate guard: check local DB before any network call
- `UNVERIFIED_PICKUP` state added to state machine extension
- Sync resolution: if backend rejects → mark REJECTED, notify rider, never silently delete

**Master doc:** §7, §8

**Status:** ✅ Done (merged into master doc in Session 3)

---

## Session 3 — 1 Mar 2026 (afternoon)
**Transcript:** `2026-03-01-13-44-31-rider-delivery-architecture-consolidation.txt`

---

### [S3-01] Document Consolidation

**Request:**
> "MAKE A FINAL MD FILE AND KEEP ADDING ON NEW MD FILE"

**Decisions made:**
- Three separate docs merged into one living master document
- Single file: `MASTER_rider_delivery_architecture.md`
- All future additions appended as new numbered sections
- Version tracking added to header

**Master doc:** Entire document — v1.0 established

**Status:** ✅ Done

---

## Session 4 — 1 Mar 2026 (current session)
*All entries below are from the current conversation. No separate transcript file yet.*

---

### [S4-01] Storage Pressure & Emergency Cleanup

**Request:**
> Identified that the periodic DbCleanupWorker was insufficient for high-volume riders (1000+ shipments/day with images) who could fill device storage before the 2-day cleanup window.

**Decisions made:**
- Four storage pressure levels: NORMAL (>200MB), WARN (100–200MB), CRITICAL (50–100MB), SEVERE (<50MB)
- Emergency cleanup: 3-pass strategy, never touches unsynced data
  - Pass 1: Delete images for SYNCED terminal shipments
  - Pass 2: Delete full records synced >6 hours ago
  - Pass 3: Delete all SYNCED terminal records (no age floor)
- `StorageMonitor`: 60-second polling via `StatFs`, emits `StateFlow<StoragePressureLevel>`
- When nothing is cleanable: show exact space breakdown, primary CTA = "Sync Now", secondary = Android settings
- Image capture blocked at CRITICAL; pickup creation blocked at SEVERE

**Master doc:** §17

**Status:** ✅ Done

---

### [S4-02] Conflict Resolution — Complete Coverage (15 Scenarios)

**Request:**
> Full audit of all conflict scenarios — which were covered, which were partially covered, which were missing.

**Decisions made:**
- 4 already covered, 5 partially covered, 6 completely missing — all 15 now resolved
- New schema fields added: `action_journal.sequence_number`, `action_journal.device_id`, `shipments.local_version`
- New states added: `REASSIGNED_AWAY`, `DELETED_ON_SERVER`
- Key new scenarios documented:
  - Out-of-order actions: `sequence_number` ensures correct ordering; batch never splits one shipment's actions
  - Task reassigned while offline: `REASSIGNED` response → rider sees "Task reassigned" with Contact Support
  - Task deleted on server: `SHIPMENT_NOT_FOUND` → `DELETED_ON_SERVER` state, eligible for immediate cleanup
  - Optimistic lock version conflict: `local_version` field, retry if still valid, backend wins if invalid
  - Reinstall recovery: `/pending-recovery` endpoint + proactive intent snapshot upload
  - Replay attack: 3-layer defence — `action_id` idempotency + 5-min timestamp window + HMAC-SHA256 signature
  - WorkManager duplicate: 3 independent guards documented (KEEP policy, Mutex, backend idempotency)
- Decision matrix: 18 conflict types × {Layer 1, Layer 2, UI shown} (§18.17)

**Master doc:** §18

**Status:** ✅ Done

---

### [S4-03] Adaptive Sync Frequency (T4 Redesign)

**Request:**
> Fixed 15-min T4 periodic was identified as wasteful at fleet scale (40,000 wake-ups/hr on 10,000 riders, 720,000 empty wakes/day).

**Decisions made (original):**
- T4 interval driven by PENDING count: 0→NONE, 1–4→30min, 5–19→10min, 20–49→3min, 50+→1min
- HIGH/CRITICAL tiers use OneTimeWorkRequest chains (WorkManager 15-min floor workaround)
- Battery constraint relaxed at HIGH/CRITICAL tiers
- Fleet impact: 87% reduction in wake-ups, 99.7% reduction in empty wakes

**Later superseded by [S4-06]** — count is the wrong signal.

**Master doc:** §19

**Status:** 🔄 Superseded by S4-06

---

### [S4-04] QR Scan Validation

**Request:**
> "QR validation to ensure that the delivery pattern is scanning the wrong QR."

**Decisions made:**
- 8 wrong-QR scenarios identified: S1 Foreign QR, S2 Competitor QR, S3 Wrong rider, S4 Terminal state, S5 Wrong zone, S6 Damaged/partial, S7 Wrong action context, S8 Duplicate scan
- Two-layer validation architecture:
  - Layer 1 (client, instant, offline): format regex, prefix check, Luhn checksum, session duplicate set
  - Layer 2 (server, semantic): `POST /v1/qr/validate` — checks rider assignment, state, zone, action context
- Local DB fast path: if shipment already in DB, skip server call entirely
- `action_context` field: PICKUP / OUT_FOR_DELIVERY / DELIVERY
- `followRedirects(false)` on probe client for captive portal detection
- Scan audit log: `scan_audit_log` table, SHA-256 of raw QR (never raw PII), synced to backend
- UI: recoverable errors (S1, S2, S6, S8) = toast + camera auto-resumes; unrecoverable (S3, S4, S5, S7) = modal requiring explicit dismiss
- Haptic + sound feedback per result type
- Offline rule: if Layer 1 passes and DB has no answer → create as `UNVERIFIED_PICKUP`

**Master doc:** §20

**Status:** ✅ Done

---

### [S4-05] Sync Event Logging & Observability

**Request:**
> "If I already found any issue in syncing so let's log those events to backend so we get aware of issue of not synced data."

**Decisions made:**
- 15 loggable sync event types: SYNC_SUCCESS, SYNC_PARTIAL, SYNC_FAILED, RETRY_EXHAUSTED, CONFLICT, CLOCK_SKEW, VERSION_CONFLICT, STUCK_INPROGRESS, OFFLINE_BUILDUP, DB_WRITE_FAILURE, WORKER_EXHAUSTED, PAYLOAD_TOO_LARGE, SIGNATURE_INVALID, REASSIGNED, DELETED
- `sync_event_log` table: local buffer, separate from `action_journal`
- Critical design: `SyncEventUploader` is independent from `SyncManager` — if sync is broken, events still get through
- `runCatching` in `SyncEventLogger.log()` — event logging must never crash the app
- Backend endpoint: `POST /v1/sync-events` always returns 202, never 4xx/5xx that would block client
- Every event captures: device info, network type, battery level, free storage (answers ops' first questions)
- SHA-256 of raw QR in audit log — never store PII
- 7 alerting rules: RETRY_EXHAUSTED (P2, immediate), OFFLINE_BUILDUP (P3), SIGNATURE_INVALID (P1, auto-suspend), STUCK_INPROGRESS pattern (P3), DB_WRITE_FAILURE (P2), fleet SYNC_FAILED >5% (P1), WORKER_EXHAUSTED by OEM model (P3)
- Fleet dashboard: per-rider sync health view + fleet aggregate view

**Master doc:** §21

**Status:** ✅ Done

---

### [S4-06] T4 Redesign — Age-Based Instead of Count-Based

**Request:**
> "If I already have T3 and T4, then is it needed? What should be the best approach?"

**Decisions made:**
- Clarified T3 vs T4 roles:
  - T3 = sync driver ("sync what I just did")
  - T4 = stuck detector ("catch what T3/T2/T1 missed")
- T3 is blind after the fact: if sync fails mid-batch and rider stops working, no more T3 fires, PENDING entries sit indefinitely
- Count is the wrong T4 signal — it tells you how much work, not whether the system already had a fair chance
- Age of oldest PENDING entry is the right signal
- New T4 tiers (age-based):
  - No PENDING → NONE
  - PENDING, oldest <5min → NONE (grace window — T3 just ran)
  - PENDING, oldest 5–30min → 15-min periodic (BACKSTOP)
  - PENDING, oldest 30min–2hrs → 5-min OneTimeWork chain (ESCALATED)
  - PENDING, oldest >2hrs → 1-min OneTimeWork chain (URGENT) + OFFLINE_BUILDUP event
- Count used as secondary signal: if oldest >2hrs AND count >20 → relax battery constraint
- New DAO query: `oldestPendingAgeMs(nowMs)` → `Long?` (null if no PENDING entries)
- Section 6.1 and Section 19 both updated

**Master doc:** §6.1, §19.1, §19.3, §19.4

**Status:** ✅ Done (supersedes S4-03 count-based tiers)

---

### [S4-07] ConnectivityObserver — Active Reachability Probe

**Request:**
> "Also ping the Google.com server to confirm that we are online."

**Decisions made:**
- `ConnectivityManager.NetworkCallback` alone is insufficient — reports network connected, not internet reachable (captive portals, zero-balance SIMs, routing failures all return `onAvailable()`)
- Primary probe target: our own backend `/v1/ping` (not google.com)
  - Reason: google.com blocked in some regions; google.com up ≠ our backend up
  - google.com as fallback only — used to distinguish "internet down" vs "our backend down"
- `ConnectivityState` enum: OFFLINE / CONNECTED_UNVERIFIED / ONLINE / CAPTIVE_PORTAL
- `followRedirects(false)` on probe OkHttpClient — 302 = captive portal, not online
- Probe timeouts: 3s connect, 3s read, 3s write (fail fast)
- `NET_CAPABILITY_VALIDATED` fast path on Android 10+: if OS already validated, skip our probe
- Google fallback gives two-signal distinction: internet problem vs our backend problem → fires `BACKEND_UNREACHABLE` sync event
- `CONNECTED_UNVERIFIED` fills the gap between `onAvailable()` and probe completing
- `SyncOrchestrator` now gates on `ONLINE` state, not just `CONNECTED_UNVERIFIED`
- UI shows captive portal banner with deep link to Android Wi-Fi settings

**Master doc:** §10.2

**Status:** ✅ Done

---

### [S4-08] Open Question — Write Block at 5 Pending Post-Pickup

**Request:**
> "Should I block the delivery partner write operation if he has more than 5 items in pending for sync (after Pickup state)?"

**Decision:**
- Added as formal open question in §16 with full trade-off analysis
- Case FOR blocking: bounds data loss window, caps backend divergence, COD financial integrity, audit trail
- Case AGAINST blocking: riders operate in real dead zones, data is safe in Room DB, hard block creates worse workarounds, threshold 5 is arbitrary
- Proposed direction: two-tier response (not hard block):
  - Tier 1 (5–9 unsynced post-pickup): persistent banner, allow all actions
  - Tier 2 (10+ unsynced, OR 3+ unsynced COD): soft block modal with [Sync Now] + [Continue Anyway], rider can override with acknowledgement
  - Hard block: only at SEVERE storage level (§17), never on count alone
- COD deliveries flagged as needing stricter treatment (3+ threshold vs 10+ for non-COD)
- 4 questions for ops/product team to resolve before setting the threshold
- Key conclusion: do not set the threshold by intuition — base it on actual field data

**Master doc:** §16

**Status:** ⚠️ Open — requires ops field data and product decision

---

### [S4-09] Testing Plan & Phased Release

**Request:**
> "We need to add a testing plan for unit testing and E2E testing. How can we release this in phases?"

**Decisions made:**
- Three-layer testing strategy:
  - Unit tests: pure JVM, no Android framework, covers StateMachine, QrValidator, AdaptiveSyncScheduler, NtpTimeProvider, SyncEventLogger, EmergencyCleanupManager, ActionJournalDao (in-memory). Target: 80% line coverage on use case and domain classes
  - Integration tests: Android emulator, real Room (in-memory), fake Retrofit, real WorkManager test utilities. Covers offline→sync, crash recovery, partial batch failure, emergency cleanup, adaptive scheduler, captive portal detection, sequence ordering
  - E2E tests: physical devices (Xiaomi Redmi Note 11 + Samsung Galaxy A23 as P0). Maestro test runner (YAML, no Espresso boilerplate). Covers full offline delivery, wrong QR, OEM kill, captive portal, storage pressure, adaptive T4 escalation, reinstall recovery
- Device matrix: Xiaomi Redmi Note 11 (P0), Samsung Galaxy A23 (P0), OnePlus Nord CE (P1), Stock emulator API 34 (P1)
- 4-phase release:
  - Phase 1 (Week 1–2): Internal QA + 10 volunteer riders. Gate: zero P0 crashes, sync >95%
  - Phase 2 (Week 3–4): 500 riders, single city. Gate: RETRY_EXHAUSTED <0.1%, no P0/P1
  - Phase 3 (Week 5–8): 10%→25%→50% fleet. Automated rollback if fleet SYNC_FAILED >5% in 10-min window
  - Phase 4 (Week 9+): 100% fleet. Continuous monitoring via §21 dashboard
- 4 feature flags for risky decisions: `offline_write_block_enabled` (default off), `qr_semantic_validation_enabled` (default on), `adaptive_sync_enabled` (default on), `hmac_signature_enabled` (default off)
- Rollback: disable flag first (<5min), then force-update if flag insufficient. Root cause via `action_journal` + `sync_event_log` audit trail

**Master doc:** §22

**Status:** ✅ Done

---

### [S4-10] Document Cleanup

**Request:**
> "I can see that this document is too big. Can we remove duplicate points?"

**Removed:**
- Section 12 (Edge Case table, 17 rows) — all detail already in §17–21. Replaced with 6-row index pointing to the real sections
- Section 14 (Observability bullets) — shallow subset of §21. Collapsed to 3-line summary + pointer
- Section 15 (AI Usage) — journal entry, not architecture. Removed entirely
- Stale `Version: 4.0` footer orphaned mid-document between §17 and §19
- `5-action offline restriction` trade-off row in §13 — superseded by §16 open question. Replaced with pointer

**Net result:** ~50 lines removed, zero architectural content lost

**Status:** ✅ Done

---

## Audit: Have We Done Everything?

Cross-reference of every feature mentioned in the original problem statement PDFs vs master doc coverage:

| Requirement | Covered | Where |
|---|---|---|
| Offline-first workflow (full CRUD offline) | ✅ | §5, §6 |
| State machine with strict transitions | ✅ | §4 |
| Sync when connectivity returns | ✅ | §6 |
| 10,000+ riders at scale | ✅ | §19 (fleet analysis) |
| 1,000+ shipments per rider | ✅ | §8 (pagination), §17 (storage) |
| DB cleanup — 2-day rule | ✅ | §9 |
| Never delete unsynced data | ✅ | §9.1, §9.2 |
| OEM background kill (Xiaomi/Samsung) | ✅ | §6.6 |
| Barcode scanner + manual fallback | ✅ | §7, §20 |
| Rider-created pickup task | ✅ | §7 |
| Image capture and upload | ✅ | §10.4 |
| NTP / clock tamper prevention | ✅ | §10.1 |
| Conflict resolution | ✅ | §18 (15 scenarios) |
| Idempotent API | ✅ | §11 |
| Storage pressure handling | ✅ | §17 |
| QR wrong-parcel detection | ✅ | §20 |
| Sync observability / telemetry | ✅ | §21 |
| Internet reachability (captive portal) | ✅ | §10.2 |
| Adaptive sync (battery efficiency) | ✅ | §19 |
| T4 stuck-state detection | ✅ | §6.1, §19 |
| Testing plan | ✅ | §22 |
| Phased release strategy | ✅ | §22.5 |
| Write-block threshold (5 pending) | ⚠️ | §16 — open question, needs ops data |
| COD-specific restrictions | ⚠️ | §16 — proposed, not finalised |

---

## What Remains Open

| # | Question | Blocking? | Next step |
|---|---|---|---|
| 1 | Offline write-block threshold — hard block, soft block, or warn only? | No — current behaviour is warn-only | Get field incident data from ops team |
| 2 | COD delivery stricter threshold (3 vs 10 unsynced) | No | Product + finance team input needed |
| 3 | HMAC key rotation policy | No — flag off by default | Security team review before enabling |
| 4 | Competitor QR prefix list — who maintains it, how often refreshed? | No — falls back to FOREIGN_QR if stale | Ops team to provide known prefix list |
| 5 | Maximum offline window SLA — is there a contractual POD timestamp accuracy requirement? | No | Legal / ops team to check contracts |
