# Rider Delivery Application — Offline-First Architecture
## Master Design Document (HLD + LLD)

> **Platform:** Android Native | **Version:** 10.0
> **Last Updated:** This is the living document. All new sections are appended here.

---

## Table of Contents

1. [Problem Summary](#1-problem-summary)
2. [High-Level Design (HLD)](#2-high-level-design-hld)
   - 2.1 System Overview
   - 2.2 Offline-First Flow
   - 2.3 Tech Stack Decisions
   - 2.4 Module Structure
3. [Database Schema](#3-database-schema)
4. [State Machine](#4-state-machine)
   - 4.1 Standard Shipment States
   - 4.2 Rider-Created Pickup States Extension
5. [Offline-First Data Flow](#5-offline-first-data-flow)
   - 5.1 Golden Rule
   - 5.2 App Launch Flow
   - 5.3 Rider Action Flow (Offline or Online — Same Code Path)
   - 5.4 Conflict Merge Strategy
6. [Sync Mechanism](#6-sync-mechanism)
   - 6.1 Sync Triggers
   - 6.2 Journal Entry State Lifecycle
   - 6.3 SyncManager Execution Flow
   - 6.4 Sync Status vs Business State
   - 6.5 Startup Recovery
   - 6.6 OEM Background Kill Handling
7. [Rider-Created Pickup Task Flow](#7-rider-created-pickup-task-flow)
   - 7.1 Flow Overview
   - 7.2 Create Pickup UseCase
   - 7.3 Sync Resolution for Rider-Created Tasks
   - 7.4 UI States for Unverified Tasks
8. [List Management & Pagination](#8-list-management--pagination)
   - 8.1 Why Not Offset-Based
   - 8.2 Cursor-Based Pagination
   - 8.3 Paging 3 Implementation
   - 8.4 Three-Section List Structure
9. [Database Cleanup Policy](#9-database-cleanup-policy)
   - 9.1 The Core Rule
   - 9.2 Cleanup Eligibility Decision Tree
   - 9.3 UI vs DB Retention Distinction
   - 9.4 Cleanup Worker
   - 9.5 Full Lifecycle Example
10. [Key Component Implementations](#10-key-component-implementations)
    - 10.1 NTP Time Provider
    - 10.2 Connectivity Observer
    - 10.3 MVI Pattern — Task Feature
    - 10.4 Image Handling
11. [API Contracts](#11-api-contracts)
12. [Edge Case Handling](#12-edge-case-handling)
13. [Trade-offs](#13-trade-offs)
14. [Observability & Debugging](#14-observability--debugging)
16. [Open Questions](#16-open-questions)
17. [Storage Pressure & Emergency Cleanup](#17-storage-pressure--emergency-cleanup)
    - 17.1 The Problem
    - 17.2 Storage Pressure Levels
    - 17.3 StorageMonitor — Continuous Watching
    - 17.4 Emergency Cleanup: Decision Logic
    - 17.5 EmergencyCleanupManager Implementation
    - 17.6 What Happens When Nothing Is Cleanable
    - 17.7 UI — Progressive Pressure Communication
    - 17.8 How Emergency Cleanup Differs from Periodic Cleanup
    - 17.9 Full Pressure Scenario Walkthrough
19. [Adaptive Sync Frequency](#19-adaptive-sync-frequency)
    - 19.1 Why Fixed Periodic Is Wrong at Scale
    - 19.2 Battery & Data Cost Analysis
    - 19.3 Age-Based Adaptive Frequency Tiers
    - 19.4 AdaptiveSyncScheduler Implementation
    - 19.5 WorkManager Rescheduling Strategy
    - 19.6 Constraint Layering
    - 19.7 Interaction with Other Sync Triggers
    - 19.8 Fleet-Level Impact
    - 19.9 Trade-offs of Adaptive Sync
18. [Conflict Resolution — Complete Coverage](#18-conflict-resolution--complete-coverage)
    - 18.1 Audit Summary
    - 18.2 Duplicate Action Submission
    - 18.3 Task Already Updated on Server
    - 18.4 Out-of-Order Actions
    - 18.5 Multiple Devices Conflict
    - 18.6 Task Reassigned While Offline
    - 18.7 Task Deleted on Server
    - 18.8 Partial Batch Sync Failure
    - 18.9 Clock Skew Conflict
    - 18.10 Optimistic Lock Version Conflict
    - 18.11 Retry After App Reinstall
    - 18.12 WorkManager Duplicate Execution
    - 18.13 Network Flapping
    - 18.14 Payload Too Large
    - 18.15 Data Corruption in Local DB
    - 18.16 Security Conflict (Replay Attack)
    - 18.17 Decision Matrix
20. [QR Scan Validation](#20-qr-scan-validation)
    - 20.1 All Wrong-QR Scenarios
    - 20.2 Validation Architecture — Two Layers
    - 20.3 Layer 1: Client-Side Structural Validation
    - 20.4 Layer 2: Server-Side Semantic Validation
    - 20.5 QrValidationUseCase — Full Decision Tree
    - 20.6 Wrong QR Scenario Handling (All 8 Cases)
    - 20.7 Offline Validation Behaviour
    - 20.8 Scan Audit Log
    - 20.9 UI Feedback per Rejection Reason
    - 20.10 Decision Matrix
21. [Sync Event Logging & Observability](#21-sync-event-logging--observability)
    - 21.1 The Problem — Silent Failures
    - 21.2 What Constitutes a Sync Event
    - 21.3 SyncEvent Schema
    - 21.4 All Loggable Sync Failure Points
    - 21.5 SyncEventLogger Implementation
    - 21.6 Local Buffer — Log Before You Send
    - 21.7 Backend Ingest Endpoint
    - 21.8 SyncManager — Fully Instrumented
    - 21.9 Alerting Rules on Backend
    - 21.10 Dashboard Metrics
    - 21.11 Trade-offs
22. [Testing Plan & Phased Release](#22-testing-plan--phased-release)
    - 22.1 What to Test and Why
    - 22.2 Unit Tests
    - 22.3 Integration Tests
    - 22.4 E2E Tests

---

## 1. Problem Summary

Riders operate in areas with poor or no connectivity. The app must support full workflow execution offline — create pickup tasks, view assigned deliveries, and mark shipments as Picked Up / Delivered / Failed — and sync everything reliably when connectivity returns.

**Scale requirements:** 10,000+ concurrent riders, 1,000+ shipments per rider.

---

## 2. High-Level Design (HLD)

### 2.1 System Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Rider Android App                          │
│                                                                 │
│  ┌──────────────┐   ┌───────────────┐   ┌───────────────────┐  │
│  │   UI Layer   │   │  Domain Layer  │   │    Data Layer     │  │
│  │  (Compose)   │◄──│  (UseCases)   │◄──│  (Room + Repo)    │  │
│  └──────────────┘   └───────────────┘   └───────────────────┘  │
│                                                 │               │
│                          ┌──────────────────────▼────────────┐  │
│                          │           Sync Engine             │  │
│                          │  (SyncManager + WorkManager)      │  │
│                          └──────────────────────┬────────────┘  │
└─────────────────────────────────────────────────┼───────────────┘
                                                  │ HTTPS / REST
                                                  ▼
                             ┌──────────────────────────────────┐
                             │         Backend Services         │
                             │   (API Gateway + Micro-services) │
                             └──────────────────────────────────┘
```

### 2.2 Offline-First Flow

```
Rider Action
     │
     ▼
Write to Local Room DB ──► Update UI Immediately (optimistic)
     │
     ▼
Append to Action Journal  (sync_status = PENDING)
     │
     ▼
SyncManager  (triggered by: app launch / network change / 5 pending actions / 15-min fallback)
     │
     ├── Online? ──► Batch push to Backend (50 records/batch)
     │                    │
     │              Per-record result:
     │              ACCEPTED       ──► Mark SYNCED
     │              ALREADY_PROCESSED ──► Mark SYNCED  (idempotent duplicate)
     │              INVALID_TRANSITION ──► Mark FAILED, show conflict UI
     │              Network error  ──► Back to PENDING, exponential backoff
     │
     └── Offline? ──► WorkManager queues retry on connectivity restored
```

### 2.3 Tech Stack Decisions

| Category | Choice | Rationale |
|---|---|---|
| Language | **Kotlin** | Coroutines, sealed classes, null safety — purpose-built for async offline logic |
| Platform | **Android Native** | Full Jetpack ecosystem, OEM background task control, best runtime performance |
| UI | **Jetpack Compose** | Declarative, state-driven — ideal for complex offline/sync UI states |
| Local DB | **Room** | Type-safe SQLite, Flow reactive queries, WAL mode, compile-time query checks |
| Architecture | **MVI + Clean Architecture** | Unidirectional data flow makes offline states predictable and testable |
| Background Sync | **WorkManager** | Survives process death, OEM battery restriction aware, constraint-based |
| DI | **Dagger Hilt** | First-class Android DI, less boilerplate than vanilla Dagger |
| Navigation | **Compose Navigation** | Type-safe, ViewModel lifecycle integration |
| Build System | **Kotlin DSL + TOML version catalog** | Type-safe builds, single source of truth for dependency versions |
| Image Format | **WebP** | ~30% smaller than JPEG at same quality; stored locally, uploaded on sync |
| Notifications | **FCM + CleverTap** | FCM for transactional alerts (order cancelled), CleverTap for engagement analytics |
| Networking | **Retrofit + OkHttp** | Industry standard; interceptor chain for auth, logging, retry |
| Serialization | **Kotlin Serialization** | Native Kotlin, faster than Gson, no reflection overhead |
| Pagination | **Paging 3 + Cursor-based** | Stable under live DB mutations; see Section 8 for full rationale |

**MVI over MVVM — Why?**
An offline-first app has multi-layered state: loading, offline-pending, syncing, conflict, error, restricted. MVI enforces a single immutable `UiState` object, making it structurally impossible to have inconsistent screens. MVVM can achieve the same but requires strict discipline; MVI enforces it by design.

**Room over raw SQLite — Why?**
Compile-time query verification catches SQL errors at build time. Flow-based reactive queries mean UI auto-updates the moment DB changes — critical when the sync engine is writing in the background. Migration support handles OTA schema changes safely.

### 2.4 Module Structure

Domain-based + Feature-based hybrid. Full multi-module is over-engineering for a focused delivery app initially, but the structure scales when team grows.

```
:app
:feature:tasks          (Task listing, detail screen, task actions)
:feature:scan           (Barcode scanner, manual ID entry)
:feature:sync           (SyncManager, WorkManager workers, SyncOrchestrator)
:core:database          (Room DB, DAOs, entities, migrations)
:core:network           (Retrofit services, interceptors, API models)
:core:domain            (UseCases, domain models, state machine, business rules)
:core:common            (Utils, extensions, NTP provider, connectivity observer)
```

---

## 3. Database Schema

### `shipments` table

```sql
CREATE TABLE shipments (
    shipment_id          TEXT PRIMARY KEY,
    rider_id             TEXT NOT NULL,
    recipient_name       TEXT,                   -- nullable: populated after server validation
    address              TEXT,
    shipment_type        TEXT NOT NULL,          -- PICKUP | DELIVERY
    current_state        TEXT NOT NULL,          -- state machine enum (see Section 4)
    input_source         TEXT,                   -- SCAN | MANUAL | BACKEND_ASSIGNED
    is_cod               INTEGER NOT NULL DEFAULT 0,
    image_path           TEXT,                   -- local WebP file path
    image_uploaded       INTEGER NOT NULL DEFAULT 0,
    server_pending_state TEXT,                   -- server's state when local has unsynced actions
    created_at           INTEGER NOT NULL,       -- NTP-adjusted epoch ms
    updated_at           INTEGER NOT NULL,
    server_updated_at    INTEGER                 -- last known backend timestamp
);

CREATE INDEX idx_shipments_state    ON shipments(current_state);
CREATE INDEX idx_shipments_rider    ON shipments(rider_id);
CREATE INDEX idx_shipments_cursor   ON shipments(created_at DESC, shipment_id DESC);
```

### `action_journal` table — Sync Engine Driver

```sql
CREATE TABLE action_journal (
    action_id     TEXT PRIMARY KEY,             -- UUID v4: idempotency key sent to backend
    shipment_id   TEXT NOT NULL,
    action_type   TEXT NOT NULL,
    -- Values: PICKUP_CREATED | PICKUP | OUT_FOR_DELIVERY
    --         DELIVERED | FAILED | CANCELLED
    payload       TEXT NOT NULL,                -- JSON blob (OTP ref, image ref, metadata)
    sync_status   TEXT NOT NULL DEFAULT 'PENDING',
    -- Values: PENDING | IN_PROGRESS | SYNCED | FAILED
    retry_count   INTEGER NOT NULL DEFAULT 0,
    created_at    INTEGER NOT NULL,             -- NTP-adjusted epoch ms
    synced_at     INTEGER,                      -- when backend confirmed it
    error_message TEXT,

    FOREIGN KEY (shipment_id) REFERENCES shipments(shipment_id)
);

CREATE INDEX idx_journal_sync_status ON action_journal(sync_status);
CREATE INDEX idx_journal_shipment_id ON action_journal(shipment_id);
CREATE INDEX idx_journal_created_at  ON action_journal(created_at ASC);
```

### `ntp_offset` table

```sql
CREATE TABLE ntp_offset (
    id         INTEGER PRIMARY KEY DEFAULT 1,   -- singleton row
    offset_ms  INTEGER NOT NULL,               -- server_time - device_time
    fetched_at INTEGER NOT NULL
);
```

---

## 4. State Machine

### 4.1 Standard Shipment States

```
                      ┌──────────────────┐
                      │     ASSIGNED     │  ← Backend assigned to rider
                      └────────┬─────────┘
                               │  PICKUP action
                      ┌────────▼─────────┐
                      │    PICKED_UP     │
                      └────────┬─────────┘
                               │  OUT_FOR_DELIVERY action
                  ┌────────────▼────────────┐
                  │     OUT_FOR_DELIVERY    │
                  └────┬──────────┬─────────┘
                       │          │
                 DELIVERED      FAILED
                       │          │
              ┌────────▼──┐  ┌────▼──────┐
              │ DELIVERED │  │  FAILED   │
              └───────────┘  └───────────┘
                                   │
                            CANCELLED (from any active state, triggered by backend push)
```

**Kotlin State Machine:**

```kotlin
sealed class ShipmentState {
    object Assigned        : ShipmentState()
    object PickedUp        : ShipmentState()
    object OutForDelivery  : ShipmentState()
    object Delivered       : ShipmentState()
    object Failed          : ShipmentState()
    object Cancelled       : ShipmentState()
    // Rider-created states (Section 4.2):
    object UnverifiedPickup : ShipmentState()
    object Rejected         : ShipmentState()
    object Conflict         : ShipmentState()
}

object ShipmentStateMachine {
    private val validTransitions = mapOf(
        ShipmentState.Assigned        to setOf(ShipmentState.PickedUp, ShipmentState.Cancelled),
        ShipmentState.PickedUp        to setOf(ShipmentState.OutForDelivery, ShipmentState.Cancelled),
        ShipmentState.OutForDelivery  to setOf(ShipmentState.Delivered, ShipmentState.Failed, ShipmentState.Cancelled),
        ShipmentState.UnverifiedPickup to setOf(ShipmentState.Assigned, ShipmentState.Rejected, ShipmentState.Conflict)
    )

    fun canTransition(from: ShipmentState, to: ShipmentState): Boolean =
        validTransitions[from]?.contains(to) == true

    fun transition(from: ShipmentState, to: ShipmentState): Result<ShipmentState> =
        if (canTransition(from, to)) Result.success(to)
        else Result.failure(IllegalStateTransitionException("$from → $to not allowed"))
}
```

### 4.2 Rider-Created Pickup States Extension

```
                ┌─────────────────────┐
                │  UNVERIFIED_PICKUP  │  ← Rider scanned / typed offline
                └──────────┬──────────┘
                           │ On sync: backend validates
                           │
          ┌────────────────┼──────────────────┐
          │                │                  │
          ▼                ▼                  ▼
     ASSIGNED          REJECTED           CONFLICT
    (valid, full      (invalid ID,     (another rider
     flow continues)   show error)      already has it)
          │
          ▼
    PICKED_UP → OUT_FOR_DELIVERY → DELIVERED / FAILED / CANCELLED
```

---

## 5. Offline-First Data Flow

### 5.1 Golden Rule

> **UI always reads from Room DB. Network only writes into Room DB. Nothing reads from network directly.**

```
┌──────────────────────────────────────────────────────────────────┐
│                      DATA FLOW                                   │
│                                                                  │
│  BACKEND ──(fetch on launch)──► REPOSITORY ──► ROOM DB          │
│                                                    │             │
│  RIDER ACTIONS (offline/online) ──────────────────►│             │
│                                                    │             │
│                             UI ◄── ViewModel ◄── Flow            │
│                                  (Room reactive query)           │
│                                                    │             │
│  SYNC ENGINE ◄────────────────────────────────────►│             │
│  (action_journal PENDING entries → push to backend)│             │
└──────────────────────────────────────────────────────────────────┘
```

### 5.2 App Launch Flow

```
App Opens
    │
    ├── 1. StartupRecovery
    │       Reset any IN_PROGRESS journal entries → PENDING
    │       Reset partially uploaded images → re-queue
    │
    ├── 2. Fetch NTP offset
    │       GET /v1/time → offset_ms = server_time - device_time
    │       Store in ntp_offset table (singleton row)
    │
    ├── 3. Background fetch from backend (if online)
    │       GET /v1/shipments?rider_id=R1&updated_after=<last_sync_ts>
    │       → Merge into Room DB (see 5.4 conflict strategy)
    │       → UI auto-updates via Room Flow
    │
    └── 4. SyncManager triggered
            Push any PENDING journal entries to backend
```

### 5.3 Rider Action Flow (Same Code Path — Online or Offline)

```
Rider taps "Delivered" on SHP123
    │
    ├── STEP 1: VALIDATE
    │       StateMachine.canTransition(OUT_FOR_DELIVERY → DELIVERED)?
    │       NO  → Show error toast, stop.
    │       YES → Continue.
    │
    ├── STEP 2: ATOMIC WRITE (single Room Transaction)
    │
    │   ┌──────────────────────────────────────────────────────────┐
    │   │  BEGIN TRANSACTION                                        │
    │   │                                                          │
    │   │  UPDATE shipments                                        │
    │   │    SET current_state = 'DELIVERED',                      │
    │   │        updated_at    = ntp_time()                        │
    │   │    WHERE shipment_id = 'SHP123'                          │
    │   │                                                          │
    │   │  INSERT INTO action_journal VALUES (                     │
    │   │    action_id   = UUID(),      ← idempotency key          │
    │   │    shipment_id = 'SHP123',                               │
    │   │    action_type = 'DELIVERED',                            │
    │   │    payload     = '{"otp_verified": true}',               │
    │   │    sync_status = 'PENDING',                              │
    │   │    created_at  = ntp_time()                              │
    │   │  )                                                       │
    │   │                                                          │
    │   │  COMMIT                                                  │
    │   └──────────────────────────────────────────────────────────┘
    │       ↑ If either write fails → full rollback.
    │       ↑ App state is NEVER updated without a journal entry.
    │
    ├── STEP 3: UI UPDATES INSTANTLY
    │       Room Flow emits new data → ViewModel → Compose recomposes
    │       Rider sees "Delivered" with "Sync Pending" badge
    │
    └── STEP 4: TRIGGER SYNC (async, non-blocking)
            SyncOrchestrator.onActionRecorded()
            → Online:  fire OneTimeWorkRequest immediately
            → Offline: WorkManager retries when network available
```

### 5.4 Conflict Merge Strategy (Server Pull vs Local Unsynced)

When pulling fresh data from backend on app launch:

```
For each shipment from backend response:
    │
    ├── Not in local DB?
    │       → INSERT directly.
    │
    └── Already in local DB?
            │
            ├── Any journal entry PENDING or IN_PROGRESS for this shipment?
            │       → YES: DO NOT overwrite local state.
            │              Store server state in `server_pending_state` column.
            │              Sync engine will push local action → backend resolves.
            │              If backend rejects → conflict screen shown.
            │
            └── NO unsynced actions?
                    → Overwrite local state with server state.
                      Backend is ground truth for fully-synced data.
```

**Why this matters:** If rider marks Delivered offline and server shows Cancelled, we must preserve the rider's unsynced action. It pushes to backend, backend returns `INVALID_TRANSITION`, app shows conflict + "Contact Support" screen.

---

## 6. Sync Mechanism

### 6.1 Sync Triggers

```
┌───────────────────────────────────────────────────────────────┐
│                       SYNC TRIGGERS                           │
│                                                               │
│  T1: App comes to foreground                                  │
│      ProcessLifecycleOwner.ON_START                          │
│                                                               │
│  T2: Network becomes available                                │
│      ConnectivityManager.NetworkCallback.onAvailable()        │
│                                                               │
│  T3: After every rider action                                 │
│      Always fires. Attempts sync immediately.                 │
│      Role: "sync what I just did"                             │
│      ≥ 5 pending → also show banner to rider                  │
│                                                               │
│  T4: Stuck-state detector (NOT a sync driver)                 │
│      Only fires when T1 + T2 + T3 have all had a fair chance  │
│      and PENDING entries are still sitting unsynced.          │
│      Role: "catch what T3/T2/T1 missed"                       │
│                                                               │
│      Signal used: AGE of oldest PENDING entry, not count.     │
│      Count = how much work. Age = whether system already      │
│      had a fair chance to handle it.                          │
│                                                               │
│      0 PENDING              → NONE (cancel periodic)          │
│      PENDING, oldest < 5min → NONE (T3 just fired, wait)      │
│      PENDING, oldest > 5min → 15 min (stuck detector active)  │
│      PENDING, oldest > 30min→ 5 min  (escalate)               │
│      PENDING, oldest > 2hrs → 1 min  (urgent + event log)     │
│                                                               │
│  All triggers → WorkManager.enqueueUniqueWork("SYNC", KEEP)   │
│  KEEP policy → no duplicate sync jobs ever run in parallel    │
└───────────────────────────────────────────────────────────────┘
```

> **Why is T4 still needed if T3 exists?** T3 fires on rider actions. If sync
> fails mid-batch and the rider stops working (phone in pocket), no more actions
> = no more T3. T1 fires on foreground, T2 on network reconnect — but T2 is
> unreliable on some OEMs. T4 is the backstop for stuck PENDING entries that
> T3/T2/T1 structurally cannot reach. See Section 19 for full design.

### 6.2 Journal Entry State Lifecycle

```
  PENDING ──────────────────────────────────► IN_PROGRESS
    ▲     SyncManager picks it up                  │
    │                                              │
    │  App crash detected on restart               ├──► SYNCED ──► eligible for cleanup
    │  (reset back to PENDING)                     │
    │                                              │  network error / 5xx
    └──────────────────── FAILED ◄─────────────────┘  (retry_count < 5 → back to PENDING)
          max retries hit                              (retry_count ≥ 5 → FAILED permanently)
```

### 6.3 SyncManager Execution Flow

```kotlin
class SyncManager @Inject constructor(
    private val journalDao: ActionJournalDao,
    private val shipmentDao: ShipmentDao,
    private val apiService: DeliveryApiService
) {
    private val mutex = Mutex()

    suspend fun sync() = mutex.withLock {          // prevent parallel sync calls
        val pending = journalDao.getPending(limit = 100)
        if (pending.isEmpty()) return@withLock

        pending.chunked(50).forEach { batch ->     // 50 records per API call
            syncBatch(batch)
        }
    }

    private suspend fun syncBatch(batch: List<ActionJournalEntity>) {
        journalDao.updateStatus(batch.map { it.actionId }, SyncStatus.IN_PROGRESS)

        val response = try {
            apiService.submitBatch(batch.map { it.toRequest() })
        } catch (e: IOException) {
            // Network error: put all back to PENDING, WorkManager handles retry
            journalDao.resetToPending(batch.map { it.actionId })
            return
        }

        // Per-record resolution — one failure doesn't block others
        response.results.forEach { result ->
            when (result.status) {
                "ACCEPTED", "ALREADY_PROCESSED" -> {
                    journalDao.markSynced(result.actionId)
                }
                "INVALID_TRANSITION" -> {
                    journalDao.markFailed(result.actionId, result.error)
                    // Show conflict screen for affected shipment
                }
                else -> {
                    journalDao.incrementRetry(result.actionId)
                    // Back to PENDING if retry_count < 5, else FAILED
                }
            }
        }
    }
}
```

### 6.4 Sync Status vs Business State

These two fields are independent. Never confuse them.

| Field | Meaning |
|---|---|
| `shipments.current_state` | The business lifecycle state (ASSIGNED, DELIVERED, etc.) — what the rider sees |
| `action_journal.sync_status` | Whether a specific action has been acknowledged by the backend |

A shipment can be locally `DELIVERED` while `sync_status = PENDING`. That is the normal offline window. Both fields must be independently tracked and checked.

### 6.5 Startup Recovery

```kotlin
class StartupRecoveryUseCase @Inject constructor(
    private val journalDao: ActionJournalDao,
    private val shipmentDao: ShipmentDao
) {
    suspend fun execute() {
        // Reset stuck states from previous crash or process kill
        journalDao.resetInProgressToPending()

        // Reset partially uploaded images so they re-upload on next sync
        shipmentDao.resetPartialImageUploads()
    }
}
```

Called at app startup before any UI is shown. Ensures no journal entry is stuck in `IN_PROGRESS` from a crashed previous session.

### 6.6 OEM Background Kill Handling (Xiaomi / Redmi / Samsung)

Aggressive OEM battery optimization kills background processes. Mitigation:

```kotlin
class SyncWorker @HiltWorker constructor(
    context: Context,
    params: WorkerParameters,
    private val syncManager: SyncManager
) : CoroutineWorker(context, params) {

    override suspend fun getForegroundInfo(): ForegroundInfo =
        ForegroundInfo(SYNC_NOTIFICATION_ID, buildSyncNotification())
        // Foreground service notification keeps process alive on OEM devices

    override suspend fun doWork(): Result {
        return try {
            setForeground(getForegroundInfo())   // elevate priority
            syncManager.sync()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 5) Result.retry()  // exponential backoff
            else Result.failure()
        }
    }
}
```

WorkManager constraints ensure sync only fires when network is available and battery is not critically low:

```kotlin
Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .setRequiresBatteryNotLow(true)
    .build()
```

---

## 7. Rider-Created Pickup Task Flow

Riders can self-create pickup tasks — e.g., scanning a parcel for reverse pickup or self-assigning an unclaimed shipment.

### 7.1 Flow Overview

```
Rider wants to create a pickup task
    │
    ├── Input Method A: Scan barcode (camera)
    │       → Decode shipment ID from QR / barcode
    │
    └── Input Method B: Manual entry
            → Rider types shipment ID
                     │
                     ▼
           Check local DB: already exists?
                     │
           YES → show existing task, do not duplicate
                     │
           NO  → check connectivity
                     │
          ┌──────────┴──────────┐
       Online                Offline
          │                     │
          ▼                     ▼
  POST /v1/shipments/validate   INSERT locally as
  → backend validates + claims  UNVERIFIED_PICKUP
  → INSERT to DB as ASSIGNED    + journal PICKUP_CREATED (PENDING)
  → full flow continues         → sync resolves later
          │                     │
          └──────────┬──────────┘
                     ▼
          action_journal: PICKUP_CREATED
          UI shows task immediately
```

### 7.2 Create Pickup UseCase

```kotlin
class CreatePickupTaskUseCase @Inject constructor(
    private val shipmentDao: ShipmentDao,
    private val journalDao: ActionJournalDao,
    private val apiService: DeliveryApiService,
    private val connectivity: ConnectivityObserver,
    private val ntpTime: NtpTimeProvider
) {
    suspend fun execute(shipmentId: String, source: InputSource): Result<Unit> {
        // Duplicate guard
        val existing = shipmentDao.getById(shipmentId)
        if (existing != null) return Result.failure(ShipmentAlreadyExistsException(existing.currentState))

        return if (connectivity.isOnline.value) createOnline(shipmentId, source)
        else createOffline(shipmentId, source)
    }

    private suspend fun createOnline(shipmentId: String, source: InputSource): Result<Unit> {
        return try {
            val response = apiService.validateAndClaimShipment(
                ValidateShipmentRequest(
                    shipmentId = shipmentId,
                    riderId    = getCurrentRiderId(),
                    actionId   = UUID.randomUUID().toString()   // idempotency
                )
            )
            withTransaction {
                shipmentDao.insert(response.toEntity(state = "ASSIGNED"))
                journalDao.insert(ActionJournalEntity(
                    actionId   = response.claimActionId,
                    shipmentId = shipmentId,
                    actionType = "PICKUP_CREATED",
                    syncStatus = SyncStatus.SYNCED,    // already confirmed by server
                    createdAt  = ntpTime.currentTimeMs()
                ))
            }
            Result.success(Unit)
        } catch (e: ShipmentNotFoundException)    { Result.failure(InvalidShipmentIdException()) }
        catch  (e: ShipmentAlreadyClaimedException) { Result.failure(ShipmentConflictException(e.currentRiderId)) }
    }

    private suspend fun createOffline(shipmentId: String, source: InputSource): Result<Unit> {
        withTransaction {
            shipmentDao.insert(ShipmentEntity(
                shipmentId   = shipmentId,
                riderId      = getCurrentRiderId(),
                currentState = "UNVERIFIED_PICKUP",
                inputSource  = source.name,             // SCAN | MANUAL
                createdAt    = ntpTime.currentTimeMs(),
                updatedAt    = ntpTime.currentTimeMs()
            ))
            journalDao.insert(ActionJournalEntity(
                actionId   = UUID.randomUUID().toString(),
                shipmentId = shipmentId,
                actionType = "PICKUP_CREATED",
                syncStatus = SyncStatus.PENDING,
                createdAt  = ntpTime.currentTimeMs()
            ))
        }
        return Result.success(Unit)
    }
}
```

### 7.3 Sync Resolution for Rider-Created Tasks

SyncManager handles `PICKUP_CREATED` entries specially:

```kotlin
"PICKUP_CREATED" -> {
    val response = apiService.validateAndClaimShipment(action.toRequest())
    when (response.status) {
        "VALID" -> {
            // Backend confirmed → upgrade state and fill in shipment details
            shipmentDao.updateState(action.shipmentId, "ASSIGNED")
            shipmentDao.updateWithServerData(response.shipmentDetail)
            journalDao.markSynced(action.actionId)
        }
        "INVALID_ID" -> {
            shipmentDao.updateState(action.shipmentId, "REJECTED")
            journalDao.markFailed(action.actionId, "Invalid shipment ID")
            // Notify rider via local notification
        }
        "ALREADY_CLAIMED" -> {
            shipmentDao.updateState(action.shipmentId, "CONFLICT")
            journalDao.markFailed(action.actionId, "Claimed by rider ${response.otherRiderId}")
            // Show conflict screen
        }
    }
}
```

### 7.4 UI States for Unverified Tasks

```
Pending verification (UNVERIFIED_PICKUP):
┌────────────────────────────────────────────────┐
│  📦  SHP-UNKNOWN-123                           │
│  ⚠️  Unverified — Pending server validation    │
│  Scanned • 2 mins ago                          │
│  [Actions disabled until verified]             │
└────────────────────────────────────────────────┘

After sync → VALID (ASSIGNED):
┌────────────────────────────────────────────────┐
│  📦  SHP-UNKNOWN-123  →  Recipient: Rahul      │
│  ✅  Verified — Assigned to you                │
│  [Pickup]  [View Details]                      │
└────────────────────────────────────────────────┘

After sync → REJECTED:
┌────────────────────────────────────────────────┐
│  📦  SHP-UNKNOWN-123                           │
│  ❌  Invalid shipment ID                       │
│  [Contact Support]                             │
└────────────────────────────────────────────────┘

After sync → CONFLICT:
┌────────────────────────────────────────────────┐
│  📦  SHP-UNKNOWN-123                           │
│  ⚠️  Already claimed by another rider          │
│  [Contact Support]                             │
└────────────────────────────────────────────────┘
```

---

## 8. List Management & Pagination

### 8.1 Why Not Offset-Based

```sql
SELECT * FROM shipments ORDER BY created_at DESC LIMIT 50 OFFSET 200
```

**The problem:** With a live-syncing DB, new rows arrive constantly in the background.

```
Page 1 loaded at T=0:   [SHP1000 ... SHP951]
                                ↓
SHP_NEW synced at T=1 — inserted at top of list
                                ↓
Page 2 loaded at T=2:   OFFSET 50
DB now has 1001 rows → SHP951 shifts one position
→ SHP951 appears again at top of page 2

Result: DUPLICATE ITEM shown to rider ❌
```

Additionally, `OFFSET 900` on 1000 rows forces DB to scan and discard 900 rows on every page load — extremely slow at scale.

**Verdict: Offset-based is unsuitable for a live-syncing offline-first list.**

### 8.2 Cursor-Based Pagination

```sql
-- Page 1 (no cursor)
SELECT * FROM shipments
ORDER BY created_at DESC, shipment_id DESC
LIMIT 50

-- Page N (cursor = last item from previous page)
SELECT * FROM shipments
WHERE (created_at < :lastCreatedAt)
   OR (created_at = :lastCreatedAt AND shipment_id < :lastShipmentId)
ORDER BY created_at DESC, shipment_id DESC
LIMIT 50
```

**How it handles live inserts:**

```
Page 1 loaded:   [...SHP951]  cursor = (SHP951.created_at, SHP951.shipment_id)
                        ↓
SHP_NEW inserted anywhere in list
                        ↓
Page 2 query:    WHERE created_at < SHP951.created_at
                 SHP_NEW is above the cursor → NOT in page 2
                 No duplicates. No skips. ✅
                 New items appear naturally on next list refresh.
```

**Performance:** Index on `(created_at DESC, shipment_id DESC)` lets the DB jump directly to the cursor position — O(log n) regardless of total shipment count.

**Verdict: ✅ Cursor-based pagination everywhere — on Room and on backend API.**

### 8.3 Paging 3 Implementation

**PagingSource (cursor-based):**

```kotlin
class ShipmentCursorPagingSource(
    private val shipmentDao: ShipmentDao
) : PagingSource<ShipmentCursor, ShipmentEntity>() {

    override fun getRefreshKey(state: PagingState<ShipmentCursor, ShipmentEntity>): ShipmentCursor? = null

    override suspend fun load(params: LoadParams<ShipmentCursor>): LoadResult<ShipmentCursor, ShipmentEntity> {
        return try {
            val cursor = params.key   // null = first page
            val items  = shipmentDao.getPage(
                cursorCreatedAt  = cursor?.createdAt,
                cursorShipmentId = cursor?.shipmentId,
                limit            = params.loadSize
            )
            LoadResult.Page(
                data    = items,
                prevKey = null,
                nextKey = if (items.size < params.loadSize) null
                          else items.last().let { ShipmentCursor(it.createdAt, it.shipmentId) }
            )
        } catch (e: Exception) { LoadResult.Error(e) }
    }
}

data class ShipmentCursor(val createdAt: Long, val shipmentId: String)
```

**Room DAO:**

```kotlin
@Query("""
    SELECT * FROM shipments
    WHERE (:cursorCreatedAt IS NULL
           OR created_at < :cursorCreatedAt
           OR (created_at = :cursorCreatedAt AND shipment_id < :cursorShipmentId))
    ORDER BY created_at DESC, shipment_id DESC
    LIMIT :limit
""")
suspend fun getPage(cursorCreatedAt: Long?, cursorShipmentId: String?, limit: Int): List<ShipmentEntity>
```

**ViewModel:**

```kotlin
val shipments: Flow<PagingData<ShipmentUiModel>> = Pager(
    config = PagingConfig(
        pageSize         = 50,
        prefetchDistance = 10,        // trigger next page when 10 items from end
        enablePlaceholders = false
    )
) { ShipmentCursorPagingSource(dao) }
    .flow
    .map { it.map(ShipmentEntity::toUiModel) }
    .cachedIn(viewModelScope)
```

**Compose UI:**

```kotlin
@Composable
fun TaskListScreen(viewModel: TaskListViewModel) {
    val shipments = viewModel.shipments.collectAsLazyPagingItems()

    LazyColumn {
        items(count = shipments.itemCount, key = { shipments.peek(it)?.shipmentId ?: it }) { index ->
            shipments[index]?.let { ShipmentCard(it) }
        }
        when (val append = shipments.loadState.append) {
            is LoadState.Loading -> item { CircularProgressIndicator() }
            is LoadState.Error   -> item { RetryButton { shipments.retry() } }
            else                 -> Unit
        }
    }
}
```

### 8.4 Three-Section List Structure

With 1000 shipments, a flat list is unusable. The list is split into three sections:

```
┌──────────────────────────────────────────────┐
│  🔴  SYNC PENDING  (3)                       │  ← Non-paginated Flow. Always pinned at top.
│  SHP991 • Delivered • Sync pending           │     Rider must always see unsynced items.
│  SHP990 • Failed • Sync pending              │
│  SHP989 • Pickup • Sync pending              │
├──────────────────────────────────────────────┤
│  📋  ACTIVE TASKS  (47)                      │  ← Paging 3, cursor-based
│  SHP988 • Out for Delivery                   │
│  SHP987 • Assigned                           │
│  SHP986 • Unverified Pickup ⚠️               │
│  ... load more on scroll ...                 │
├──────────────────────────────────────────────┤
│  ✅  COMPLETED  (last 2 days, fully synced)  │  ← Paging 3, cursor-based
│  SHP100 • Delivered • Yesterday              │
│  SHP099 • Failed • 2 days ago                │
│  ... load more on scroll ...                 │
└──────────────────────────────────────────────┘
```

**Sync Pending query (non-paginated, Flow-based):**

```kotlin
@Query("""
    SELECT s.* FROM shipments s
    WHERE EXISTS (
        SELECT 1 FROM action_journal j
        WHERE j.shipment_id = s.shipment_id
          AND j.sync_status IN ('PENDING', 'IN_PROGRESS', 'FAILED')
    )
    ORDER BY s.updated_at DESC
""")
fun observePendingSync(): Flow<List<ShipmentEntity>>
```

---

## 9. Database Cleanup Policy

### 9.1 The Core Rule

```
┌──────────────────────────────────────────────────────────────────┐
│                   DELETION ALLOWED ONLY WHEN:                    │
│                                                                  │
│  ✅  shipment.current_state IN ('DELIVERED','FAILED','CANCELLED') │
│                         AND                                      │
│  ✅  ALL action_journal entries for this shipment = SYNCED        │
│      (zero PENDING, IN_PROGRESS, or FAILED entries remain)       │
│                         AND                                      │
│  ✅  MAX(action_journal.synced_at) < NOW() - 2 days              │
│      (past the UI display window)                                │
│                                                                  │
│  ❌  NEVER delete if any journal entry is PENDING, IN_PROGRESS,  │
│      or FAILED — regardless of how old the shipment is           │
│                                                                  │
│  ❌  NEVER delete ASSIGNED, PICKED_UP, OUT_FOR_DELIVERY          │
└──────────────────────────────────────────────────────────────────┘
```

### 9.2 Cleanup Eligibility Decision Tree

```
For each shipment in Room DB:
    │
    ├── current_state IN (DELIVERED, FAILED, CANCELLED)?
    │       → NO  → KEEP. Active shipment.
    │
    └── YES → Any journal entry NOT SYNCED?
                │
                ├── YES (PENDING / IN_PROGRESS / FAILED) → KEEP. Unsynced data. Never delete.
                │
                └── NO (all SYNCED) → MAX(synced_at) < NOW() - 2 days?
                        │
                        ├── NO  → KEEP. Within 2-day display window.
                        │
                        └── YES → ELIGIBLE FOR DELETION
                                    1. Delete local image file
                                    2. DELETE FROM action_journal WHERE shipment_id = ?
                                    3. DELETE FROM shipments WHERE shipment_id = ?
```

### 9.3 UI vs DB Retention Distinction

```
UI SHOWS:
─────────────────────────────────────────────────────────────────────
  Active section:
    WHERE current_state IN ('ASSIGNED','PICKED_UP','OUT_FOR_DELIVERY')
    + UNVERIFIED_PICKUP (rider-created, awaiting validation)

  Sync Pending section:
    WHERE any journal entry is PENDING / IN_PROGRESS / FAILED

  Completed section:
    WHERE current_state IN ('DELIVERED','FAILED','CANCELLED')
      AND all journal entries SYNCED
      AND MAX(synced_at) > NOW() - 2 days

IMPORTANT:
  A DELIVERED shipment with PENDING journal entries is shown in the
  Sync Pending section — NOT in Completed. It is NOT eligible for deletion.

DB RETAINS:
─────────────────────────────────────────────────────────────────────
  Everything shown on UI
  + Any shipment with ANY unsynced journal entry (INDEFINITELY)
  + Completed+synced records for exactly 2 days after sync
```

### 9.4 Cleanup Worker

```kotlin
class DbCleanupWorker @HiltWorker constructor(
    context: Context,
    params: WorkerParameters,
    private val shipmentDao: ShipmentDao,
    private val journalDao: ActionJournalDao,
    private val imageManager: LocalImageManager,
    private val ntpTime: NtpTimeProvider
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        val twoDaysAgoMs = ntpTime.currentTimeMs() - TimeUnit.DAYS.toMillis(2)

        val eligible = shipmentDao.findEligibleForCleanup(twoDaysAgoMs)
        // SQL:
        // SELECT s.shipment_id FROM shipments s
        // WHERE s.current_state IN ('DELIVERED','FAILED','CANCELLED')
        //   AND NOT EXISTS (
        //       SELECT 1 FROM action_journal j
        //       WHERE j.shipment_id = s.shipment_id
        //         AND j.sync_status != 'SYNCED'
        //   )
        //   AND (SELECT MAX(j.synced_at) FROM action_journal j
        //        WHERE j.shipment_id = s.shipment_id) < :twoDaysAgoMs

        eligible.forEach { imageManager.deleteLocalImage(it) }
        journalDao.deleteForShipments(eligible)
        shipmentDao.deleteByIds(eligible)

        return Result.success()
    }
}

// Schedule: once daily
val cleanupWork = PeriodicWorkRequestBuilder<DbCleanupWorker>(1, TimeUnit.DAYS)
    .setConstraints(Constraints.Builder().setRequiresBatteryNotLow(true).build())
    .build()
workManager.enqueueUniquePeriodicWork("DB_CLEANUP", ExistingPeriodicWorkPolicy.KEEP, cleanupWork)
```

### 9.5 Full Lifecycle Example

```
T= 0h   Backend assigns SHP123 → App fetches → INSERT (state=ASSIGNED)

T= 1h   Rider goes offline
        Taps "Picked Up"     → state=PICKED_UP,       journal A1=PENDING
        Taps "Out Delivery"  → state=OUT_FOR_DELIVERY, journal A2=PENDING
        Taps "Delivered"     → state=DELIVERED,        journal A3=PENDING

        [UI: SHP123 shown in Sync Pending section with badge]
        [DB: 3 PENDING entries → cleanup blocked]

T= 2.5h Rider back online → SyncManager fires
        A1 → SYNCED
        A2 → SYNCED
        A3 → SYNCED

        [UI: SHP123 moves to Completed section ✅]
        [Cleanup clock starts from T=2.5h]

T= 50h  DbCleanupWorker runs
        SHP123: DELIVERED + all SYNCED + synced_at is 47.5h ago (> 2 days)
        → Delete image file
        → Delete journal entries A1, A2, A3
        → Delete shipment record SHP123
        [SHP123 gone from DB and UI]

─── What if rider stays offline for 3+ days? ─────────────────────────
        Journal entries remain PENDING indefinitely
        Shipment stays in DB indefinitely (never deleted)
        After 5 unsynced actions: blocking alert shown to rider
        Ops team alerted (no heartbeat from rider's device)
```

---

## 10. Key Component Implementations

### 10.1 NTP Time Provider

```kotlin
class NtpTimeProvider @Inject constructor(
    private val apiService: DeliveryApiService,
    private val ntpOffsetDao: NtpOffsetDao
) {
    private var cachedOffsetMs: Long = 0L

    suspend fun initialize() {
        val serverTimeMs = apiService.getServerTime().epochMs
        cachedOffsetMs   = serverTimeMs - System.currentTimeMillis()
        ntpOffsetDao.save(NtpOffsetEntity(offsetMs = cachedOffsetMs, fetchedAt = System.currentTimeMillis()))
    }

    // Never call System.currentTimeMillis() directly in business logic — use this
    fun currentTimeMs(): Long = System.currentTimeMillis() + cachedOffsetMs
}
```

Offset is fetched once per app session. Battery-efficient. Prevents timestamp tampering by riders. The offset is stored in DB so it survives process restarts within the same session.

### 10.2 Connectivity Observer

**The problem with `ConnectivityManager` alone:**

`NetworkCallback.onAvailable()` fires when the device joins a network — Wi-Fi,
mobile data, ethernet. It does **not** mean the internet is actually reachable.

```
Cases where isConnected = true but internet = false:
  ─────────────────────────────────────────────────
  Captive portal (hotel/airport Wi-Fi — connected but all traffic blocked
                  until login page is submitted)
  Mobile data with zero balance
  ISP DNS working but routing broken
  Network interface up but carrier signal too weak to pass packets
  Corporate proxy that intercepts all traffic
```

If `ConnectivityObserver` only wraps `NetworkCallback`, the SyncManager will
attempt sync on a dead connection, get an `IOException`, retry, and loop —
while the UI incorrectly shows "online." The rider sees the spinner and nothing
happens.

**The fix — two-layer connectivity check:**

```
Layer 1: ConnectivityManager.NetworkCallback
  → Fast. Event-driven. Zero battery cost.
  → Answers: "is a network interface connected?"
  → Used for: triggering T2 sync, fast UI updates

Layer 2: Active reachability probe (HTTP HEAD)
  → Runs after Layer 1 reports connected.
  → Answers: "can we actually reach our backend?"
  → Used for: gating actual sync attempts
```

**Why our own backend, not google.com:**

```
google.com is blocked in some regions (China, certain enterprise networks).
google.com up ≠ our backend up — would mask real backend outages.
We already have a /v1/ping endpoint — reusing it is free and gives
us two signals in one: internet works AND our server is reachable.

google.com is used as a fallback — if our server fails the probe but
google.com succeeds, the problem is our backend, not the internet.
This distinction matters for ops alerting.
```

```kotlin
enum class ConnectivityState {
    OFFLINE,              // No network interface connected
    CONNECTED_UNVERIFIED, // Network connected, reachability not yet confirmed
    ONLINE,               // Network connected + probe succeeded
    CAPTIVE_PORTAL        // Network connected but probe redirected (HTTP != 200)
}

class ConnectivityObserver @Inject constructor(
    private val context: Context,
    private val httpClient: OkHttpClient
) {
    private val _state = MutableStateFlow(ConnectivityState.OFFLINE)
    val state: StateFlow<ConnectivityState> = _state.asStateFlow()

    // Convenience — existing callers only care about ONLINE vs not
    val isOnline: Boolean get() = _state.value == ConnectivityState.ONLINE

    private val probeScope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    init {
        registerNetworkCallback()
    }

    // ── Layer 1: ConnectivityManager callback ──────────────────────────
    private fun registerNetworkCallback() {
        val mgr = context.getSystemService(ConnectivityManager::class.java)
        val cb  = object : ConnectivityManager.NetworkCallback() {

            override fun onAvailable(network: Network) {
                // Network interface is up — but don't claim ONLINE yet
                _state.value = ConnectivityState.CONNECTED_UNVERIFIED
                // Layer 2: confirm actual reachability
                probeScope.launch { probe() }
            }

            override fun onLost(network: Network) {
                _state.value = ConnectivityState.OFFLINE
            }

            override fun onCapabilitiesChanged(
                network: Network,
                caps: NetworkCapabilities
            ) {
                // Android 10+: VALIDATED means OS already confirmed internet
                // Use it as a fast path — skip our probe if OS validated
                if (caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET) &&
                    caps.hasCapability(NetworkCapabilities.NET_CAPABILITY_VALIDATED)) {
                    _state.value = ConnectivityState.ONLINE
                }
            }
        }

        val request = NetworkRequest.Builder()
            .addCapability(NetworkCapabilities.NET_CAPABILITY_INTERNET)
            .build()
        mgr.registerNetworkCallback(request, cb)
    }

    // ── Layer 2: Active reachability probe ─────────────────────────────
    suspend fun probe(): ConnectivityState {
        val result = probeOurBackend() ?: probeGoogleFallback()

        _state.value = result
        return result
    }

    private suspend fun probeOurBackend(): ConnectivityState? {
        return withContext(Dispatchers.IO) {
            try {
                val response = httpClient.newCall(
                    Request.Builder()
                        .url("https://api.ourdelivery.com/v1/ping")
                        .head()                          // HEAD — no body, minimal bytes
                        .header("Cache-Control", "no-cache")
                        .build()
                ).execute()

                when (response.code) {
                    200  -> ConnectivityState.ONLINE
                    302,
                    301  -> ConnectivityState.CAPTIVE_PORTAL  // redirect = portal
                    else -> null   // unexpected — fall through to Google probe
                }
            } catch (e: IOException) {
                null  // our server unreachable — try Google to separate internet vs backend issue
            }
        }
    }

    private suspend fun probeGoogleFallback(): ConnectivityState {
        return withContext(Dispatchers.IO) {
            try {
                val response = httpClient.newCall(
                    Request.Builder()
                        .url("https://www.google.com")
                        .head()
                        .header("Cache-Control", "no-cache")
                        .build()
                ).execute()

                when (response.code) {
                    200  -> {
                        // Google reachable but our backend wasn't
                        // → internet works, OUR BACKEND IS DOWN
                        // Log this as a backend outage signal
                        eventLogger.log(
                            eventType    = SyncEventType.SYNC_FAILED,
                            severity     = SyncEventSeverity.ERROR,
                            errorCode    = "BACKEND_UNREACHABLE",
                            errorMessage = "Google reachable but /v1/ping failed"
                        )
                        // Still return OFFLINE from sync's perspective —
                        // our backend is down so sync will fail anyway
                        ConnectivityState.OFFLINE
                    }
                    else -> ConnectivityState.CAPTIVE_PORTAL
                }
            } catch (e: IOException) {
                ConnectivityState.OFFLINE   // neither backend nor Google reachable
            }
        }
    }
}
```

**Probe configuration on the OkHttpClient:**

```kotlin
// Injected via Hilt — separate from the main API client
// Short timeouts: probe should fail fast, not block sync for 30 seconds
@Provides @Named("probe")
fun provideProbeHttpClient(): OkHttpClient = OkHttpClient.Builder()
    .connectTimeout(3, TimeUnit.SECONDS)   // fail fast
    .readTimeout(3, TimeUnit.SECONDS)
    .writeTimeout(3, TimeUnit.SECONDS)
    .followRedirects(false)                // detect captive portals (they redirect)
    .build()
```

**`followRedirects(false)` is critical for captive portal detection.** A captive
portal intercepts the request and returns a 302 redirect to its login page.
If we follow redirects, we'd GET the login page, get a 200, and incorrectly
think we're online. With redirects disabled, we see the 302 directly and
correctly classify it as `CAPTIVE_PORTAL`.

**How `ConnectivityState.CAPTIVE_PORTAL` is used:**

```kotlin
// In SyncOrchestrator — gate sync behind ONLINE, not just CONNECTED
fun onNetworkAvailable() {
    if (connectivity.state.value != ConnectivityState.ONLINE) return
    triggerSync()
}

// In UI — show captive portal prompt
when (connectivity.state.value) {
    ConnectivityState.CAPTIVE_PORTAL -> {
        // Show banner: "Connected to Wi-Fi but internet is blocked.
        //               Tap to open login page."
        // Deep link to: Settings → Wi-Fi → captive portal page
    }
    ConnectivityState.OFFLINE -> { /* offline banner */ }
    ConnectivityState.ONLINE  -> { /* normal */ }
    ConnectivityState.CONNECTED_UNVERIFIED -> { /* probing... */ }
}
```

**`NET_CAPABILITY_VALIDATED` fast path (Android 10+):**

Android's OS validates internet connectivity itself on API 29+. When
`onCapabilitiesChanged` fires with both `INTERNET` and `VALIDATED` capabilities,
the OS has already confirmed real internet access — we can skip our probe entirely
and go straight to `ONLINE`. Our probe only runs on older Android versions or
when the OS validation hasn't fired yet.

**Probe frequency — when does it run:**

```
Trigger                          Probe runs?   Reason
─────────────────────────────────────────────────────────────────────
onAvailable() fires              Yes           New network, must verify
onCapabilitiesChanged (VALIDATED)No            OS already confirmed, trust it
T2 sync trigger                  Only if UNVERIFIED  Don't re-probe if already ONLINE
Explicit retry by user           Yes           Manual "Retry" button triggers probe
App foreground (T1)              Only if OFFLINE/UNVERIFIED  Don't probe every foreground
─────────────────────────────────────────────────────────────────────
```

**The two-signal value of Google as fallback:**

```
Our backend probe fails + Google probe fails → OFFLINE (internet is down)
Our backend probe fails + Google probe works → our backend is DOWN, internet fine
Our backend probe works                      → ONLINE

This gives ops two different alerts:
  "Rider offline" → internet problem, ops cannot help
  "Backend unreachable" → our infra problem, ops must act immediately
```
```

### 10.3 MVI Pattern — Task Feature

```kotlin
// Single immutable UI state
data class TaskListUiState(
    val syncPendingTasks : List<ShipmentUiModel> = emptyList(),
    val activeTasks      : PagingData<ShipmentUiModel> = PagingData.empty(),
    val completedTasks   : PagingData<ShipmentUiModel> = PagingData.empty(),
    val isOffline        : Boolean = false,
    val pendingCount     : Int = 0,
    val isRestricted     : Boolean = false,   // true when pendingCount ≥ 5
    val error            : String? = null
)

// Explicit intents — all state changes go through here
sealed class TaskListIntent {
    object LoadTasks : TaskListIntent()
    data class PerformAction(val shipmentId: String, val action: TaskAction) : TaskListIntent()
    data class CreatePickup(val shipmentId: String, val source: InputSource) : TaskListIntent()
    object RetrySync : TaskListIntent()
    object DismissError : TaskListIntent()
}
```

### 10.4 Image Handling

```
Rider captures delivery photo
         │
         ▼
Compress to WebP (quality = 80%)
         │
         ▼
Save to app-internal storage  (path stored in shipments.image_path)
         │
    On Sync
         │
         ▼
Upload image → receive server image_ref
         │
         ▼
UPDATE shipments SET image_uploaded=1, image_path=NULL
         │
         ▼
Delete local file
(local copy also removed by DbCleanupWorker after 2-day retention)
```

Monitor storage availability before capture. Alert rider and block image capture if storage critically low.

---

## 11. API Contracts

### Submit Actions (Idempotent)

```
POST /v1/actions/batch
Content-Type: application/json

{
  "actions": [
    {
      "action_id":   "uuid-v4",        ← idempotency key
      "shipment_id": "SHP123",
      "rider_id":    "R456",
      "action_type": "DELIVERED",
      "timestamp":   1709123456789,    ← NTP-adjusted epoch ms
      "payload": { "otp_verified": true, "image_ref": "img-uuid" }
    }
  ]
}

Response 200:
{
  "results": [
    { "action_id": "uuid1", "status": "ACCEPTED" },
    { "action_id": "uuid2", "status": "ALREADY_PROCESSED" },   ← safe duplicate
    { "action_id": "uuid3", "status": "INVALID_TRANSITION",
      "error": "Shipment is CANCELLED", "current_server_state": "CANCELLED" }
  ]
}
```

### Validate and Claim Shipment (Rider-Created Pickup)

```
POST /v1/shipments/validate
{
  "action_id":  "uuid-v4",
  "shipment_id": "SHP-UNKNOWN-123",
  "rider_id":    "R456"
}

Response 200: { "status": "VALID",           "shipment_detail": { ... }, "claim_action_id": "uuid" }
Response 200: { "status": "INVALID_ID" }
Response 200: { "status": "ALREADY_CLAIMED", "other_rider_id": "R789" }
```

### Get Server Time

```
GET /v1/time
Response: { "epoch_ms": 1709123456789 }
```

### Get Shipments (Incremental Fetch)

```
GET /v1/shipments?rider_id=R1&updated_after=1709000000000&cursor=SHP950&limit=50
Response: {
  "shipments": [ ... ],
  "next_cursor": "SHP900",
  "has_more": true
}
```

---

## 12. Edge Case Handling

All edge cases are documented in full detail in their respective sections. This table is intentionally kept as a quick-reference index.

| Category | See Section |
|---|---|
| Sync conflicts (duplicate, out-of-order, version, clock skew, replay) | §18 |
| Storage full, emergency cleanup | §17 |
| OEM kills, WorkManager, network flapping | §6.6, §18.12, §18.13, §19 |
| QR scan failures, wrong parcel, damaged label | §20 |
| Silent sync failures, crash recovery, DB corruption | §6.5, §18.15, §21 |
| Offline write-block threshold (open) | §16 |

---

## 13. Trade-offs

| Decision | Benefit | Trade-off |
|---|---|---|
| **Offline-first (Room as source of truth)** | Uninterrupted rider workflow | Higher sync complexity; conflict resolution required |
| **MVI architecture** | Predictable, testable, no UI state inconsistency | More boilerplate vs MVVM |
| **Idempotent APIs (action_id)** | Safe retries, zero duplicate records | Backend must maintain idempotency store (Redis/DB lookup overhead) |
| **Cursor-based pagination** | Stable under live DB mutations; O(log n) performance | Cannot jump to arbitrary page; forward-only scrolling |
| **Batch sync (50/batch)** | Fewer API calls; better throughput | Partial failure handling adds complexity |
| **Strict state machine** | Prevents invalid transitions at every layer | More validation logic; stricter UX |
| **WorkManager foreground service** | Survives OEM battery kills | Persistent sync notification (minor UX friction) |
| **NTP offset (fetch once per session)** | Battery-efficient; no repeated network calls | Minor time drift in very long sessions; mitigated by re-fetching on foreground |
| **Offline write restriction threshold** | Limits data loss window | Threshold and policy unresolved — see §16 Open Questions |
| **Action journal as sync driver** | Full audit trail; replay capability; decoupled from UI | Additional table to maintain; cleanup logic required |
| **UNVERIFIED_PICKUP for offline task creation** | Never blocks rider workflow | Phantom tasks appear until sync resolves them |
| **2-day DB retention post-sync** | Rider can review recent history | DB grows until cleanup runs; 1000+ shipments × 2 days can be significant |
| **Domain + feature modularisation (no full multi-module)** | Faster initial development | Refactoring effort when team/codebase scales |

---

## 14. Observability & Debugging

Full observability design — event schema, local buffer, backend ingest endpoint, alerting rules, and dashboard metrics — is in **Section 21**.

Quick reference:
- Local debug screen: long-press app version → shows journal log, sync history, NTP offset
- Crashlytics breadcrumbs on every state transition and sync trigger
- Target metrics: sync success rate > 99.5%, p95 time-to-sync, RETRY_EXHAUSTED count = 0

---

## 16. Open Questions

| Question | Proposed Direction |
|---|---|
| Rider uninstalls before sync? | Backend TTL monitor: no incoming actions from rider while shipments are in-flight for >N hours → alert ops |
| Two devices, same rider ID? | Single active session: backend invalidates old token; app detects 401 → force re-login |
| Room DB corrupted? | Detect `SQLiteException` on open → export pending journal entries via emergency HTTP fallback → prompt reinstall |
| Backend partially commits a batch? | Per-record status in batch response; unacknowledged entries remain PENDING and retry next cycle |
| Rider in offline zone for > 24 hrs? | After 5 unsynced deliveries: restrict. Ops team alerted via backend monitoring. If GPS available, use last known ping |
| UNVERIFIED_PICKUP: what if rider performs 10 offline pickups and 8 are rejected? | Each is independently resolved on sync. Rejected tasks show clearly in list with "Contact Support". No cascading failure |
| NTP offset: what if server time fetch fails on launch? | Fall back to last persisted offset from `ntp_offset` table. If no offset available, use device time with a warning logged |

---

### Open Question: Should write operations be blocked when pending sync count > 5 (post-Pickup state)?

**The question precisely:**
Once a rider has picked up a shipment (state = `PICKED_UP` or beyond), should the
app block further state transitions — OUT_FOR_DELIVERY, DELIVERED, FAILED — if
there are already 5 or more unsynced actions sitting in the journal?

**Why this question is genuinely hard:**
It sits at the intersection of data integrity, rider operations, and user trust.
Neither "always block" nor "never block" is clearly right.

---

**Case FOR blocking (hard restriction):**

```
1. Data loss window is bounded
   Each unsynced action is a delivery event that only exists on one device.
   If the device is lost, stolen, or crashes, those 5+ events are gone.
   Blocking forces the rider to sync before accumulating more risk.

2. Backend state divergence is capped
   With 5+ unsynced transitions, the backend's view of SHP123 is
   potentially 5 states behind. If ops reassigns the shipment, cancels it,
   or another system acts on it, the conflicts compound with each
   additional unsynced action.

3. Compliance and proof-of-delivery
   In COD (Cash on Delivery) shipments, DELIVERED is a financial event.
   If 5 COD deliveries are unsynced, the financial reconciliation is broken.
   A hard block ensures financial events are anchored to a sync point.

4. Rider accountability
   If a rider marks 20 deliveries offline and 10 are disputed by recipients,
   there is no server-side timestamp trail. Blocking at 5 creates regular
   sync checkpoints that provide an audit trail.
```

**Case AGAINST blocking (soft warning only):**

```
1. Riders operate in real dead zones
   Delivery routes through tunnels, basements, rural areas, industrial
   estates can have zero signal for hours. Blocking mid-route means
   the rider physically cannot complete their shift even when doing
   everything correctly. Operations halt for an infrastructure problem.

2. The data is safe on-device
   Room DB with WAL mode is durable. The 5 unsynced actions are not
   "at risk" unless the device itself is lost. Blocking to prevent
   data loss is overly conservative — the data exists, it just hasn't
   been confirmed by the server yet.

3. Blocking creates worse workarounds
   If riders are blocked, they will find ways around it — clearing app
   data, logging out and back in, using paper notes. These workarounds
   create more data integrity problems than the offline actions themselves.

4. The threshold is arbitrary
   Why 5? A rider doing 100 deliveries/day might hit 5 unsynced actions
   in 10 minutes in a weak signal area. The threshold doesn't map to any
   real risk boundary — it just creates friction.
```

**The nuanced middle ground — state-aware selective restriction:**

Rather than a blanket count-based block, restrict based on **which states** are
unsynced and **what the rider is trying to do next:**

```
Unsynced action type         Next action attempted     Allow?
──────────────────────────────────────────────────────────────────────
PICKUP actions (any count)   PICKUP (new shipment)     ✅ Always allow
                             OUT_FOR_DELIVERY           ✅ Allow (same shipment chain)
                             DELIVERED / FAILED         ⚠️ Warn at 5+, don't block

DELIVERED / FAILED (COD)     Any new action             🔴 Block at 3+ unsynced COD
(financial events)                                          events specifically

DELIVERED / FAILED (non-COD) Any new action             ⚠️ Warn at 10+, don't block

Any type, offline > 2 hours  DELIVERED / FAILED         ⚠️ Strong warning + "Sync now"
                                                            but don't hard block
──────────────────────────────────────────────────────────────────────
```

**Proposed direction (to be validated with ops team):**

Apply a **two-tier response** rather than a binary block:

```
Tier 1 — WARNING (5–9 unsynced post-pickup actions):
  Show persistent banner: "5 deliveries waiting to sync. Connect to sync now."
  Allow all actions. No friction on task completion.
  T4 adaptive sync escalates to 5-minute interval (Section 19).

Tier 2 — SOFT BLOCK (10+ unsynced post-pickup actions, OR 3+ unsynced COD):
  Show blocking modal before each new DELIVERED / FAILED action.
  Modal has [Sync Now] and [Continue Anyway] options.
  "Continue Anyway" is available but requires explicit acknowledgement.
  Ops is alerted via OFFLINE_BUILDUP sync event (Section 21).
  Rider is never fully stopped — they can override with awareness.

Hard block:
  Only if device storage is critically low (Section 17 SEVERE level).
  Not based on pending count alone.
```

**Key questions for the ops/product team to resolve:**

```
1. What is the actual data loss incident rate from offline action accumulation?
   If it's < 0.01%, a hard block is worse than the problem it solves.

2. Are COD deliveries tracked separately in the financial system?
   If yes, they warrant stricter treatment than non-COD.

3. What do riders do today when they lose signal mid-route?
   Their workaround behaviour defines the real constraint.

4. Is there a contractual SLA on proof-of-delivery timestamp accuracy?
   If so, that SLA defines the maximum allowable offline window,
   which maps directly to the threshold.
```

**This question should remain open until real field data from rider behaviour
and ops incident reports is available. Do not set the threshold by intuition.**

---

---

## 17. Storage Pressure & Emergency Cleanup

### 17.1 The Problem

The periodic `DbCleanupWorker` (Section 9.4) runs once daily and only deletes records older than 2 days. Under normal conditions this is sufficient. But a high-volume rider — 1000+ shipments per day, with delivery images — can fill device storage before the 2-day window expires.

This is a low-probability but high-impact scenario. When it hits, the app must:

1. Detect the pressure **before** it causes a crash or write failure
2. Clean up what is **safely deletable** — using the exact same state machine rules, never touching unsynced data
3. Communicate clearly to the rider at each pressure level so they are never surprised

The cleanup eligibility rules never change regardless of pressure level:

```
ALWAYS SAFE TO DELETE:
  ✅  current_state IN (DELIVERED, FAILED, CANCELLED)
  ✅  ALL journal entries for shipment = SYNCED
  ✅  (No time constraint during emergency — we drop the 2-day window)

NEVER DELETE:
  ❌  Any journal entry is PENDING, IN_PROGRESS, or FAILED
  ❌  current_state is ASSIGNED, PICKED_UP, OUT_FOR_DELIVERY
  ❌  current_state is UNVERIFIED_PICKUP, REJECTED, CONFLICT
```

The only thing emergency cleanup relaxes is the **2-day age requirement**. Safety rules are absolute.

---

### 17.2 Storage Pressure Levels

```
Device Free Storage          Level          App Behaviour
─────────────────────────────────────────────────────────────────────
> 200 MB                     NORMAL         No action. Periodic cleanup only.

100 MB – 200 MB              WARN           Show dismissible yellow banner.
                                            Suggest rider sync + free space.
                                            Trigger emergency cleanup quietly.

50 MB – 100 MB               CRITICAL       Show persistent orange banner.
                                            Block new image captures.
                                            Trigger aggressive emergency cleanup.
                                            Prompt rider to sync immediately.

< 50 MB                      SEVERE         Show blocking modal (cannot dismiss).
                                            Block pickup task creation.
                                            Block all image capture.
                                            Attempt sync + cleanup in foreground.
                                            If nothing cleanable → show exact guidance.

0 MB (write failure)         FATAL          Catch DB write exception.
                                            Cannot perform any new actions.
                                            Show critical error screen.
                                            All actions blocked until space freed.
```

Thresholds are configurable via remote config so they can be tuned per device class without an app update.

---

### 17.3 StorageMonitor — Continuous Watching

```kotlin
class StorageMonitor @Inject constructor(
    private val context: Context,
    private val emergencyCleanup: EmergencyCleanupManager,
    private val ntpTime: NtpTimeProvider
) {
    // Emits StoragePressureLevel whenever free space crosses a threshold
    val pressureLevel: StateFlow<StoragePressureLevel> = flow {
        while (true) {
            emit(measurePressure())
            delay(60_000)   // re-check every 60 seconds
        }
    }
    .distinctUntilChanged()     // only emit when level actually changes
    .onEach { level ->
        if (level >= StoragePressureLevel.CRITICAL) {
            emergencyCleanup.runEmergencyCleanup(level)
        }
    }
    .stateIn(CoroutineScope(Dispatchers.IO), SharingStarted.Eagerly, StoragePressureLevel.NORMAL)

    private fun measurePressure(): StoragePressureLevel {
        val stat = StatFs(context.filesDir.path)
        val freeMb = stat.availableBlocksLong * stat.blockSizeLong / (1024 * 1024)
        return when {
            freeMb < 50  -> StoragePressureLevel.SEVERE
            freeMb < 100 -> StoragePressureLevel.CRITICAL
            freeMb < 200 -> StoragePressureLevel.WARN
            else         -> StoragePressureLevel.NORMAL
        }
    }
}

enum class StoragePressureLevel { NORMAL, WARN, CRITICAL, SEVERE, FATAL }
```

The monitor uses `StatFs` — Android's API for querying filesystem free space. It re-checks every 60 seconds and only re-emits when the level actually changes (`distinctUntilChanged`), so the UI doesn't flicker and cleanup doesn't trigger on every tick.

---

### 17.4 Emergency Cleanup: Decision Logic

The emergency cleanup applies the same state machine eligibility tree as periodic cleanup, but **removes the 2-day age floor**. It also prioritises what to delete first to maximise freed space with minimum deletions.

```
EmergencyCleanupManager triggered
    │
    ├── PASS 1: Delete local images for SYNCED terminal shipments
    │       These are the biggest space consumers.
    │       DELETE local WebP files WHERE shipment is SYNCED terminal.
    │       Do NOT delete DB records yet — just reclaim image storage.
    │       ─────────────────────────────────────────────────────────
    │       Re-measure free space → pressure resolved? → STOP.
    │
    ├── PASS 2: Delete full records — SYNCED terminal, synced > 6 hours ago
    │       Slightly more conservative than immediate deletion.
    │       DELETE action_journal + shipments for eligible records.
    │       ─────────────────────────────────────────────────────────
    │       Re-measure free space → pressure resolved? → STOP.
    │
    ├── PASS 3: Delete full records — ALL SYNCED terminal, no age floor
    │       Even recently synced records (< 6 hours ago).
    │       Last resort before SEVERE blocking.
    │       ─────────────────────────────────────────────────────────
    │       Re-measure free space → pressure resolved? → STOP.
    │
    └── Nothing left to safely delete?
            │
            └── Remaining space is held by UNSYNCED data.
                → Cannot delete safely.
                → Show "Sync Now" screen with exact space breakdown.
                → Block further operations (level-dependent).
```

**Why three passes?**

Images are ~200–500KB each. With 1000 shipments, images alone can be 200–500MB. Pass 1 often resolves pressure without touching any DB records, which is faster and less risky. Only escalate to record deletion if image deletion isn't enough.

---

### 17.5 EmergencyCleanupManager Implementation

```kotlin
class EmergencyCleanupManager @Inject constructor(
    private val shipmentDao: ShipmentDao,
    private val journalDao: ActionJournalDao,
    private val imageManager: LocalImageManager,
    private val storageMonitor: StorageMonitor,
    private val ntpTime: NtpTimeProvider
) {
    private val mutex = Mutex()   // prevent concurrent emergency cleanups

    suspend fun runEmergencyCleanup(triggerLevel: StoragePressureLevel) = mutex.withLock {

        // ── PASS 1: Images only ──────────────────────────────────────────
        val syncedTerminalIds = shipmentDao.findSyncedTerminalShipments()
        // SQL:
        // SELECT s.shipment_id FROM shipments s
        // WHERE s.current_state IN ('DELIVERED','FAILED','CANCELLED')
        //   AND s.image_path IS NOT NULL
        //   AND NOT EXISTS (
        //       SELECT 1 FROM action_journal j
        //       WHERE j.shipment_id = s.shipment_id
        //         AND j.sync_status != 'SYNCED'
        //   )

        syncedTerminalIds.forEach { id ->
            imageManager.deleteLocalImage(id)
            shipmentDao.clearImagePath(id)      // set image_path = NULL
        }

        if (storageMonitor.measurePressure() < triggerLevel) {
            logCleanup("Pass 1 resolved pressure", syncedTerminalIds.size, "images")
            return@withLock
        }

        // ── PASS 2: Full records, synced > 6 hours ago ──────────────────
        val sixHoursAgo = ntpTime.currentTimeMs() - TimeUnit.HOURS.toMillis(6)
        val pass2Ids = shipmentDao.findEligibleForCleanup(syncedBefore = sixHoursAgo)

        journalDao.deleteForShipments(pass2Ids)
        shipmentDao.deleteByIds(pass2Ids)

        if (storageMonitor.measurePressure() < triggerLevel) {
            logCleanup("Pass 2 resolved pressure", pass2Ids.size, "records (>6h)")
            return@withLock
        }

        // ── PASS 3: All synced terminal — no age floor ───────────────────
        val pass3Ids = shipmentDao.findEligibleForCleanup(syncedBefore = Long.MAX_VALUE)
            .minus(pass2Ids.toSet())   // skip already deleted

        journalDao.deleteForShipments(pass3Ids)
        shipmentDao.deleteByIds(pass3Ids)

        if (storageMonitor.measurePressure() < triggerLevel) {
            logCleanup("Pass 3 resolved pressure", pass3Ids.size, "records (all synced)")
            return@withLock
        }

        // ── Nothing cleanable — remaining space is UNSYNCED data ─────────
        logCleanup("Emergency cleanup exhausted — unsynced data holds space", 0, "")
        // StorageMonitor StateFlow still holds SEVERE → UI reacts
    }
}
```

---

### 17.6 What Happens When Nothing Is Cleanable

If all three passes run and pressure is still SEVERE, it means the remaining storage is held entirely by **unsynced data** — actions that cannot be deleted because the rider is offline and has a large backlog.

This is the most dangerous state. The response is:

```
Remaining data breakdown shown to rider:
─────────────────────────────────────────────────────────────────
  ⚠️  Storage critically low
  We cannot free space automatically because you have unsynced
  deliveries that haven't reached the server yet.

  📊  Space breakdown:
      Unsynced delivery images:   340 MB   (68 images)
      Unsynced action records:      2 MB   (47 actions)
      System / other app data:    150 MB
      ──────────────────────────────────────
      Free space remaining:        38 MB

  What you can do:
  ┌─────────────────────────────────────────────┐
  │  [Connect to Wi-Fi and Sync Now]            │  ← primary CTA
  │  Syncing will free ~342 MB immediately      │
  └─────────────────────────────────────────────┘
  ┌─────────────────────────────────────────────┐
  │  [Free space on my device]                  │  ← opens Android storage settings
  └─────────────────────────────────────────────┘

  Until space is freed:
  ❌  New image captures are blocked
  ❌  New pickup task creation is blocked
  ✅  You can still mark existing tasks as Delivered or Failed
      (without images, if image capture is required, it will be skipped)
```

The UI tells the rider **exactly how much space syncing will free** — this is calculable: sum of unsynced image file sizes + estimated DB record sizes for PENDING journal entries.

---

### 17.7 UI — Progressive Pressure Communication

All banners are driven by `StorageMonitor.pressureLevel` as a `StateFlow` observed in the root ViewModel.

```
WARN level — dismissible yellow banner (persistent for session):
┌──────────────────────────────────────────────────────────────┐
│  ⚠️  Storage getting low. Sync when possible.  [Sync Now] [✕]│
└──────────────────────────────────────────────────────────────┘

CRITICAL level — persistent orange banner (not dismissible):
┌──────────────────────────────────────────────────────────────┐
│  🔶  Storage low. Image capture blocked. [Sync Now to free   │
│      space]                                                  │
└──────────────────────────────────────────────────────────────┘

SEVERE level — blocking modal (blocks task creation, image capture):
┌──────────────────────────────────────────────────────────────┐
│            ⚠️  STORAGE CRITICALLY LOW                        │
│                                                              │
│  Free space: 38 MB                                           │
│  Blocked until resolved: image capture, new pickup creation  │
│                                                              │
│  Syncing will free approximately 342 MB                      │
│                                                              │
│        [ Connect to Wi-Fi & Sync Now ]                       │
│        [ Free device storage ]                               │
│        [ Continue without images ]   ← only if applicable   │
└──────────────────────────────────────────────────────────────┘

FATAL level — full error screen (all actions blocked):
┌──────────────────────────────────────────────────────────────┐
│               ❌  CANNOT SAVE DATA                           │
│                                                              │
│  Your device has no free storage. The app cannot record      │
│  any new actions until space is freed.                       │
│                                                              │
│  Your existing unsynced data is safe and will not be lost.   │
│                                                              │
│        [ Free device storage ]                               │
│        [ Sync Now ]                                          │
└──────────────────────────────────────────────────────────────┘
```

**Compose implementation pattern:**

```kotlin
@Composable
fun RootScreen(viewModel: RootViewModel) {
    val pressureLevel by viewModel.storagePressure.collectAsState()

    Box {
        MainNavHost()   // normal app content

        // Overlays driven entirely by pressure level
        when (pressureLevel) {
            StoragePressureLevel.WARN     -> StorageWarnBanner(onSync = { viewModel.triggerSync() })
            StoragePressureLevel.CRITICAL -> StorageCriticalBanner(onSync = { viewModel.triggerSync() })
            StoragePressureLevel.SEVERE   -> StorageSevereModal(
                freeableMb = viewModel.freeableSpaceMb,
                onSync     = { viewModel.triggerSync() },
                onSettings = { openAndroidStorageSettings() }
            )
            StoragePressureLevel.FATAL    -> StorageFatalScreen()
            else                          -> Unit
        }
    }
}
```

The `freeableSpaceMb` value shown in the modal is computed as:

```kotlin
val freeableSpaceMb: Long
    get() = imageManager.sizeOfUnsyncedImages() / (1024 * 1024)
    // Sum of file sizes for images attached to PENDING/IN_PROGRESS journal entries
```

---

### 17.8 How Emergency Cleanup Differs from Periodic Cleanup

| Aspect | Periodic Cleanup (Section 9) | Emergency Cleanup (Section 17) |
|---|---|---|
| **Trigger** | Once daily via WorkManager | Continuous StorageMonitor crossing threshold |
| **Age requirement** | synced_at > 2 days ago | Pass 1 & 2: > 6 hours ago. Pass 3: no floor |
| **Safety rules** | Identical — never delete unsynced | Identical — never delete unsynced |
| **Scope** | All eligible records | Progressive passes: images first, then records |
| **UI** | Silent background worker | Progressive banners → modal → full error screen |
| **Priority** | Low (battery-not-low constraint) | High (runs immediately in foreground coroutine) |
| **What it relaxes** | Nothing | The 2-day retention window only |
| **What it never relaxes** | Sync safety rules | Sync safety rules — these are absolute |

---

### 17.9 Full Pressure Scenario Walkthrough

```
Rider: 1000 deliveries today, 800 completed and synced, each had a photo (~300 KB)
       Device had 500 MB free at start of day.

T=0h    500 MB free. NORMAL. Periodic cleanup runs tonight.

T=4h    800 deliveries done. Images = 800 × 300KB = 240 MB consumed.
        Free space = 260 MB. Still NORMAL.
        But periodic cleanup hasn't run yet (daily schedule).

T=6h    100 more deliveries. 50 of them have unsynced images (offline zone).
        Free space drops to 190 MB. → WARN level.
        StorageMonitor detects it.
        Emergency cleanup Pass 1 fires:
          → Deletes images for 800 SYNCED terminal shipments
          → Frees 240 MB
          → Free space = 430 MB → back to NORMAL ✅
        Rider sees yellow banner briefly, then it auto-dismisses.

T=8h    Rider enters deep offline zone for 3 hours.
        200 more deliveries, all unsynced, all have images.
        Images = 200 × 300KB = 60 MB consumed.
        Free space = 370 MB → still NORMAL.
        But now 250 unsynced images (50 + 200) held in storage.

T=11h   Rider back online. SyncManager fires.
        250 unsynced actions synced → all SYNCED.
        Emergency cleanup Pass 1 now eligible for these 250 images.
        Fires automatically on next StorageMonitor tick.
        60 MB freed.

── Extreme scenario: rider never syncs for 24 hours ─────────────────

T=24h   1000 deliveries, all unsynced, all have images.
        1000 × 300 KB = 300 MB consumed by images alone.
        Plus DB records ~ 5 MB.
        If device started at 400 MB free:
        Free = 400 - 305 = 95 MB → CRITICAL level.

        Emergency cleanup fires:
          Pass 1: looks for SYNCED terminal images to delete → NONE.
                  (nothing is synced — rider was offline all day)
          Pass 2: looks for SYNCED terminal records → NONE.
          Pass 3: same → NONE.
          Exhausted. All space held by UNSYNCED data.

        UI: persistent orange CRITICAL banner.
        Image capture blocked.
        "Sync will free ~300 MB" shown to rider.
        Rider connects to Wi-Fi → SyncManager fires → 300 MB freed.
        Back to NORMAL.
```


---

## 19. Adaptive Sync Frequency

### 19.1 Why Fixed Periodic Is Wrong at Scale — And Why Count Is Also the Wrong Signal

The original T4 trigger was a `PeriodicWorkRequest` firing every 15 minutes.
Section 19 then improved this to an adaptive interval driven by **pending count**.
Both are better than a fixed periodic, but count is still not the right signal
for T4's actual job.

**The insight from Section 6.1:** T3 and T4 have fundamentally different roles.

```
T3 = sync driver      fires on rider action → "sync what I just did"
T4 = stuck detector   fires on time        → "catch what T3/T2/T1 missed"
```

If T4's job is to detect **stuck** PENDING entries — entries that T3 already
tried to sync but couldn't — then the right question to ask is not
"how many PENDING entries are there?" but
**"how long have PENDING entries been sitting here without being synced?"**

```
Why count is wrong as the sole T4 signal:

  Scenario A: Rider records 20 actions, all synced immediately by T3.
              Count = 0. T4 does nothing. ✅ Correct.

  Scenario B: Rider records 3 actions, sync fails (network drops),
              rider puts phone in pocket for 45 minutes.
              Count = 3. Count-based T4 says: "low urgency, 30-min interval."
              But oldest entry is 45 minutes old — something is stuck.
              Age-based T4 says: "escalate to 5-min interval." ✅ More correct.

  Scenario C: Rider records 50 actions in quick succession (hub sorting scan),
              T3 syncs them all successfully in seconds.
              Count-based T4 sees 50 mid-flight and schedules a 1-min periodic
              that fires to find everything already SYNCED. Wasted wake-up.
              Age-based T4: entries are < 5 min old, T3 just ran, no periodic
              scheduled. ✅ More correct.
```

**The right signal for T4: age of the oldest PENDING entry.**
Count is still useful — but as a severity amplifier within a tier, not the
primary axis.

---

### 19.2 Battery & Data Cost Analysis

Understanding the cost of each sync wake-up:

```
Per sync wake-up cost breakdown:
────────────────────────────────────────────────────────────────
Component              Empty sync       Sync with 10 actions
──────────────────────────────────────────────────────────────
WorkManager wake-up    ~5ms CPU         ~5ms CPU
DB query (PENDING)     ~2ms, ~0KB       ~2ms, ~1KB
API call (if pending)  not made         ~80ms, ~5KB up/down
DB writes (SYNCED)     not made         ~3ms, ~1KB
Total CPU time         ~7ms             ~90ms
Total data             ~0 KB            ~6 KB
Battery (est.)         ~0.002 mAh       ~0.04 mAh
────────────────────────────────────────────────────────────────

Empty syncs are cheap individually but compound badly at fleet scale.
720,000 empty wakes/day × 0.002 mAh = 1,440 mAh wasted fleet-wide per day.
That's ~1.4 full phone batteries drained doing nothing, every day.

Data cost (metered connections matter for riders):
  Fixed 15-min: 96 periodic checks/day × ~0.5KB overhead = ~48 KB/day/rider
  Age-adaptive: ~8 checks/day average  × ~0.5KB           = ~4 KB/day/rider
  Fleet saving: 10,000 × 44 KB/day = ~440 MB/day saved
```

**For riders on metered mobile data in cost-sensitive markets, this matters.**

---

### 19.3 Age-Based Adaptive Frequency Tiers

T4 interval is determined by **how long the oldest PENDING entry has been
waiting**, recalculated after every sync attempt and every rider action.

```
┌────────────────────────────────────────────────────────────────────────┐
│                  T4 STUCK-DETECTOR — AGE-BASED TIERS                  │
│                                                                        │
│  Condition                          Interval  Rationale                │
│  ──────────────────────────────────────────────────────────────────    │
│  No PENDING entries                 NONE      Nothing to detect.       │
│                                               Cancel any scheduled T4. │
│                                                                        │
│  PENDING exist,                     NONE      T3 just ran. Give it     │
│  oldest entry < 5 min old                     a fair chance before     │
│                                               T4 interferes.           │
│                                                                        │
│  PENDING exist,                     15 min    Entries are stuck past   │
│  oldest 5 – 30 min old                        T3's window. T4 activates│
│                                               as a quiet backstop.     │
│                                                                        │
│  PENDING exist,                     5 min     Something is wrong.      │
│  oldest 30 min – 2 hrs old                    T2 may have misfired.    │
│                                               Escalate frequency.      │
│                                                                        │
│  PENDING exist,                     1 min     Urgent. Fire             │
│  oldest > 2 hours                             OFFLINE_BUILDUP event.   │
│                                               Show rider banner.       │
└────────────────────────────────────────────────────────────────────────┘
```

**Count as a severity amplifier (secondary signal):**

Within the `oldest > 30 min` and `oldest > 2 hrs` tiers, count adjusts the
battery constraint — if count is also high (> 20), relax `BATTERY_NOT_LOW`
because data loss risk outweighs battery cost.

```
oldest > 2 hrs AND count > 20  → 1-min interval, no battery constraint
oldest > 2 hrs AND count ≤ 20  → 1-min interval, keep battery constraint
oldest 30min–2hrs AND count > 20 → 5-min interval, relax battery constraint
```

---

### 19.4 AdaptiveSyncScheduler Implementation

```kotlin
class AdaptiveSyncScheduler @Inject constructor(
    private val workManager: WorkManager,
    private val journalDao: ActionJournalDao,
    private val ntpTime: NtpTimeProvider
) {
    companion object {
        private const val WORK_TAG_PERIODIC = "ADAPTIVE_SYNC_PERIODIC"

        // Age thresholds (milliseconds)
        private val AGE_GRACE     = TimeUnit.MINUTES.toMillis(5)   // T3 just ran, wait
        private val AGE_STUCK     = TimeUnit.MINUTES.toMillis(30)  // something is stuck
        private val AGE_URGENT    = TimeUnit.HOURS.toMillis(2)     // serious problem

        // Intervals (minutes)
        private const val INTERVAL_BACKSTOP  = 15L   // PeriodicWorkRequest-safe
        private const val INTERVAL_ESCALATED = 5L    // OneTimeWorkRequest chain
        private const val INTERVAL_URGENT    = 1L    // OneTimeWorkRequest chain

        // Count threshold for battery constraint relaxation
        private const val COUNT_HIGH = 20
    }

    /**
     * Called after every rider action and after every sync completion.
     * Re-evaluates age of oldest PENDING entry and reschedules T4 accordingly.
     */
    suspend fun reschedule() {
        val oldestPendingAge = journalDao.oldestPendingAgeMs(ntpTime.currentTimeMs())
        val pendingCount     = journalDao.countPending()
        val tier             = resolveTier(oldestPendingAge)

        when (tier) {
            SyncTier.NONE -> {
                workManager.cancelAllWorkByTag(WORK_TAG_PERIODIC)
            }
            SyncTier.BACKSTOP -> {
                // 15-min PeriodicWorkRequest — WorkManager native
                schedulePeriodicWork(
                    intervalMinutes  = INTERVAL_BACKSTOP,
                    relaxBattery     = false
                )
            }
            SyncTier.ESCALATED -> {
                scheduleOneTimeWork(
                    delayMinutes = INTERVAL_ESCALATED,
                    relaxBattery = pendingCount > COUNT_HIGH
                )
            }
            SyncTier.URGENT -> {
                scheduleOneTimeWork(
                    delayMinutes = INTERVAL_URGENT,
                    relaxBattery = pendingCount > COUNT_HIGH
                )
                // Also fire OFFLINE_BUILDUP event if not already fired recently
                eventLogger.log(
                    eventType    = SyncEventType.OFFLINE_BUILDUP,
                    severity     = SyncEventSeverity.WARN,
                    pendingCount = pendingCount,
                    errorMessage = "Oldest PENDING entry is ${oldestPendingAge / 60_000} minutes old"
                )
            }
        }
    }

    private fun resolveTier(oldestPendingAgeMs: Long?): SyncTier = when {
        oldestPendingAgeMs == null              -> SyncTier.NONE       // nothing pending
        oldestPendingAgeMs < AGE_GRACE          -> SyncTier.NONE       // T3 just ran, wait
        oldestPendingAgeMs < AGE_STUCK          -> SyncTier.BACKSTOP   // quiet backstop
        oldestPendingAgeMs < AGE_URGENT         -> SyncTier.ESCALATED  // escalate
        else                                    -> SyncTier.URGENT     // urgent
    }

    private fun schedulePeriodicWork(intervalMinutes: Long, relaxBattery: Boolean) {
        val request = PeriodicWorkRequestBuilder<AdaptiveSyncWorker>(intervalMinutes, TimeUnit.MINUTES)
            .addTag(WORK_TAG_PERIODIC)
            .setConstraints(buildConstraints(relaxBattery))
            .build()
        workManager.enqueueUniquePeriodicWork(
            WORK_TAG_PERIODIC,
            ExistingPeriodicWorkPolicy.KEEP,   // don't reset the clock if already ticking
            request
        )
    }

    private fun scheduleOneTimeWork(delayMinutes: Long, relaxBattery: Boolean) {
        val request = OneTimeWorkRequestBuilder<AdaptiveSyncWorker>()
            .addTag(WORK_TAG_PERIODIC)
            .setInitialDelay(delayMinutes, TimeUnit.MINUTES)
            .setConstraints(buildConstraints(relaxBattery))
            .build()
        workManager.enqueueUniqueWork(
            WORK_TAG_PERIODIC,
            ExistingWorkPolicy.REPLACE,        // reset delay on each reschedule
            request
        )
    }

    private fun buildConstraints(relaxBattery: Boolean) = Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .apply { if (!relaxBattery) setRequiresBatteryNotLow(true) }
        .build()
}

enum class SyncTier { NONE, BACKSTOP, ESCALATED, URGENT }
```

**Required DAO addition — `oldestPendingAgeMs`:**

```kotlin
@Query("""
    SELECT :nowMs - MIN(created_at)
    FROM action_journal
    WHERE sync_status = 'PENDING'
""")
suspend fun oldestPendingAgeMs(nowMs: Long): Long?
// Returns null if no PENDING entries exist
```
```

---

### 19.5 WorkManager Rescheduling Strategy

**Why the HIGH/CRITICAL tiers use `OneTimeWorkRequest` instead of `PeriodicWorkRequest`:**

```
WorkManager minimum PeriodicWorkRequest interval = 15 minutes (Android OS limit).
We cannot schedule a periodic every 1 or 3 minutes via PeriodicWorkRequest.

Solution: Self-rescheduling OneTimeWorkRequest chain.

  SyncWorker runs
       │
       ▼
  Sync completes (success or failure)
       │
       ▼
  AdaptiveSyncScheduler.reschedule() called from within doWork()
       │
  Re-evaluates pending count
       │
  ┌────┴─────────────────────────────────────────┐
  │                                              │
  Count dropped to 0?              Count still HIGH/CRITICAL?
  → Cancel periodic                → Schedule next OneTimeWorkRequest
  → Done                             with same delay (1 or 3 min)
```

```kotlin
// AdaptiveSyncWorker — wraps SyncWorker, adds self-reschedule
class AdaptiveSyncWorker @HiltWorker constructor(
    context: Context,
    params: WorkerParameters,
    private val syncManager: SyncManager,
    private val scheduler: AdaptiveSyncScheduler
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            setForeground(getForegroundInfo())
            syncManager.sync()
            // After sync: re-evaluate and reschedule
            scheduler.reschedule()
            Result.success()
        } catch (e: Exception) {
            scheduler.reschedule()   // reschedule even on failure
            if (runAttemptCount < 3) Result.retry()
            else Result.failure()
        }
    }
}
```

**Why `ExistingWorkPolicy.REPLACE` for HIGH/CRITICAL:**

If T1/T2/T3 already triggered a sync while the periodic is pending, the periodic
timer should reset — no point firing it 30 seconds after a successful T2 sync.
`REPLACE` cancels the pending delay and starts a fresh countdown from now.

```
T=0:00  Rider marks DELIVERED (action 22 pending) → HIGH tier
        → OneTimeWorkRequest scheduled, delay = 3 min, fires at T=0:03

T=0:01  Network reconnects → T2 trigger → SyncWorker fires immediately
        → Sync completes → pending count = 0
        → AdaptiveSyncScheduler.reschedule() → NONE tier → cancel periodic ✅

Without REPLACE: the T=0:03 periodic would still fire and do nothing.
With REPLACE:    it gets cancelled before it runs.
```

---

### 19.6 Constraint Layering

**`setRequiresBatteryNotLow(true)` — when to relax it:**

The `BATTERY_NOT_LOW` constraint is correct for LOW and MEDIUM tiers — idle syncing
should yield to the battery. But for HIGH and CRITICAL tiers, the rider has a large
backlog and data loss risk is real. The constraint should be relaxed.

```kotlin
private fun buildConstraints(tier: SyncTier) = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .apply {
        when (tier) {
            SyncTier.LOW,
            SyncTier.MEDIUM   -> setRequiresBatteryNotLow(true)   // yield to battery

            SyncTier.HIGH,
            SyncTier.CRITICAL -> { /* no battery constraint — sync is urgent */ }

            SyncTier.NONE     -> { /* irrelevant — not scheduled */ }
        }
    }
    .build()
```

**Wi-Fi vs mobile data awareness:**

On metered connections (mobile data), HIGH/CRITICAL tiers still sync — data loss
is more costly than a few KB of data usage. But the batch size can be reduced:

```kotlin
val batchSize = if (isOnWifi()) 50 else 20
// Smaller batches on mobile data: more API calls but each is smaller
// Rider on metered data pays per KB, not per call
```

**`isOnWifi()` check:**

```kotlin
fun isOnWifi(): Boolean {
    val mgr = context.getSystemService(ConnectivityManager::class.java)
    val caps = mgr.getNetworkCapabilities(mgr.activeNetwork) ?: return false
    return caps.hasTransport(NetworkCapabilities.TRANSPORT_WIFI)
}
```

---

### 19.7 Interaction with Other Sync Triggers

The adaptive periodic (T4) does not replace T1, T2, T3 — it is the **fallback** for
when those event-based triggers are missed. Here is how all four interact:

```
T1 (app foreground):
  Always fires. Not throttled. No dependency on pending count.
  Resets the T4 periodic timer via REPLACE policy.

T2 (network available):
  Always fires. Throttled by 30s cooldown (Section 18.13).
  Resets the T4 periodic timer via REPLACE policy.

T3 (after rider action):
  Always fires. Triggers AdaptiveSyncScheduler.reschedule() to re-evaluate tier.
  May upgrade or downgrade the T4 interval.
  Example: action #5 is recorded → count crosses LOW→MEDIUM → interval drops 30→10 min.

T4 (adaptive periodic):
  Fires ONLY if T1/T2/T3 didn't already handle the pending actions.
  Self-cancels when pending count reaches 0.
  Self-upgrades/downgrades interval as count changes.

Interaction example:
  T=0:00  Rider records action 5 → T3 fires → sync succeeds → count = 0 → T4 cancelled
  T=0:30  Rider goes offline, records 8 more actions → T3 fires → no network → sync fails
           Count = 8 → MEDIUM tier → T4 scheduled every 10 min
  T=0:40  T4 fires → still offline → reschedule T4 for T=0:50
  T=0:45  Network returns → T2 fires → sync succeeds → count = 0 → T4 cancelled ✅
  T=0:50  T4 would have fired — cancelled, does not run ✅
```

---

### 19.8 Fleet-Level Impact

Comparing fixed 15-min periodic vs adaptive across 10,000 riders over a full day:

```
Metric                    Fixed 15-min      Adaptive          Saving
──────────────────────────────────────────────────────────────────────
WorkManager wake-ups/day  960,000           ~120,000          87% ↓
Useful wake-ups/day       ~240,000          ~118,000          (same useful work)
Empty wake-ups/day        ~720,000          ~2,000            99.7% ↓
Estimated battery wasted  1,440 mAh/day     ~4 mAh/day        ~1,436 mAh ↓
Mobile data overhead      ~480 MB/day       ~60 MB/day        87% ↓
Backend health check load ~720,000 hits/day ~2,000 hits/day   99.7% ↓
──────────────────────────────────────────────────────────────────────
Notes:
  - "Useful" = sync triggered with ≥1 PENDING action
  - Battery estimate: 0.002 mAh per empty wake × count
  - Data: 0.5 KB overhead per wake-up × count
  - Adaptive empty wakes ≈ 2,000 (only T4 NONE→LOW boundary edge cases)
```

**The adaptive approach does the same useful work as the fixed periodic but
eliminates nearly all wasteful work.** T1, T2, and T3 handle the vast majority
of real sync events. T4 adaptive is the narrow safety net it was always meant to be —
now correctly sized to the actual risk level at any given moment.

---

### 19.9 Trade-offs of Adaptive Sync

| Aspect | Fixed 15-min | Adaptive | Notes |
|---|---|---|---|
### 19.9 Trade-offs of Adaptive Sync

| Aspect | Fixed 15-min | Adaptive | Notes |
|---|---|---|---|
| Battery cost (idle rider) | High — wakes every 15 min | None — no periodic when 0 pending | Major win for off-shift riders |
| Battery cost (active rider) | Same as adaptive at HIGH | Same as fixed at MEDIUM | No difference in peak load |
| Data cost (metered) | ~48 KB/day/rider overhead | ~6 KB/day/rider overhead | Important in cost-sensitive markets |
| Backend load | 40,000 wake-ups/hr constant | Proportional to actual work | Smoother server load curve |
| Implementation complexity | Simple — one PeriodicWorkRequest | Moderate — tier logic + self-reschedule | Worth the complexity at 10k riders |
| Safety net coverage | Fires regardless | Only fires when there is work | Event triggers (T1/T2/T3) must be reliable |
| WorkManager OS minimum | Not a concern | 15-min floor for PERIODIC mode | HIGH/CRITICAL use OneTimeWorkRequest chain |
| Debuggability | Simple to reason about | Need to log tier transitions | Add tier to sync breadcrumbs in Crashlytics |

---

## 18. Conflict Resolution — Complete Coverage

Every conflict scenario has been audited against the existing design. The table below
shows what was already covered, what needed extending, and what was missing entirely.

| # | Conflict | Previous Status | Action |
|---|---|---|---|
| 1 | Duplicate Action Submission | ✅ Covered | Consolidated below |
| 2 | Task Already Updated on Server | ⚠️ Partial | Extended with version field |
| 3 | Out-of-Order Actions | ❌ Missing | New — sequence_number added |
| 4 | Multiple Devices Conflict | ⚠️ Partial | Extended with full resolution flow |
| 5 | Task Reassigned While Offline | ❌ Missing | New — full scenario |
| 6 | Task Deleted on Server | ❌ Missing | New — 404 handling |
| 7 | Partial Batch Sync Failure | ✅ Covered | Consolidated below |
| 8 | Clock Skew Conflict | ⚠️ Partial | Extended with server-side rejection window |
| 9 | Optimistic Lock Version Conflict | ❌ Missing | New — version field on shipment |
| 10 | Retry After App Reinstall | ❌ Missing | New — server-side journal recovery |
| 11 | WorkManager Duplicate Execution | ⚠️ Partial | Extended — Mutex as second guard |
| 12 | Network Flapping | ❌ Missing | New — debounce + cooldown |
| 13 | Payload Too Large | ❌ Missing | New — adaptive chunking |
| 14 | Data Corruption in Local DB | ⚠️ Partial | Extended with full recovery strategy |
| 15 | Security Conflict (Replay Attack) | ❌ Missing | New — HMAC + replay window |

**Schema addition required:** `sequence_number` and `local_version` columns added to
`action_journal` and `shipments` respectively. See 18.3 and 18.10.

---

### 18.2 Duplicate Action Submission (Idempotency Conflict)

**Scenario:** SyncManager sends action A1 (DELIVERED). Network times out after server
has already processed it. SyncManager retries — server receives A1 again.

**Resolution — already in place, documented here completely:**

```
Client sends:  { action_id: "uuid-A1", action_type: "DELIVERED", shipment_id: "SHP123" }
                                    │
Server checks: idempotency store (Redis, TTL = 7 days)
                                    │
              ┌─────────────────────┴──────────────────────────┐
              │                                                │
     action_id NOT seen                              action_id ALREADY seen
              │                                                │
     Process normally                       Return { status: "ALREADY_PROCESSED" }
     Store action_id in Redis               WITHOUT re-applying any state change
     Return { status: "ACCEPTED" }                            │
              │                                                │
              └─────────────────────┬──────────────────────────┘
                                    │
              Client treats both "ACCEPTED" and "ALREADY_PROCESSED" as success
              → journalDao.markSynced(action.actionId)
```

**Why UUID v4 as idempotency key?** Generated client-side at action creation time,
persisted in `action_journal.action_id`. Survives app restarts, retries, and network
failures. The same UUID is always sent for the same logical action.

---

### 18.3 Task Already Updated on Server

**Scenario:** Rider picks up SHP123 offline. Meanwhile, backend updates SHP123
(e.g., new delivery address, COD amount change, or admin remark). Rider syncs —
their PICKUP action is valid, but the shipment metadata is stale on their device.

**What was missing:** We had `INVALID_TRANSITION` handling, but no mechanism to
refresh stale shipment metadata after a successful sync.

**Resolution:**

```
SyncManager submits PICKUP action for SHP123
                    │
Backend response:
  { status: "ACCEPTED", updated_shipment: { ...full shipment object... } }
                    │
                    ↓
App ALWAYS overwrites local shipment record with updated_shipment from response
(safe to overwrite — the action was accepted, so there is no local conflict)
                    │
                    ↓
Room Flow emits → UI refreshes with latest metadata (address, COD amount, etc.)
```

**API contract addition:** Every `ACCEPTED` response now includes the server's
current view of the shipment:

```
POST /v1/actions/batch response:
{
  "results": [
    {
      "action_id": "uuid-A1",
      "status": "ACCEPTED",
      "updated_shipment": {          ← NEW: always returned on ACCEPTED
        "shipment_id": "SHP123",
        "current_state": "PICKED_UP",
        "server_version": 5,         ← see 18.10
        "address": "...",
        "cod_amount": 250
      }
    }
  ]
}
```

---

### 18.4 Out-of-Order Actions

**Scenario:** Rider performs PICKUP → OUT_FOR_DELIVERY → DELIVERED offline.
Three journal entries: A1, A2, A3 with `created_at` timestamps. On sync, a race
condition or network retry causes them to arrive at the backend in order A1, A3, A2.
Backend tries to apply DELIVERED before OUT_FOR_DELIVERY → `INVALID_TRANSITION`.

**What was missing:** No explicit ordering guarantee on the journal or API call.

**Resolution — `sequence_number` field added to `action_journal`:**

```sql
-- Schema addition
ALTER TABLE action_journal ADD COLUMN sequence_number INTEGER NOT NULL DEFAULT 0;

-- Sequence is per-shipment, monotonically increasing
-- Set at action creation time:
sequence_number = (SELECT COALESCE(MAX(sequence_number), 0) + 1
                   FROM action_journal
                   WHERE shipment_id = :shipmentId)
```

**SyncManager ordering guarantee:**

```kotlin
// Journal query always orders by (shipment_id, sequence_number ASC)
// This ensures actions for the same shipment are always submitted in order

@Query("""
    SELECT * FROM action_journal
    WHERE sync_status = 'PENDING'
    ORDER BY shipment_id ASC, sequence_number ASC
    LIMIT :limit
""")
suspend fun getPendingOrdered(limit: Int): List<ActionJournalEntity>
```

**Batch construction rule:**

```
When building a batch of 50 to send:
→ For any given shipment_id, ALL its pending actions must appear
  in the batch in sequence_number order.
→ Never split a shipment's actions across two batches unless the
  first batch is fully SYNCED first.
```

```kotlin
private fun buildOrderedBatches(pending: List<ActionJournalEntity>): List<List<ActionJournalEntity>> {
    // Group by shipment, preserve sequence order within each group
    val grouped = pending.groupBy { it.shipmentId }
        .mapValues { (_, actions) -> actions.sortedBy { it.sequenceNumber } }

    // Flatten respecting intra-shipment order, chunk into 50
    return grouped.values.flatten().chunked(50)
}
```

**Backend also validates sequence:**

```
Backend rule: for a given shipment_id, reject any action whose
sequence_number is not exactly (last_applied_sequence + 1).

Response: { status: "OUT_OF_ORDER",
            expected_sequence: 2,
            received_sequence: 3 }

Client reaction:
→ Mark A3 as PENDING (not FAILED — it's not wrong, just early)
→ Re-fetch A2 status from backend
→ If A2 was processed: re-submit A3
→ If A2 was not processed: re-submit A2 first, then A3
```

---

### 18.5 Multiple Devices Conflict

**Scenario:** Rider logs into Device A and Device B with the same credentials.
Both go offline. Device A marks SHP123 as DELIVERED. Device B marks SHP123 as FAILED.
Both sync when online.

**What was missing:** Only "backend invalidates older session token" was documented.
The actual conflict resolution flow was not.

**Resolution — two layers:**

**Layer 1: Session enforcement (prevent this from happening)**

```
On login:
  Backend issues a new session_token and invalidates all previous tokens for this rider.
  Device A receives 401 on next API call → forced re-login.
  Only one device can have a valid session at a time.
```

**Layer 2: If it still happens (token hasn't expired yet when offline)**

```
Device A syncs first: DELIVERED accepted by backend.
Device B syncs:       FAILED rejected → INVALID_TRANSITION
                      (backend state = DELIVERED, cannot go to FAILED)

Device B response:
  { status: "INVALID_TRANSITION",
    current_server_state: "DELIVERED",
    resolved_by: "DEVICE_A",
    conflict_at: 1709123456789 }

Device B app:
  → Update local DB: state = DELIVERED
  → Mark journal entry as FAILED with error = "Resolved by another session"
  → Show conflict banner: "This task was already completed on another device"
  → No further action required
```

**DB addition:** Track which device last wrote each action:

```sql
ALTER TABLE action_journal ADD COLUMN device_id TEXT;
-- Populated with a stable device fingerprint at action creation
-- Helps support team trace which device caused a conflict
```

---

### 18.6 Task Reassigned While Offline

**Scenario:** SHP123 is assigned to Rider R1. R1 goes offline. Backend ops team
reassigns SHP123 to Rider R2 (e.g., R1's shift ended). R1 comes back online and
tries to sync their PICKUP action for SHP123.

**What was missing entirely.**

**Resolution:**

```
R1 syncs PICKUP for SHP123
                │
Backend checks: is SHP123 still assigned to R1?
                │
        NO — reassigned to R2
                │
Response: { status: "REASSIGNED",
            new_rider_id: "R2",
            current_server_state: "ASSIGNED" }
                │
                ↓
App reaction:
  → Update local shipment: current_state = "REASSIGNED_AWAY"
  → Mark journal entry as FAILED with error = "Task reassigned"
  → Show UI:

  ┌────────────────────────────────────────────────────┐
  │  📦  SHP123                                        │
  │  ⚠️  This task has been reassigned to another rider │
  │  Any actions you took offline have been discarded. │
  │  [Contact Support if this is incorrect]            │
  └────────────────────────────────────────────────────┘

  → If rider had taken physical possession of the parcel,
    "Contact Support" is the resolution path.
    Ops team can re-assign back to R1 if needed.
```

**New shipment state: `REASSIGNED_AWAY`**

```kotlin
// Added to ShipmentStateMachine
// Terminal state — no further actions allowed on this device for this shipment
object ReassignedAway : ShipmentState()

// Valid transition: any active state → REASSIGNED_AWAY
// But only settable by sync resolution, never by rider action
```

---

### 18.7 Task Deleted on Server

**Scenario:** SHP123 is in rider's local DB (ASSIGNED). Backend permanently deletes
it — e.g., fraudulent shipment, system error, or order fully cancelled at source.
Rider tries to act on it or sync their actions.

**What was missing entirely.**

**Resolution — two sub-cases:**

**Sub-case A: Rider tries to sync an action for a deleted shipment**

```
SyncManager submits PICKUP for SHP123
                │
Backend: SHP123 not found
                │
Response: { status: "SHIPMENT_NOT_FOUND",
            shipment_id: "SHP123",
            reason: "DELETED" }
                │
App reaction:
  → Mark journal entry as FAILED (permanent, no retry)
  → Update local state to "DELETED_ON_SERVER"
  → Show UI:

  ┌──────────────────────────────────────────────┐
  │  📦  SHP123                                  │
  │  ❌  This task no longer exists on the server │
  │  [Contact Support]                           │
  └──────────────────────────────────────────────┘
```

**Sub-case B: Backend pushes deletion via FCM**

```
FCM message: { type: "SHIPMENT_DELETED", shipment_id: "SHP123" }
                │
        Does local journal have PENDING entries for SHP123?
                │
       ┌────────┴────────┐
      YES               NO
       │                 │
  Mark journal       DELETE shipment
  entries FAILED     from local DB
  Update state       immediately
  to DELETED_ON_SERVER
  Show UI banner
```

**New state: `DELETED_ON_SERVER`** — terminal, rider cannot perform any action.
Eligible for cleanup immediately (no 2-day wait), since data is already lost on server.

---

### 18.8 Partial Batch Sync Failure

**Already fully covered in Section 6.3.** Documented here for completeness.

Each action in a batch receives an independent result status. One failure does not
block or roll back other actions in the same batch. Failed entries are retried
independently in the next sync cycle. The ordering guarantee (18.4) ensures that
if A2 fails, A3 (which depends on A2) also stays PENDING until A2 succeeds.

```
Batch of [A1, A2, A3] submitted:
  A1 → ACCEPTED  → SYNCED ✅
  A2 → 5xx error → PENDING (retry_count++) 🔄
  A3 → stays PENDING (depends on A2 by sequence_number) ⏸
```

---

### 18.9 Clock Skew Conflict

**Scenario:** Rider's device clock is wrong (e.g., no NTP sync ever ran, timezone
misconfiguration, or deliberate manipulation). NTP offset partially addresses this,
but what if the server still receives a timestamp that is wildly outside an
acceptable window?

**What was missing:** Server-side rejection policy for extreme clock skew was not
documented.

**Resolution — two layers:**

**Layer 1 (client): NTP offset — already in place (Section 10.1)**

Every timestamp uses `ntpTime.currentTimeMs()` = device time + server offset.
This corrects for honest drift. Re-fetched every foreground event.

**Layer 2 (server): Reject timestamps outside a ±5 minute window**

```
Backend receives action with timestamp T_action.
Server current time: T_server.

If |T_server - T_action| > 5 minutes (configurable):
  Response: { status: "CLOCK_SKEW_REJECTED",
              server_time_ms: T_server,
              received_time_ms: T_action,
              allowed_skew_ms: 300000 }
```

**Client reaction to CLOCK_SKEW_REJECTED:**

```kotlin
"CLOCK_SKEW_REJECTED" -> {
    // Re-fetch NTP offset immediately — something went very wrong
    ntpTimeProvider.initialize()

    // Rewrite the action's timestamp with the corrected offset
    val correctedTimestamp = ntpTimeProvider.currentTimeMs()
    journalDao.updateTimestamp(result.actionId, correctedTimestamp)

    // Reset to PENDING so it retries with corrected timestamp
    journalDao.resetToPending(listOf(result.actionId))
}
```

**What this prevents:** A rider who manually sets their device clock to an earlier
time to fraudulently timestamp a late delivery gets their action rejected and must
re-sync with corrected server time.

---

### 18.10 Optimistic Lock Version Conflict

**Scenario:** Two concurrent updates to the same shipment — e.g., an admin panel
updates SHP123's delivery window at the same time the rider syncs a state change.
Without versioning, the last writer wins silently and one update is lost.

**What was missing entirely.**

**Schema addition — `local_version` on shipments, `server_version` on API:**

```sql
ALTER TABLE shipments ADD COLUMN local_version INTEGER NOT NULL DEFAULT 0;
-- Incremented by 1 on every local state change
-- Sent to backend with every action
-- Backend compares against its own version
```

**Every action now carries the version the rider saw:**

```json
{
  "action_id":     "uuid-A1",
  "shipment_id":   "SHP123",
  "action_type":   "DELIVERED",
  "local_version": 3,           ← "I am modifying version 3"
  "timestamp":     1709123456789
}
```

**Backend version check:**

```
Backend current version of SHP123 = 3?
  → YES: Apply action. Increment version to 4. Return version: 4.
  → NO (version = 5, someone else updated it):
      Response: { status: "VERSION_CONFLICT",
                  client_version: 3,
                  server_version: 5,
                  current_shipment: { ...full object at version 5... } }
```

**Client reaction to VERSION_CONFLICT:**

```kotlin
"VERSION_CONFLICT" -> {
    val serverShipment = result.currentShipment

    // Check: is the rider's action still valid given the new server state?
    val localAction = action.actionType.toShipmentAction()
    val serverState = serverShipment.currentState.toShipmentState()

    if (stateMachine.canTransition(serverState, localAction.toTargetState())) {
        // Action is still valid — update version and retry
        shipmentDao.updateVersion(action.shipmentId, serverShipment.serverVersion)
        journalDao.updateVersion(action.actionId, serverShipment.serverVersion)
        journalDao.resetToPending(listOf(action.actionId))
    } else {
        // Action is no longer valid — conflict, backend wins
        shipmentDao.update(serverShipment.toEntity())
        journalDao.markFailed(action.actionId, "Version conflict — state changed")
        // Show conflict UI
    }
}
```

---

### 18.11 Retry After App Reinstall

**Scenario:** Rider uninstalls and reinstalls the app before all actions have synced.
Room DB is wiped. All pending journal entries are gone. The rider's actions (DELIVERED,
FAILED etc.) are lost on-device and were never pushed to backend.

**What was missing entirely.**

**This is the hardest conflict to recover from fully.** The local audit trail is gone.
Recovery depends entirely on what the backend knows.

**Resolution — two layers:**

**Layer 1: Backend-side pending action tracking**

```
Every action submitted to backend is stored server-side with:
  - action_id
  - rider_id
  - shipment_id
  - sync_status (RECEIVED | APPLIED)

On fresh app install, after login:
  GET /v1/rider/pending-recovery?rider_id=R1

Backend returns:
  {
    "unsynced_shipments": [
      {
        "shipment_id": "SHP123",
        "last_known_state": "ASSIGNED",     ← what backend knows
        "note": "No sync received from rider since 2024-02-28T10:00Z"
      }
    ]
  }

App re-inserts these shipments into the fresh DB.
Rider must re-perform any actions they took while offline.
```

**Layer 2: Proactive backup of journal before uninstall**

This is not always possible (uninstall can happen instantly), but the app can
minimise risk:

```
On every SYNCED journal entry → safe.
On PENDING entries older than 1 hour and no network → upload a lightweight
"intent snapshot" to backend:

POST /v1/rider/intent-snapshot
{
  "rider_id": "R1",
  "pending_actions": [
    { "shipment_id": "SHP123", "intended_state": "DELIVERED", "captured_at": ... }
  ]
}

This is not a formal action — it is a hint to ops team that the rider
attempted these actions. Ops can manually verify and apply if the rider contacts support.
```

**UI after reinstall:**

```
┌──────────────────────────────────────────────────────────────┐
│  Welcome back. We found shipments that were in-progress      │
│  before you reinstalled.                                     │
│                                                              │
│  SHP123 • Assigned to you (last known state)                 │
│  SHP456 • Assigned to you (last known state)                 │
│                                                              │
│  Please re-confirm the status of these shipments.           │
│  [Review Tasks]                                              │
└──────────────────────────────────────────────────────────────┘
```

---

### 18.12 WorkManager Duplicate Execution

**Scenario:** Multiple sync triggers fire in quick succession (e.g., network reconnect
fires at the same time as app foreground event). Without guards, two SyncWorker
instances could run concurrently and submit the same PENDING actions twice.

**What was partially covered:** `enqueueUniqueWork("SYNC", KEEP)` was documented.
The Mutex guard was in code but not explicitly named as the second safety layer.

**Two independent guards — both required:**

```
Guard 1: WorkManager ExistingWorkPolicy.KEEP
─────────────────────────────────────────────
When a "SYNC" unique work is already ENQUEUED or RUNNING:
  → Any new enqueue with the same name is silently ignored.
  → Only one SyncWorker ever enters the execution queue at a time.

  WorkManager.enqueueUniqueWork(
      "SYNC",
      ExistingWorkPolicy.KEEP,   ← new request dropped if one already exists
      syncWorkRequest
  )


Guard 2: Mutex inside SyncManager.sync()
─────────────────────────────────────────
Even if Guard 1 fails (e.g., two WorkManager instances from different
process restarts both start before the uniqueness check registers):

  private val mutex = Mutex()

  suspend fun sync() = mutex.withLock {
      // Only one coroutine can be here at a time
      // Second caller suspends until first completes
  }

Why both are needed:
  WorkManager KEEP prevents queueing duplicates.
  Mutex prevents execution duplicates in edge cases
  (process restart race, direct SyncManager.sync() calls from tests, etc.)
```

**Result:** `ALREADY_PROCESSED` from backend (18.2) is the final safety net even if
both guards somehow fail — the backend will never double-apply an action.

**Three independent guards total** — WorkManager KEEP → Mutex → Backend idempotency.

---

### 18.13 Network Flapping

**Scenario:** Rider's device rapidly alternates between online and offline (weak
signal area, elevator, moving vehicle). Each `onAvailable()` callback fires a sync
trigger. SyncManager starts, network drops mid-batch, entries reset to PENDING,
network returns, sync fires again — tight loop that drains battery and hammers the backend.

**What was missing entirely.**

**Resolution — sync cooldown + debounce:**

```kotlin
class SyncOrchestrator @Inject constructor(
    private val workManager: WorkManager,
    private val prefs: SyncPreferences
) {
    companion object {
        private const val MIN_SYNC_INTERVAL_MS = 30_000L   // 30 seconds cooldown
    }

    fun onNetworkAvailable() {
        val lastSyncAttempt = prefs.lastSyncAttemptMs
        val now = System.currentTimeMillis()

        if (now - lastSyncAttempt < MIN_SYNC_INTERVAL_MS) {
            // Too soon after last attempt — skip this trigger
            return
        }

        prefs.lastSyncAttemptMs = now
        triggerSync()
    }

    // T3 trigger (action recorded) bypasses cooldown — rider-initiated actions
    // should always attempt sync immediately regardless of cooldown
    fun onActionRecorded() = triggerSync()
}
```

**Network flapping UI:** If connectivity changes more than 5 times in 60 seconds:

```
┌──────────────────────────────────────────────────┐
│  📶  Unstable connection detected.               │
│  Sync will resume when signal stabilises.        │
│  Your actions are saved and will sync shortly.   │
└──────────────────────────────────────────────────┘
```

**Exponential backoff on SyncWorker** (already in Section 6.6) also limits retry
frequency naturally. The cooldown adds a client-side gate before WorkManager even
gets involved.

---

### 18.14 Payload Too Large (Scale Conflict)

**Scenario:** Rider has 500 PENDING actions accumulated (deep offline zone). Each
action has a payload. The batch of 50 actions exceeds the backend's request body
limit (e.g., 1 MB), causing HTTP 413 Payload Too Large.

**What was missing entirely.**

**Resolution — adaptive chunking with binary split:**

```kotlin
private suspend fun syncBatch(batch: List<ActionJournalEntity>) {
    val estimatedSizeBytes = batch.sumOf { it.payload.length * 2 }   // rough UTF-16 estimate
    val maxBatchSizeBytes  = 512 * 1024   // 512 KB safety threshold

    if (estimatedSizeBytes > maxBatchSizeBytes) {
        // Binary split: try half the batch
        val half = batch.size / 2
        syncBatch(batch.take(half))
        syncBatch(batch.drop(half))
        return
    }

    val response = try {
        apiService.submitBatch(batch.map { it.toRequest() })
    } catch (e: HttpException) {
        when (e.code()) {
            413 -> {
                // Server explicitly rejected size — binary split and retry
                val half = batch.size / 2
                if (half == 0) {
                    // Single action is too large — payload itself is corrupt/oversized
                    journalDao.markFailed(batch[0].actionId, "Payload too large — single action rejected")
                    return
                }
                syncBatch(batch.take(half))
                syncBatch(batch.drop(half))
                return
            }
            else -> { journalDao.resetToPending(batch.map { it.actionId }); return }
        }
    }
    // ... normal result handling
}
```

**Image payload specifically:** Delivery images are never included in the action
batch body. They are uploaded separately as multipart before the action is submitted,
and only the server `image_ref` UUID is included in the action payload. This caps
action payload size to < 1 KB per action.

**Backend max batch size** is communicated in API response headers so the client
can adapt without a code change:

```
Response header: X-Max-Batch-Size: 25
Client reads this and reduces default batch size for subsequent calls
```

---

### 18.15 Data Corruption in Local DB

**Scenario:** Room DB file is corrupted — e.g., storage hardware fault, interrupted
write during system shutdown, or filesystem-level corruption.

**What was partially covered:** "Catch SQLiteException" was in the edge case table
with no detail.

**Resolution — three-tier recovery strategy:**

**Tier 1: WAL mode + integrity check on open**

```kotlin
Room.databaseBuilder(context, AppDatabase::class.java, "rider_db")
    .setJournalMode(JournalMode.WRITE_AHEAD_LOGGING)   // reduces corruption risk
    .addCallback(object : RoomDatabase.Callback() {
        override fun onOpen(db: SupportSQLiteDatabase) {
            // Run integrity check on every open
            val cursor = db.query("PRAGMA integrity_check")
            cursor.moveToFirst()
            val result = cursor.getString(0)
            if (result != "ok") {
                // Corruption detected — escalate to Tier 2
                corruptionHandler.onCorruptionDetected()
            }
        }
    })
    .build()
```

**Tier 2: Targeted table recovery**

```kotlin
class DbCorruptionHandler @Inject constructor(
    private val context: Context,
    private val apiService: DeliveryApiService
) {
    suspend fun onCorruptionDetected() {
        try {
            // Attempt to read just the action_journal — it may still be intact
            val journal = db.actionJournalDao().getAllRaw()

            // Upload any readable PENDING entries directly as emergency sync
            val pendingActions = journal.filter { it.syncStatus == "PENDING" }
            if (pendingActions.isNotEmpty()) {
                apiService.submitEmergencyJournal(pendingActions.map { it.toRequest() })
            }
        } catch (e: SQLiteException) {
            // Table itself is corrupt — cannot recover journal
            // Log to Crashlytics with db file metadata
        }

        // Wipe and rebuild DB
        context.deleteDatabase("rider_db")
        // Re-fetch all shipments from backend on next launch
    }
}
```

**Tier 3: Fresh start with server re-fetch**

After wiping the corrupt DB, on next launch:
- Server is fetched for all active shipments for this rider
- Any actions that were PENDING at time of corruption and not recovered via Tier 2
  follow the Retry After Reinstall (18.11) flow — ops team can assist

**What is guaranteed:** SYNCED data is already safe on the server. Only PENDING
actions at the moment of corruption are at risk. The Tier 2 emergency upload
minimises this window.

**Crashlytics alert:** DB corruption fires a non-fatal Crashlytics event with device
model, Android version, and available storage — correlates with specific device
hardware issues in the field.

---

### 18.16 Security Conflict (Replay Attack)

**Scenario:** A malicious actor (or compromised device) captures a valid action
payload and replays it — e.g., replaying a DELIVERED action for a shipment to
fraudulently claim delivery completion, or replaying a PICKUP to create ghost pickups.

**What was missing entirely.**

**Resolution — three layers:**

**Layer 1: `action_id` idempotency (already in place)**

Each UUID is single-use. Backend rejects any action_id it has seen before.
An exact replay of a captured payload is rejected as `ALREADY_PROCESSED`.

This handles naive replay. Does not handle attacks where the adversary strips the
action_id and generates a new one while keeping the rest of the payload.

**Layer 2: Replay window — timestamp validation (18.9 extended)**

Backend rejects any action with `timestamp` older than 5 minutes from server time.
Even if an attacker re-signs a payload with a new action_id, they cannot backdate it
past the 5-minute window. Old captured payloads become useless after 5 minutes.

**Layer 3: HMAC payload signature**

```kotlin
object ActionSigner {
    fun sign(action: ActionRequest, riderSecret: String): String {
        val payload = "${action.actionId}|${action.shipmentId}|${action.actionType}|${action.timestamp}"
        val mac = Mac.getInstance("HmacSHA256")
        mac.init(SecretKeySpec(riderSecret.toByteArray(), "HmacSHA256"))
        return Base64.encodeToString(mac.doFinal(payload.toByteArray()), Base64.NO_WRAP)
    }
}

// Every action request includes:
{
  "action_id":   "uuid-A1",
  "shipment_id": "SHP123",
  "action_type": "DELIVERED",
  "timestamp":   1709123456789,
  "signature":   "HMAC-SHA256(action_id|shipment_id|action_type|timestamp, rider_secret)"
}
```

**`rider_secret`:** A per-rider secret issued at login, stored in Android Keystore
(hardware-backed TEE where available). Never stored in plain SharedPreferences.

```kotlin
// Store in Android Keystore
val keyStore = KeyStore.getInstance("AndroidKeyStore")
// Retrieve for signing — secret never leaves Keystore in plain form
```

**Backend verification:**

```
Backend recomputes HMAC with rider's stored secret.
If signatures don't match → { status: "SIGNATURE_INVALID" } → log security event.
If signatures match but action_id already seen → ALREADY_PROCESSED.
If signatures match, action_id is new, timestamp within window → process normally.
```

**What this prevents:**

| Attack | Defeated by |
|---|---|
| Exact replay of captured packet | Layer 1: action_id idempotency |
| Replay with new action_id but old timestamp | Layer 2: 5-minute timestamp window |
| Replay with new action_id and fresh timestamp | Layer 3: HMAC signature (no rider_secret = invalid signature) |
| Rider manually crafting fake deliveries | Layer 3: HMAC (secret in Keystore) + Layer 2 (server time) |

**Security event monitoring:** All `SIGNATURE_INVALID` responses are logged with
`rider_id`, device fingerprint, and IP. More than 3 in 24 hours → auto-suspend
rider account pending security review.

---

### 18.17 Conflict Resolution Decision Matrix

Complete reference — for any conflict, which layer catches it and what happens:

```
Conflict                    | Layer 1 (Client)           | Layer 2 (Backend)           | UI Shown to Rider
────────────────────────────┼────────────────────────────┼─────────────────────────────┼─────────────────────────────
Duplicate action            | action_id in journal       | ALREADY_PROCESSED           | Nothing (silent success)
Task updated on server      | server merge on pull       | updated_shipment in response| Auto-refresh, no friction
Out-of-order actions        | sequence_number ordering   | Reject wrong sequence       | Nothing (auto-retry)
Multiple devices            | Session invalidation       | INVALID_TRANSITION          | "Completed on another device"
Task reassigned             | server_pending_state guard | REASSIGNED response         | "Task reassigned" banner
Task deleted                | FCM push handler           | SHIPMENT_NOT_FOUND          | "Task no longer exists" banner
Partial batch failure       | Per-record retry           | Per-record result           | Nothing (auto-retry)
Clock skew                  | NTP offset                 | CLOCK_SKEW_REJECTED + time  | Nothing (auto-correct + retry)
Version conflict            | local_version tracking     | VERSION_CONFLICT + shipment | "Conflict" banner if irresolvable
App reinstall               | Intent snapshot upload     | /pending-recovery endpoint  | "Review tasks" screen
WorkManager duplicate       | ExistingWorkPolicy.KEEP    | action_id idempotency       | Nothing
Network flapping            | 30s cooldown + debounce    | Idempotency (safety net)    | "Unstable connection" banner
Payload too large           | Adaptive binary split      | 413 + X-Max-Batch-Size      | Nothing (auto-split)
DB corruption               | WAL + integrity_check      | Emergency journal upload    | "Technical issue" + restart
Replay attack               | Timestamp + HMAC           | Signature verification      | Nothing (blocked silently)
```
---

## 20. QR Scan Validation

### 20.1 All Wrong-QR Scenarios

A rider can scan the wrong QR in many different ways. Each requires a different
response — some are silent rejections, some need clear UI feedback, and some are
security signals worth logging.

```
Wrong QR scenarios:
────────────────────────────────────────────────────────────────────
  S1  Completely foreign QR     (restaurant menu, UPI link, URL, Wi-Fi)
  S2  Competitor logistics QR   (valid shipment format, different company)
  S3  Wrong rider's shipment    (our company, valid ID, assigned to R2 not R1)
  S4  Terminal state shipment   (already DELIVERED / CANCELLED / FAILED)
  S5  Wrong zone / hub          (valid shipment, but outside rider's service area)
  S6  Damaged / partial scan    (decodes to garbled string, checksum fails)
  S7  Wrong action context      (delivery QR scanned at pickup stage, or vice versa)
  S8  Duplicate scan            (same QR scanned twice in the same session)
────────────────────────────────────────────────────────────────────
```

Validation must catch these at **two layers**:

- **Layer 1 — Client-side (instant, no network needed):** Structural checks.
  Catches S1, S2, S6, S8 immediately, before any network call.
- **Layer 2 — Server-side (requires connectivity):** Semantic checks.
  Catches S3, S4, S5, S7 definitively.

---

### 20.2 Validation Architecture — Two Layers

```
Camera decodes raw QR string
            │
            ▼
┌───────────────────────────────────────────────────────────┐
│           LAYER 1: CLIENT-SIDE STRUCTURAL VALIDATION      │
│           (instant, zero network, happens always)         │
│                                                           │
│  1. Is it non-empty and within length bounds?             │
│  2. Does it match our QR format/prefix? (e.g. "SHP-...")  │
│  3. Does checksum digit pass?                             │
│  4. Has this QR already been scanned this session?        │
│                                                           │
│  FAIL → reject immediately with specific error code       │
│  PASS → continue to Layer 2                               │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│           LOCAL DB FAST PATH                              │
│           (before any network call)                       │
│                                                           │
│  Is shipment_id already in local Room DB?                 │
│    → YES, state = active    → proceed directly (no server │
│                               call needed)                │
│    → YES, state = terminal  → S4 rejection (immediate)   │
│    → NO                     → go to Layer 2              │
└──────────────────────────────┬────────────────────────────┘
                               │
                               ▼
┌───────────────────────────────────────────────────────────┐
│           LAYER 2: SERVER-SIDE SEMANTIC VALIDATION        │
│           (requires connectivity; skipped if offline)     │
│                                                           │
│  POST /v1/qr/validate                                     │
│  { shipment_id, rider_id, action_context, scan_token }    │
│                                                           │
│  Server checks:                                           │
│  - Does shipment exist?          → S1/S2 if not           │
│  - Is it assigned to this rider? → S3 if not              │
│  - Is it in a terminal state?    → S4 if yes              │
│  - Is it in rider's zone?        → S5 if not              │
│  - Is action_context valid?      → S7 if wrong            │
│                                                           │
│  PASS → proceed to CreatePickupTaskUseCase / action flow  │
└───────────────────────────────────────────────────────────┘
```

---

### 20.3 Layer 1: Client-Side Structural Validation

```kotlin
object QrStructuralValidator {

    // Our shipment QR format: "SHP-{10 alphanumeric}-{1 checksum digit}"
    // Example: "SHP-A1B2C3D4E5-7"
    private val QR_REGEX = Regex("^SHP-[A-Z0-9]{10}-[0-9]\$")

    fun validate(rawQr: String, sessionScannedIds: Set<String>): QrStructuralResult {

        // Check 1: Empty or oversized
        if (rawQr.isBlank() || rawQr.length > 64) {
            return QrStructuralResult.Reject(QrRejectionReason.FOREIGN_QR)
        }

        // Check 2: Format / prefix match
        if (!QR_REGEX.matches(rawQr)) {
            // Does it look like a URL, UPI, Wi-Fi etc.?
            val reason = when {
                rawQr.startsWith("http")  -> QrRejectionReason.FOREIGN_QR_URL
                rawQr.startsWith("upi://")-> QrRejectionReason.FOREIGN_QR_PAYMENT
                rawQr.startsWith("WIFI:") -> QrRejectionReason.FOREIGN_QR_WIFI
                else                      -> QrRejectionReason.FOREIGN_QR
            }
            return QrStructuralResult.Reject(reason)
        }

        // Check 3: Luhn / checksum digit validation
        val shipmentCore = rawQr.substringAfter("SHP-").substringBeforeLast("-")
        val checkDigit   = rawQr.last().digitToInt()
        if (!validateChecksum(shipmentCore, checkDigit)) {
            return QrStructuralResult.Reject(QrRejectionReason.DAMAGED_QR)
        }

        // Check 4: Duplicate scan in current session
        val shipmentId = rawQr  // or extract the canonical ID from the QR
        if (shipmentId in sessionScannedIds) {
            return QrStructuralResult.Reject(QrRejectionReason.DUPLICATE_SCAN)
        }

        return QrStructuralResult.Pass(shipmentId = shipmentId)
    }

    // Luhn-style checksum: sum of digit values mod 10 == checkDigit
    private fun validateChecksum(core: String, expected: Int): Boolean {
        val sum = core.sumOf {
            if (it.isDigit()) it.digitToInt()
            else (it.code - 'A'.code + 1) % 10   // map letters to 1–9
        }
        return (sum % 10) == expected
    }
}

sealed class QrStructuralResult {
    data class Pass(val shipmentId: String) : QrStructuralResult()
    data class Reject(val reason: QrRejectionReason) : QrStructuralResult()
}
```

---

### 20.4 Layer 2: Server-Side Semantic Validation

New dedicated endpoint — lighter than the full `validateAndClaimShipment` used in
pickup creation. This is a **read-only validation** that does not claim or mutate anything.

```
POST /v1/qr/validate
{
  "shipment_id":    "SHP-A1B2C3D4E5-7",
  "rider_id":       "R456",
  "action_context": "PICKUP",           ← what the rider is trying to do
  "scan_token":     "uuid-v4"           ← idempotency: prevents log flooding on retry
}

Response:
{
  "result": "VALID" | "INVALID",
  "rejection_reason": null | "WRONG_RIDER" | "TERMINAL_STATE" |
                               "WRONG_ZONE"  | "WRONG_CONTEXT"  |
                               "NOT_FOUND",
  "detail": {
    "current_state":      "DELIVERED",        ← for TERMINAL_STATE
    "assigned_to_zone":   "North Mumbai",     ← for WRONG_ZONE
    "rider_current_zone": "South Mumbai",
    "expected_context":   "DELIVERY"          ← for WRONG_CONTEXT
  }
}
```

**`action_context` values:**

```
PICKUP          → rider is scanning to self-create a pickup task
OUT_FOR_DELIVERY → rider is scanning to confirm they have the parcel before leaving hub
DELIVERY        → rider is scanning at customer doorstep to mark delivery
```

---

### 20.5 QrValidationUseCase — Full Decision Tree

```kotlin
class QrValidationUseCase @Inject constructor(
    private val structuralValidator: QrStructuralValidator,
    private val shipmentDao: ShipmentDao,
    private val apiService: DeliveryApiService,
    private val connectivity: ConnectivityObserver,
    private val scanAuditLog: ScanAuditLog,
    private val sessionScans: ScanSessionTracker   // in-memory, cleared on app restart
) {
    suspend fun validate(
        rawQr: String,
        actionContext: ActionContext
    ): QrValidationResult {

        // ── LAYER 1: Structural ────────────────────────────────────────
        val structural = structuralValidator.validate(rawQr, sessionScans.scannedIds)
        if (structural is QrStructuralResult.Reject) {
            scanAuditLog.record(rawQr, structural.reason, actionContext)
            return QrValidationResult.Rejected(structural.reason)
        }
        val shipmentId = (structural as QrStructuralResult.Pass).shipmentId

        // ── LOCAL DB FAST PATH ─────────────────────────────────────────
        val localShipment = shipmentDao.getById(shipmentId)
        if (localShipment != null) {
            when {
                localShipment.isTerminal() -> {
                    scanAuditLog.record(shipmentId, QrRejectionReason.TERMINAL_STATE, actionContext)
                    return QrValidationResult.Rejected(
                        reason  = QrRejectionReason.TERMINAL_STATE,
                        detail  = "Shipment is already ${localShipment.currentState}"
                    )
                }
                localShipment.isActive() -> {
                    // Already in our DB and active — valid, skip server call
                    sessionScans.record(shipmentId)
                    return QrValidationResult.Valid(shipmentId, source = ValidationSource.LOCAL_DB)
                }
            }
        }

        // ── LAYER 2: Server-side (if online) ──────────────────────────
        if (!connectivity.isOnline.value) {
            // Offline: cannot do semantic validation
            // Return UNVERIFIED — caller decides whether to accept or defer
            return QrValidationResult.UnverifiedOffline(shipmentId)
        }

        val serverResult = try {
            apiService.validateQr(
                QrValidateRequest(
                    shipmentId    = shipmentId,
                    riderId       = getCurrentRiderId(),
                    actionContext = actionContext.name,
                    scanToken     = UUID.randomUUID().toString()
                )
            )
        } catch (e: IOException) {
            return QrValidationResult.UnverifiedOffline(shipmentId)
        }

        return when (serverResult.result) {
            "VALID" -> {
                sessionScans.record(shipmentId)
                scanAuditLog.record(shipmentId, null, actionContext)
                QrValidationResult.Valid(shipmentId, source = ValidationSource.SERVER)
            }
            "INVALID" -> {
                val reason = QrRejectionReason.fromServerCode(serverResult.rejectionReason)
                scanAuditLog.record(shipmentId, reason, actionContext)
                QrValidationResult.Rejected(reason, detail = serverResult.detail)
            }
            else -> QrValidationResult.UnverifiedOffline(shipmentId)
        }
    }
}

sealed class QrValidationResult {
    data class Valid(
        val shipmentId: String,
        val source: ValidationSource           // LOCAL_DB | SERVER
    ) : QrValidationResult()

    data class Rejected(
        val reason: QrRejectionReason,
        val detail: Any? = null
    ) : QrValidationResult()

    data class UnverifiedOffline(
        val shipmentId: String                 // pass to CreatePickupTaskUseCase as UNVERIFIED_PICKUP
    ) : QrValidationResult()
}

enum class ValidationSource { LOCAL_DB, SERVER }
```

---

### 20.6 Wrong QR Scenario Handling (All 8 Cases)

#### S1 — Completely Foreign QR (restaurant menu, URL, UPI, Wi-Fi)

```
Caught by: Layer 1 — format regex fails immediately
Response:  QrRejectionReason.FOREIGN_QR (sub-typed: FOREIGN_QR_URL, FOREIGN_QR_PAYMENT etc.)

UI:
┌────────────────────────────────────────────────────────┐
│  ❌  Wrong QR code scanned                             │
│  This doesn't look like a shipment label.              │
│  Please scan the barcode on the parcel.                │
│                          [Try Again]  [Enter Manually] │
└────────────────────────────────────────────────────────┘

Camera auto-resumes after 1.5 seconds — no manual dismiss needed.
No network call. No audit log entry (too noisy — these are accidental).
```

#### S2 — Competitor Logistics QR (valid barcode format, different prefix)

```
Caught by: Layer 1 — format prefix check ("SHP-" not matched)
Response:  QrRejectionReason.FOREIGN_QR

Example: Rider scans a DTDC/BlueDart label which has its own barcode format.

UI:
┌────────────────────────────────────────────────────────┐
│  ❌  This parcel belongs to a different courier         │
│  Please scan your company's shipment label only.       │
│                          [Try Again]  [Enter Manually] │
└────────────────────────────────────────────────────────┘

Note: Competitor prefix detection requires a known-prefix list maintained
server-side and synced to the app (via remote config, refreshed daily).
Unrecognised formats fall back to FOREIGN_QR generic message.
```

#### S3 — Valid Shipment, Assigned to Another Rider

```
Caught by: Layer 2 — server checks rider assignment
Response:  QrRejectionReason.WRONG_RIDER

UI:
┌────────────────────────────────────────────────────────┐
│  ❌  This parcel is not assigned to you                 │
│  Shipment SHP-A1B2C3D4E5 belongs to a different rider. │
│  If you believe this is an error:                      │
│                   [Contact Support]   [Scan Another]   │
└────────────────────────────────────────────────────────┘

Security: WRONG_RIDER is logged in scan audit log with rider_id, shipment_id,
and timestamp. Multiple WRONG_RIDER scans from one rider → alert ops.
(Could indicate deliberate attempt to intercept another rider's parcel.)
```

#### S4 — Terminal State Shipment (Already Delivered / Cancelled / Failed)

```
Caught by: Local DB fast path (instant, no network) OR Layer 2
Response:  QrRejectionReason.TERMINAL_STATE

UI:
┌────────────────────────────────────────────────────────┐
│  ⚠️  This shipment is already closed                   │
│  SHP-A1B2C3D4E5 was marked DELIVERED on 28 Feb, 2:34PM │
│  No further actions are allowed.                       │
│                   [View History]   [Scan Another]      │
└────────────────────────────────────────────────────────┘

Shows the terminal state and timestamp so the rider understands what happened.
If the rider believes the closure is wrong → [View History] shows full audit trail.
```

#### S5 — Wrong Zone / Hub

```
Caught by: Layer 2 — server checks geographic assignment
Response:  QrRejectionReason.WRONG_ZONE

UI:
┌────────────────────────────────────────────────────────┐
│  ❌  This parcel is for a different delivery zone       │
│  SHP-A1B2C3D4E5 is assigned to: North Mumbai           │
│  Your current zone: South Mumbai                       │
│                   [Contact Support]   [Scan Another]   │
└────────────────────────────────────────────────────────┘

Common cause: hub sorting error — parcel ended up in wrong batch.
Rider should hand it back to hub supervisor, not attempt delivery.
```

#### S6 — Damaged / Partial QR (Checksum Fails)

```
Caught by: Layer 1 — checksum validation fails
Response:  QrRejectionReason.DAMAGED_QR

UI:
┌────────────────────────────────────────────────────────┐
│  ⚠️  Could not read this barcode clearly               │
│  The label may be damaged or obscured.                 │
│  Try:                                                  │
│    • Better lighting                                   │
│    • Clean the camera lens                             │
│    • Hold steady, 15-20cm from label                  │
│                   [Try Again]  [Enter ID Manually]     │
└────────────────────────────────────────────────────────┘

Actionable tips shown. Manual entry is the fallback.
Partial scan attempts (where the decoded string partially matches our format
but checksum fails) are logged — repeated checksum failures on same barcode
prefix signal a physically damaged label → ops team can re-label.
```

#### S7 — Wrong Action Context

```
Caught by: Layer 2 — action_context mismatch
Response:  QrRejectionReason.WRONG_CONTEXT

Example A: Rider is at hub scanning for PICKUP but scans a
           DELIVERY-only shipment (e.g., a C2C return already processed).

Example B: Rider is doing DELIVERY scan at doorstep but the shipment
           is still in ASSIGNED state (never picked up).

UI for Example B:
┌────────────────────────────────────────────────────────┐
│  ⚠️  This parcel hasn't been picked up yet             │
│  SHP-A1B2C3D4E5 must be picked up before delivery.    │
│  Expected action: PICKUP                               │
│                             [Go to Pickup]   [Cancel]  │
└────────────────────────────────────────────────────────┘

Deep link: [Go to Pickup] opens the task detail screen for this shipment
pre-loaded with the correct action. Rider doesn't re-scan — they just confirm.
```

#### S8 — Duplicate Scan (Same QR in Same Session)

```
Caught by: Layer 1 — sessionScannedIds check
Response:  QrRejectionReason.DUPLICATE_SCAN

This is purely client-side. If the rider already scanned SHP-A1B2C3D4E5
earlier in the same app session, we don't re-validate or re-show the task.

UI:
┌────────────────────────────────────────────────────────┐
│  ℹ️  You already scanned this parcel                   │
│  SHP-A1B2C3D4E5 is in your active task list.           │
│                        [View Task]      [Scan Another] │
└────────────────────────────────────────────────────────┘

[View Task] deep links directly to the task detail screen.
sessionScannedIds is in-memory and clears on app restart —
intentional, because a new session means a legitimate re-scan is possible.
```

---

### 20.7 Offline Validation Behaviour

When the rider is offline, Layer 2 cannot run. The decision per scenario:

```
Scenario          Offline handling
──────────────────────────────────────────────────────────────────────
S1 Foreign QR     Caught by Layer 1 — works offline ✅
S2 Competitor QR  Caught by Layer 1 (prefix list from last remote config sync)
                  If prefix list stale → falls through as UNVERIFIED_PICKUP ⚠️
S3 Wrong rider    Cannot verify offline → UNVERIFIED_PICKUP
                  Server resolves on sync → CONFLICT
S4 Terminal state Caught by local DB fast path if shipment is in DB ✅
                  If not in DB → UNVERIFIED_PICKUP (low risk — terminal
                  shipments are rarely not in local DB)
S5 Wrong zone     Cannot verify offline → UNVERIFIED_PICKUP
                  Likely resolved by hub ops anyway
S6 Damaged QR     Caught by Layer 1 checksum — works offline ✅
S7 Wrong context  Cannot verify offline → UNVERIFIED_PICKUP
                  Sync resolves if context is wrong
S8 Duplicate      Caught by Layer 1 in-memory set — works offline ✅
```

**The offline rule:** If Layer 1 passes and local DB has no answer, always create
as `UNVERIFIED_PICKUP` and let sync resolve it. Never block the rider completely
offline — a false rejection hurts operations more than a false acceptance that
gets caught on sync.

---

### 20.8 Scan Audit Log

Every scan attempt — success or failure — is written to a local `scan_audit_log`
table and synced to the backend. This is separate from `action_journal`.

```sql
CREATE TABLE scan_audit_log (
    scan_id        TEXT PRIMARY KEY,
    shipment_id    TEXT,              -- may be null for fully foreign QR
    raw_qr_hash    TEXT NOT NULL,     -- SHA-256 of raw QR string (never store raw)
    rider_id       TEXT NOT NULL,
    action_context TEXT NOT NULL,
    result         TEXT NOT NULL,     -- VALID | REJECTED | UNVERIFIED_OFFLINE
    rejection_reason TEXT,
    scanned_at     INTEGER NOT NULL,  -- NTP time
    synced         INTEGER DEFAULT 0
);
```

**Why hash the raw QR and not store it?** The raw QR may contain PII (recipient
name embedded in some formats). Storing SHA-256 gives us a unique fingerprint for
deduplication and fraud detection without storing PII locally.

**What the audit log enables:**

```
Ops / Security use cases:
  → Rider scanning many WRONG_RIDER codes → possible interception attempt
  → Rider scanning many FOREIGN_QR codes  → camera or environment issue
  → Same QR scanned by 3 different riders → hub sorting error pattern
  → WRONG_ZONE scans clustered at specific hub → systematic mis-sort
```

---

### 20.9 UI Feedback per Rejection Reason

The camera overlay should provide feedback **without dismissing the scanner** for
recoverable errors. Only unrecoverable errors (WRONG_RIDER, WRONG_ZONE, TERMINAL)
require explicit acknowledgement.

```
Rejection Type     Camera behaviour    Feedback style         Auto-resume?
────────────────────────────────────────────────────────────────────────────
FOREIGN_QR         Stay open           Red flash + toast       Yes, 1.5s
COMPETITOR_QR      Stay open           Red flash + toast       Yes, 1.5s
DAMAGED_QR         Stay open           Yellow flash + tips     Yes, 3s
DUPLICATE_SCAN     Stay open           Blue flash + "View"     Yes, 2s
WRONG_RIDER        Pause               Modal with support CTA  No (explicit)
TERMINAL_STATE     Pause               Modal with history CTA  No (explicit)
WRONG_ZONE         Pause               Modal with zone detail  No (explicit)
WRONG_CONTEXT      Pause               Modal with deep link    No (explicit)
UNVERIFIED_OFFLINE Stay open           Yellow banner           Yes, 2s
VALID              Close scanner       Success animation       N/A
────────────────────────────────────────────────────────────────────────────
```

**Haptic feedback:**
- Valid scan → single strong pulse (success)
- Soft error (FOREIGN_QR, DAMAGED) → double short pulse
- Hard error (WRONG_RIDER, TERMINAL) → long buzz

**Sound feedback (respects silent mode):**
- Valid → success chime
- Error → low error tone

---

### 20.10 Wrong QR Validation Decision Matrix

```
Scenario            Layer Catches  Online Needed  Creates UNVERIFIED?  UI Type
──────────────────────────────────────────────────────────────────────────────
S1 Foreign QR       Layer 1        No             No                   Toast
S2 Competitor QR    Layer 1        No (cached)    No (usually)         Toast
S3 Wrong rider      Layer 2        Yes            Yes (if offline)     Modal
S4 Terminal state   Local DB / L2  No             No                   Modal
S5 Wrong zone       Layer 2        Yes            Yes (if offline)     Modal
S6 Damaged QR       Layer 1        No             No                   Toast+tips
S7 Wrong context    Layer 2        Yes            Yes (if offline)     Modal+deeplink
S8 Duplicate scan   Layer 1        No             No                   Toast+deeplink
──────────────────────────────────────────────────────────────────────────────
```


---

## 21. Sync Event Logging & Observability

### 21.1 The Problem — Silent Failures

The action journal tells us *what happened on the device*. But when sync breaks,
that information is trapped on the device. The backend sees silence — no incoming
actions from a rider — and cannot distinguish between:

```
A. Rider is offline in a dead zone (expected, fine)
B. Sync is broken silently — network present but sync failing (critical)
C. WorkManager killed by OEM — app alive but background work stopped (critical)
D. Rider hasn't opened the app (operational gap)
E. Action journal permanently FAILED after retries (data loss risk)
```

Without telemetry, all five look identical from the backend. The fix is a
dedicated **sync event log** — a lightweight stream of structured events from
the device that tells the backend exactly what the sync engine is doing and
where it is failing, independently of whether sync itself succeeds.

**Critical design constraint:** The event log must work even when sync is broken.
It uses a separate lightweight endpoint and a separate local buffer — if the
main sync is failing for reason X, the event log still gets through to tell
the backend about reason X.

---

### 21.2 What Constitutes a Sync Event

Not every sync cycle is an event. Only meaningful state changes and failures:

```
Category         Event                              Severity
────────────────────────────────────────────────────────────────────────
SYNC_SUCCESS     Batch fully synced                 INFO
SYNC_PARTIAL     Some records in batch failed       WARN
SYNC_FAILED      Entire batch failed (network/5xx)  ERROR
RETRY_EXHAUSTED  Action hit max retries → FAILED    CRITICAL
CONFLICT         INVALID_TRANSITION from server     WARN
CLOCK_SKEW       Server rejected timestamp          WARN
VERSION_CONFLICT Optimistic lock mismatch           WARN
STUCK_INPROGRESS Startup found IN_PROGRESS entries  WARN
OFFLINE_BUILDUP  PENDING count > threshold offline  WARN
DB_WRITE_FAILURE Room write threw exception         CRITICAL
WORKER_EXHAUSTED WorkManager retries exhausted      ERROR
PAYLOAD_TOO_LARGE 413 received from server          WARN
SIGNATURE_INVALID HMAC rejected by server           CRITICAL
REASSIGNED       Shipment reassigned mid-sync       WARN
DELETED          Shipment deleted on server         INFO
────────────────────────────────────────────────────────────────────────
```

---

### 21.3 SyncEvent Schema (Local + Remote)

**Local Room table** — buffer before upload:

```sql
CREATE TABLE sync_event_log (
    event_id          TEXT PRIMARY KEY,   -- UUID v4
    rider_id          TEXT NOT NULL,
    event_type        TEXT NOT NULL,      -- enum values from 21.2
    severity          TEXT NOT NULL,      -- INFO | WARN | ERROR | CRITICAL
    shipment_id       TEXT,               -- nullable: not all events are shipment-specific
    action_id         TEXT,               -- nullable: journal entry that failed
    pending_count     INTEGER,            -- snapshot of PENDING count at event time
    retry_count       INTEGER,            -- for RETRY_EXHAUSTED events
    error_code        TEXT,               -- HTTP status, exception class, server code
    error_message     TEXT,               -- truncated to 512 chars
    device_info       TEXT NOT NULL,      -- JSON: {os_version, app_version, manufacturer}
    network_type      TEXT,               -- WIFI | MOBILE | NONE
    battery_level     INTEGER,            -- 0–100
    free_storage_mb   INTEGER,
    occurred_at       INTEGER NOT NULL,   -- NTP epoch ms
    uploaded          INTEGER DEFAULT 0   -- 0 = pending upload, 1 = uploaded
);

CREATE INDEX idx_sync_event_uploaded ON sync_event_log(uploaded, occurred_at);
CREATE INDEX idx_sync_event_severity ON sync_event_log(severity, uploaded);
```

**Why store device_info, network_type, battery_level?** These are the first
questions ops asks when debugging sync failures. "What network was the rider on?"
"Was the battery low — did WorkManager skip?" Capturing them at event time
costs ~100 bytes and eliminates a whole investigation step.

---

### 21.4 All Loggable Sync Failure Points

Every place in the codebase that can fail silently, mapped to its event type:

```
Failure Point                           Event Type         Where It Fires
──────────────────────────────────────────────────────────────────────────────
Action write fails (Room exception)     DB_WRITE_FAILURE   PerformTaskActionUseCase
Startup finds IN_PROGRESS entries       STUCK_INPROGRESS   StartupRecoveryUseCase
Batch network error (IOException/5xx)   SYNC_FAILED        SyncManager.syncBatch()
Per-record INVALID_TRANSITION           CONFLICT           SyncManager.syncBatch()
Per-record retry_count hits max         RETRY_EXHAUSTED    SyncManager.syncBatch()
Clock skew rejected (CLOCK_SKEW_REJECTED) CLOCK_SKEW       SyncManager.syncBatch()
Version conflict (VERSION_CONFLICT)     VERSION_CONFLICT   SyncManager.syncBatch()
Payload 413                             PAYLOAD_TOO_LARGE  SyncManager.syncBatch()
HMAC rejected (SIGNATURE_INVALID)       SIGNATURE_INVALID  SyncManager.syncBatch()
Shipment reassigned (REASSIGNED)        REASSIGNED         SyncManager.syncBatch()
Shipment deleted (SHIPMENT_NOT_FOUND)   DELETED            SyncManager.syncBatch()
WorkManager doWork() exhausted retries  WORKER_EXHAUSTED   SyncWorker.doWork()
PENDING count > 20 while offline        OFFLINE_BUILDUP    SyncOrchestrator
Batch partially failed                  SYNC_PARTIAL       SyncManager.syncBatch()
Batch fully succeeded                   SYNC_SUCCESS       SyncManager.sync()
──────────────────────────────────────────────────────────────────────────────
```

---

### 21.5 SyncEventLogger Implementation

```kotlin
class SyncEventLogger @Inject constructor(
    private val syncEventDao: SyncEventDao,
    private val connectivity: ConnectivityObserver,
    private val ntpTime: NtpTimeProvider,
    private val deviceInfoProvider: DeviceInfoProvider,
    private val batteryMonitor: BatteryMonitor,
    private val storageMonitor: StorageMonitor
) {
    /**
     * Log a sync event locally. Always succeeds — never throws.
     * Upload is handled separately by SyncEventUploader.
     */
    suspend fun log(
        eventType: SyncEventType,
        severity: SyncEventSeverity,
        shipmentId: String?      = null,
        actionId: String?        = null,
        pendingCount: Int?       = null,
        retryCount: Int?         = null,
        errorCode: String?       = null,
        errorMessage: String?    = null
    ) {
        runCatching {
            syncEventDao.insert(
                SyncEventEntity(
                    eventId       = UUID.randomUUID().toString(),
                    riderId       = getCurrentRiderId(),
                    eventType     = eventType.name,
                    severity      = severity.name,
                    shipmentId    = shipmentId,
                    actionId      = actionId,
                    pendingCount  = pendingCount,
                    retryCount    = retryCount,
                    errorCode     = errorCode,
                    errorMessage  = errorMessage?.take(512),  // cap at 512 chars
                    deviceInfo    = deviceInfoProvider.snapshot(),
                    networkType   = connectivity.currentNetworkType(),
                    batteryLevel  = batteryMonitor.currentLevel(),
                    freeStorageMb = storageMonitor.freeMb(),
                    occurredAt    = ntpTime.currentTimeMs(),
                    uploaded      = 0
                )
            )
        }
        // runCatching: if DB itself is broken, event logging must not crash the app
    }
}
```

**`DeviceInfoProvider.snapshot()` — captured as a JSON string:**

```kotlin
data class DeviceSnapshot(
    val manufacturer: String,   // "Xiaomi", "Samsung"
    val model: String,          // "Redmi Note 11"
    val osVersion: Int,         // 33 (Android 13)
    val appVersion: String,     // "4.2.1"
    val appBuild: Int           // 412
)
// Serialised to: {"manufacturer":"Xiaomi","model":"Redmi Note 11","os":33,"app":"4.2.1","build":412}
```

---

### 21.6 Local Buffer — Log Before You Send

The event log is **always written locally first**, then uploaded separately.
This is the key design — if the sync itself is broken, we still capture what
happened, and upload the events when the pipe opens even briefly.

```
Sync failure occurs
       │
       ▼
SyncEventLogger.log() → INSERT into sync_event_log (uploaded=0)
       │                  ← this always works (local Room write)
       │
       ▼
SyncEventUploader runs (separate from main SyncManager)
       │
       ├── Triggered by:
       │     - Any network reconnection (T2)
       │     - App foreground (T1)
       │     - After every main sync cycle (piggyback)
       │
       ├── Query: SELECT * FROM sync_event_log
       │          WHERE uploaded = 0
       │          ORDER BY occurred_at ASC
       │          LIMIT 200                 ← max 200 events per upload batch
       │
       ├── POST /v1/sync-events (see 21.7)
       │
       └── On success: UPDATE uploaded = 1 for sent events
           On failure: leave as uploaded = 0, retry next cycle

Cleanup: events with uploaded = 1 and occurred_at > 7 days → DELETE
```

**Why a separate uploader, not piggybacking on SyncManager?**

If SyncManager is the thing that's broken, we can't rely on it to also upload
evidence of its own failure. `SyncEventUploader` is a simpler, independent
coroutine with no Mutex dependency and a much lighter payload.

```kotlin
class SyncEventUploader @Inject constructor(
    private val syncEventDao: SyncEventDao,
    private val apiService: DeliveryApiService
) {
    suspend fun upload() {
        val pending = syncEventDao.getPendingUpload(limit = 200)
        if (pending.isEmpty()) return

        runCatching {
            apiService.uploadSyncEvents(
                SyncEventBatchRequest(events = pending.map { it.toDto() })
            )
            syncEventDao.markUploaded(pending.map { it.eventId })
        }
        // Silent fail — events stay in local buffer if upload fails
    }
}
```

---

### 21.7 Backend Ingest Endpoint

Lightweight, fire-and-forget. Backend never returns errors that block the client.

```
POST /v1/sync-events
Authorization: Bearer <session_token>
Content-Type: application/json

{
  "events": [
    {
      "event_id":       "uuid-v4",
      "rider_id":       "R456",
      "event_type":     "RETRY_EXHAUSTED",
      "severity":       "CRITICAL",
      "shipment_id":    "SHP123",
      "action_id":      "uuid-A1",
      "pending_count":  7,
      "retry_count":    5,
      "error_code":     "INVALID_TRANSITION",
      "error_message":  "Cannot transition DELIVERED → PICKED_UP",
      "device_info":    "{\"manufacturer\":\"Xiaomi\",\"model\":\"Redmi Note 11\",...}",
      "network_type":   "MOBILE",
      "battery_level":  34,
      "free_storage_mb": 412,
      "occurred_at":    1709123456789
    }
  ]
}

Response: 202 Accepted   ← always 202, never 4xx/5xx that would block client
{
  "received": 1,
  "dropped":  0           ← backend may drop malformed events silently
}
```

**Why always 202?** The client must never retry event logging uploads
aggressively. If the backend is down, events stay in local buffer. A 500 should
not cause the client to abandon the buffer or enter retry loops that drain battery.

---

### 21.8 SyncManager — Fully Instrumented

The existing `SyncManager` from Section 6.3, now with event logging wired in
at every failure point:

```kotlin
class SyncManager @Inject constructor(
    private val journalDao: ActionJournalDao,
    private val shipmentDao: ShipmentDao,
    private val apiService: DeliveryApiService,
    private val eventLogger: SyncEventLogger       // ← injected
) {
    private val mutex = Mutex()

    suspend fun sync() = mutex.withLock {
        val pending = journalDao.getPending(limit = 100)
        if (pending.isEmpty()) return@withLock

        pending.chunked(50).forEach { batch -> syncBatch(batch) }

        // Log overall success after all batches complete
        eventLogger.log(
            eventType    = SyncEventType.SYNC_SUCCESS,
            severity     = SyncEventSeverity.INFO,
            pendingCount = journalDao.countPending()
        )
    }

    private suspend fun syncBatch(batch: List<ActionJournalEntity>) {
        val pendingSnapshot = journalDao.countPending()
        journalDao.updateStatus(batch.map { it.actionId }, SyncStatus.IN_PROGRESS)

        val response = try {
            apiService.submitBatch(batch.map { it.toRequest() })
        } catch (e: IOException) {
            journalDao.resetToPending(batch.map { it.actionId })

            // ── EVENT: network failure ────────────────────────────────
            eventLogger.log(
                eventType    = SyncEventType.SYNC_FAILED,
                severity     = SyncEventSeverity.ERROR,
                pendingCount = pendingSnapshot,
                errorCode    = "NETWORK_IO",
                errorMessage = e.message
            )
            return
        } catch (e: HttpException) {
            journalDao.resetToPending(batch.map { it.actionId })

            // ── EVENT: HTTP error (5xx, 413, etc.) ───────────────────
            eventLogger.log(
                eventType    = when (e.code()) {
                                   413  -> SyncEventType.PAYLOAD_TOO_LARGE
                                   else -> SyncEventType.SYNC_FAILED
                               },
                severity     = SyncEventSeverity.ERROR,
                pendingCount = pendingSnapshot,
                errorCode    = e.code().toString(),
                errorMessage = e.message()
            )
            return
        }

        var hadPartialFailure = false

        response.results.forEach { result ->
            when (result.status) {

                "ACCEPTED", "ALREADY_PROCESSED" -> {
                    journalDao.markSynced(result.actionId)
                    // No event — success is the happy path, not worth logging per-record
                }

                "INVALID_TRANSITION" -> {
                    journalDao.markFailed(result.actionId, result.error)
                    hadPartialFailure = true

                    // ── EVENT: state conflict ─────────────────────────
                    eventLogger.log(
                        eventType    = SyncEventType.CONFLICT,
                        severity     = SyncEventSeverity.WARN,
                        shipmentId   = result.shipmentId,
                        actionId     = result.actionId,
                        pendingCount = pendingSnapshot,
                        errorCode    = "INVALID_TRANSITION",
                        errorMessage = result.error
                    )
                }

                "CLOCK_SKEW_REJECTED" -> {
                    journalDao.resetToPending(listOf(result.actionId))

                    // ── EVENT: clock skew ─────────────────────────────
                    eventLogger.log(
                        eventType    = SyncEventType.CLOCK_SKEW,
                        severity     = SyncEventSeverity.WARN,
                        actionId     = result.actionId,
                        errorCode    = "CLOCK_SKEW",
                        errorMessage = "Server time: ${result.serverTimeMs}, sent: ${result.receivedTimeMs}"
                    )
                }

                "VERSION_CONFLICT" -> {
                    hadPartialFailure = true

                    // ── EVENT: optimistic lock mismatch ──────────────
                    eventLogger.log(
                        eventType    = SyncEventType.VERSION_CONFLICT,
                        severity     = SyncEventSeverity.WARN,
                        shipmentId   = result.shipmentId,
                        actionId     = result.actionId,
                        errorCode    = "VERSION_CONFLICT",
                        errorMessage = "client v${result.clientVersion} server v${result.serverVersion}"
                    )
                }

                "SIGNATURE_INVALID" -> {
                    journalDao.markFailed(result.actionId, "Signature rejected")
                    hadPartialFailure = true

                    // ── EVENT: security — log with CRITICAL severity ──
                    eventLogger.log(
                        eventType    = SyncEventType.SIGNATURE_INVALID,
                        severity     = SyncEventSeverity.CRITICAL,
                        actionId     = result.actionId,
                        errorCode    = "SIGNATURE_INVALID"
                    )
                }

                "REASSIGNED" -> {
                    journalDao.markFailed(result.actionId, "Task reassigned")

                    // ── EVENT: reassignment during sync ───────────────
                    eventLogger.log(
                        eventType    = SyncEventType.REASSIGNED,
                        severity     = SyncEventSeverity.WARN,
                        shipmentId   = result.shipmentId,
                        actionId     = result.actionId,
                        errorCode    = "REASSIGNED"
                    )
                }

                "SHIPMENT_NOT_FOUND" -> {
                    journalDao.markFailed(result.actionId, "Shipment deleted on server")

                    // ── EVENT: shipment deleted ───────────────────────
                    eventLogger.log(
                        eventType    = SyncEventType.DELETED,
                        severity     = SyncEventSeverity.INFO,
                        shipmentId   = result.shipmentId,
                        actionId     = result.actionId
                    )
                }

                else -> {
                    val newRetryCount = journalDao.incrementRetry(result.actionId)
                    hadPartialFailure = true

                    if (newRetryCount >= MAX_RETRIES) {
                        journalDao.markFailed(result.actionId, "Max retries exhausted")

                        // ── EVENT: permanently failed ─────────────────
                        eventLogger.log(
                            eventType    = SyncEventType.RETRY_EXHAUSTED,
                            severity     = SyncEventSeverity.CRITICAL,
                            shipmentId   = result.shipmentId,
                            actionId     = result.actionId,
                            pendingCount = pendingSnapshot,
                            retryCount   = newRetryCount,
                            errorCode    = result.status,
                            errorMessage = result.error
                        )
                    }
                }
            }
        }

        if (hadPartialFailure) {
            // ── EVENT: batch partially failed ────────────────────────
            eventLogger.log(
                eventType    = SyncEventType.SYNC_PARTIAL,
                severity     = SyncEventSeverity.WARN,
                pendingCount = pendingSnapshot
            )
        }
    }
}
```

**Other instrumentation points outside SyncManager:**

```kotlin
// StartupRecoveryUseCase — stuck IN_PROGRESS entries
class StartupRecoveryUseCase {
    suspend fun execute() {
        val stuckEntries = journalDao.getInProgress()
        if (stuckEntries.isNotEmpty()) {
            eventLogger.log(
                eventType    = SyncEventType.STUCK_INPROGRESS,
                severity     = SyncEventSeverity.WARN,
                pendingCount = stuckEntries.size,
                errorMessage = "Found ${stuckEntries.size} IN_PROGRESS entries on startup"
            )
            journalDao.resetInProgressToPending()
        }
    }
}

// SyncOrchestrator — offline buildup threshold
class SyncOrchestrator {
    suspend fun onActionRecorded() {
        val pendingCount = journalDao.countPending()
        if (pendingCount >= OFFLINE_BUILDUP_THRESHOLD && !connectivity.isOnline.value) {
            eventLogger.log(
                eventType    = SyncEventType.OFFLINE_BUILDUP,
                severity     = if (pendingCount >= 20) SyncEventSeverity.WARN
                               else SyncEventSeverity.INFO,
                pendingCount = pendingCount,
                errorMessage = "Rider offline with $pendingCount unsynced actions"
            )
        }
    }
}

// PerformTaskActionUseCase — DB write failure
class PerformTaskActionUseCase {
    suspend fun execute(...): Result<Unit> {
        return try {
            withTransaction { /* ... */ }
            Result.success(Unit)
        } catch (e: SQLiteException) {
            eventLogger.log(
                eventType    = SyncEventType.DB_WRITE_FAILURE,
                severity     = SyncEventSeverity.CRITICAL,
                errorCode    = e.javaClass.simpleName,
                errorMessage = e.message
            )
            Result.failure(e)
        }
    }
}

// SyncWorker — WorkManager retries exhausted
class SyncWorker {
    override suspend fun doWork(): Result {
        return try {
            syncManager.sync()
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount >= MAX_WORKER_RETRIES) {
                eventLogger.log(
                    eventType    = SyncEventType.WORKER_EXHAUSTED,
                    severity     = SyncEventSeverity.ERROR,
                    errorCode    = e.javaClass.simpleName,
                    errorMessage = "WorkManager retries exhausted after $runAttemptCount attempts"
                )
                Result.failure()
            } else {
                Result.retry()
            }
        }
    }
}
```

---

### 21.9 Alerting Rules on Backend

Backend processes the event stream and fires alerts based on rules.
All alerts route to ops dashboard + PagerDuty/Slack.

```
Rule 1: RETRY_EXHAUSTED — immediate alert
──────────────────────────────────────────
Trigger:  Any RETRY_EXHAUSTED event received
Severity: P2 (ops must investigate within 1 hour)
Action:   Create ops ticket with rider_id, action_id, shipment_id, error_message
Reason:   Data loss has occurred — action will never be replayed automatically

Rule 2: OFFLINE_BUILDUP — delayed alert
─────────────────────────────────────────
Trigger:  Rider has OFFLINE_BUILDUP events and no SYNC_SUCCESS for > 2 hours
Severity: P3 (investigate within 4 hours)
Action:   Notify rider's hub manager + flag rider in ops dashboard
Reason:   Rider may be stuck offline or app background sync broken

Rule 3: SIGNATURE_INVALID — security alert
───────────────────────────────────────────
Trigger:  Any SIGNATURE_INVALID event
Severity: P1 (immediate — possible fraud/tamper)
Action:   Auto-suspend rider account, alert security team
Reason:   Valid sessions should never produce invalid signatures

Rule 4: STUCK_INPROGRESS — pattern alert
─────────────────────────────────────────
Trigger:  Same rider has STUCK_INPROGRESS on > 3 consecutive app launches
Severity: P3
Action:   Flag device for investigation (OEM kill pattern or crash loop)
Reason:   Repeated crash/kill mid-sync → systemic issue on specific device model

Rule 5: DB_WRITE_FAILURE — critical alert
──────────────────────────────────────────
Trigger:  Any DB_WRITE_FAILURE event
Severity: P2
Action:   Alert rider via FCM to restart app, ops creates ticket
Reason:   Rider cannot record any new actions — workflow fully blocked

Rule 6: SYNC_FAILED fleet pattern
───────────────────────────────────
Trigger:  > 5% of active riders have SYNC_FAILED in the same 10-min window
Severity: P1 (likely backend issue, not rider issue)
Action:   Page on-call backend engineer — possible API outage
Reason:   Correlated failures across riders = server-side root cause

Rule 7: WORKER_EXHAUSTED
──────────────────────────
Trigger:  WORKER_EXHAUSTED from same device manufacturer > 10 times/day
Severity: P3
Action:   Add manufacturer to OEM watchlist, review WorkManager config
Reason:   Specific OEM killing WorkManager aggressively on this model
```

---

### 21.10 Dashboard Metrics

Derived from the sync event stream. Refreshed every 5 minutes.

```
Per-rider view:
  ┌─────────────────────────────────────────────────────────┐
  │  Rider R456 — Sync Health                               │
  │  Last SYNC_SUCCESS:  12 minutes ago                     │
  │  Pending actions:    3                                  │
  │  Unresolved FAILEDs: 1  [View]                          │
  │  Events today:       SYNC_SUCCESS×24, CONFLICT×1        │
  │  Device:             Xiaomi Redmi Note 11, Android 13   │
  │  Network:            MOBILE (last seen)                  │
  └─────────────────────────────────────────────────────────┘

Fleet view (10,000 riders):
  ┌─────────────────────────────────────────────────────────┐
  │  Sync Health — Fleet                                    │
  │  Riders synced in last 30 min:   9,847 / 10,000  (98.5%)│
  │  Riders with OFFLINE_BUILDUP:       47                  │
  │  Riders with RETRY_EXHAUSTED today: 12   [View list]   │
  │  CONFLICT rate today:            0.3% of actions        │
  │  DB_WRITE_FAILURE today:            2 riders            │
  │  SIGNATURE_INVALID today:           0 ✅                │
  │  Top failure reason:             NETWORK_IO (68%)        │
  └─────────────────────────────────────────────────────────┘

Time-series charts:
  - SYNC_SUCCESS rate (p50/p95 latency: created_at → synced_at)
  - SYNC_FAILED count per 10-min bucket
  - OFFLINE_BUILDUP count over time (spikes = network outage zones)
  - RETRY_EXHAUSTED count per day (target: 0)
  - Fleet PENDING action count (aggregate across all riders)
```

---

### 21.11 Trade-offs of Event Logging

| Aspect | Decision | Rationale |
|---|---|---|
| **Separate table from action_journal** | Yes — `sync_event_log` is its own table | Events must survive even if action_journal is partially corrupt |
| **Separate uploader from SyncManager** | Yes — `SyncEventUploader` is independent | If SyncManager is broken, events still get through |
| **Always 202 from backend** | Yes | Client must never retry aggressively or abandon buffer on server error |
| **runCatching in SyncEventLogger.log()** | Yes | Event logging must never crash the app or surface to rider |
| **7-day local retention for uploaded events** | Yes | Debugging window; events uploaded to server are the canonical source |
| **200-event upload batch limit** | Yes | Caps payload size; deeply offline riders may accumulate 1000s of events |
| **No logging of SYNC_SUCCESS per-record** | Intentional | Happy-path is too noisy; one SYNC_SUCCESS per sync cycle is enough |
| **CRITICAL events logged synchronously** | Yes — DB write before returning | RETRY_EXHAUSTED and DB_WRITE_FAILURE must be captured even if app crashes next |
| **Battery/storage in every event** | Yes | Enables correlation: "SYNC_FAILEDs cluster when battery < 20%" |
| **Device info in every event** | Yes | Enables fleet segmentation: "Xiaomi devices have 3× more WORKER_EXHAUSTED" |

---

## 22. Testing Plan & Phased Release

### 22.1 What to Test and Why

Three layers. Each layer has a different job:

```
Unit tests        → logic is correct in isolation
                    Fast. Run on every commit. No device needed.

Integration tests → components work together correctly
                    Room + SyncManager + StateMachine wired up.
                    Run on emulator. Catch wiring bugs unit tests miss.

E2E tests         → full user flow works on a real device
                    Slow. Run on PR merge and before release.
                    Catch what integration tests miss: OEM kills,
                    real network, WorkManager timing.
```

---

### 22.2 Unit Tests

One test class per use case / component. No Android framework dependencies — pure JVM, fast.

| Component | What to test | Key assertions |
|---|---|---|
| `ShipmentStateMachine` | Every valid transition allowed | `ASSIGNED → PICKED_UP` = true |
| | Every invalid transition blocked | `DELIVERED → PICKED_UP` = false |
| | All terminal states are final | No outbound transitions from DELIVERED/FAILED/CANCELLED |
| `QrStructuralValidator` | Valid SHP- format passes | Returns `Pass` with shipmentId |
| | Foreign QR rejected with correct type | URL → `FOREIGN_QR_URL`, UPI → `FOREIGN_QR_PAYMENT` |
| | Checksum failure detected | Garbled suffix → `DAMAGED_QR` |
| | Duplicate in session caught | Same ID twice → `DUPLICATE_SCAN` |
| `AdaptiveSyncScheduler` | Correct tier from PENDING age | age=0 → NONE, age=10min → BACKSTOP |
| | Battery constraint relaxed only when appropriate | age>2hr + count>20 → no battery constraint |
| `NtpTimeProvider` | Returns device time + offset | offset=+5000ms → currentTimeMs is ahead |
| | Falls back to persisted offset when server unreachable | Mock server throws → uses DB value |
| `SyncEventLogger` | Never throws even if DB write fails | Simulated SQLiteException → no crash |
| | Truncates error message at 512 chars | 600-char message → stored as 512 |
| `EmergencyCleanupManager` | Pass 1 deletes only images | Image files gone, DB records intact |
| | Never deletes PENDING journal entries | PENDING entry present → record untouched |
| `ActionJournalDao` (Room in-memory) | `oldestPendingAgeMs` returns null when empty | Empty table → null |
| | Sequence ordering preserved | 3 actions → returned in sequence_number ASC order |

```kotlin
// Example: StateMachine unit test
@Test fun `DELIVERED cannot transition to PICKED_UP`() {
    val machine = ShipmentStateMachine()
    assertFalse(machine.canTransition(ShipmentState.Delivered, ShipmentAction.PickUp))
}

// Example: QR validator
@Test fun `checksum mismatch returns DAMAGED_QR`() {
    val result = QrStructuralValidator.validate("SHP-A1B2C3D4E5-9", emptySet())
    assertEquals(QrRejectionReason.DAMAGED_QR, (result as QrStructuralResult.Reject).reason)
}
```

**Coverage target: 80% line coverage on all use case and domain classes.**
Generated code (DAOs, Hilt modules) excluded.

---

### 22.3 Integration Tests

Run on Android emulator (API 30+). Use real Room (in-memory), fake Retrofit, real WorkManager test utilities.

| Scenario | Setup | Assert |
|---|---|---|
| Offline action → sync on reconnect | Insert PENDING journal entry. Simulate network. Fire SyncManager. | Entry = SYNCED. Shipment state updated. |
| Crash mid-sync recovery | Set 2 entries IN_PROGRESS. Call `StartupRecoveryUseCase`. | Both reset to PENDING. |
| Partial batch failure | Mock server returns ACCEPTED for A1, INVALID_TRANSITION for A2. | A1 = SYNCED, A2 = FAILED, A3 stays PENDING (sequence dependency). |
| Emergency cleanup Pass 1 | Insert 5 SYNCED terminal shipments with image paths. Call `EmergencyCleanupManager`. | Image files deleted. DB records intact. |
| Adaptive scheduler tier change | Insert PENDING entry. Advance fake clock past AGE_STUCK. Call `reschedule()`. | WorkManager has BACKSTOP work enqueued. |
| Captive portal detection | Mock `/v1/ping` returns 302. Mock google.com returns 200. Call `probe()`. | State = OFFLINE. `BACKEND_UNREACHABLE` event logged. |
| Sequence ordering in batch | Insert A1, A3, A2 out of order for same shipment. Build batch. | Batch contains A1, A2, A3 in sequence order. |
| DB cleanup eligibility | Mix of PENDING and SYNCED terminal entries. Run `DbCleanupWorker`. | Only SYNCED terminal entries older than 2 days deleted. PENDING untouched. |

```kotlin
// Example: sync recovery integration test
@Test fun `IN_PROGRESS entries reset to PENDING on startup`() = runTest {
    journalDao.insert(entry(status = IN_PROGRESS))
    journalDao.insert(entry(status = IN_PROGRESS))

    StartupRecoveryUseCase(journalDao, shipmentDao).execute()

    val pending = journalDao.getPending()
    assertEquals(2, pending.size)
}
```

---

### 22.4 E2E Tests

Run on physical device (Xiaomi Redmi Note 11 + Samsung Galaxy A series — top OEM kill offenders). Use real WorkManager. Use a dedicated staging backend.

| Flow | Steps | Assert |
|---|---|---|
| **Full delivery offline** | Enable airplane mode → mark PICKED_UP → OUT_FOR_DELIVERY → DELIVERED → restore network | All 3 actions synced. Server state = DELIVERED. |
| **QR wrong parcel** | Scan a foreign QR (URL) | Instant rejection toast. Camera stays open. No DB write. |
| **QR correct parcel, wrong rider** | Scan parcel assigned to Rider B while logged in as Rider A | `WRONG_RIDER` modal shown. No task created. Audit log entry written. |
| **OEM background kill** | Start sync. Force-kill app via OEM battery settings. Reopen. | IN_PROGRESS entries reset. Sync resumes. No duplicates on server. |
| **Captive portal** | Connect to Wi-Fi hotspot with no internet. Open app. | `CAPTIVE_PORTAL` banner shown. Sync does not attempt. |
| **Storage pressure** | Fill device to < 100MB free. Open app. | CRITICAL banner shown. Image capture blocked. Emergency cleanup fires. |
| **Adaptive T4 escalation** | Go offline. Perform 3 actions. Wait 6 minutes without opening app. | WorkManager BACKSTOP job fired. Entries remain PENDING (no network). No crash. |
| **Reinstall recovery** | Perform 2 deliveries offline. Uninstall. Reinstall. Login. | Server recovery endpoint returns 2 in-progress shipments. UI shows review screen. |

**Device matrix:**

```
Device                  Android   Priority   Why
──────────────────────────────────────────────────────────
Xiaomi Redmi Note 11    12        P0         Top OEM kill offender, high rider usage
Samsung Galaxy A23      13        P0         Second most common rider device
OnePlus Nord CE         11        P1         OPPO/OEM battery optimisation variant
Stock Android emulator  14        P1         Clean baseline for CI
```

**E2E test runner:** Maestro (YAML-based, no Espresso boilerplate, works across OEMs without accessibility service setup).

```yaml
# Example Maestro flow: offline delivery
- launchApp
- tapOn: "Airplane Mode"         # via test helper
- tapOn: "SHP-A1B2C3D4E5"
- tapOn: "Mark Picked Up"
- assertVisible: "Sync Pending"
- tapOn: "Mark Delivered"
- tapOn: "Airplane Mode Off"
- waitForAnimationToEnd
- assertVisible: "Synced ✓"
```
