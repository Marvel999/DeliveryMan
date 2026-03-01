# Rider Delivery Application — Offline-First Architecture
## Master Design Document (HLD + LLD)

> **Platform:** Android Native | **Version:** 4.0
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
15. [AI Usage](#15-ai-usage)
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
18. [Conflict Resolution — Complete Coverage](#18-conflict-resolution--complete-coverage)
    - 18.1 Audit Summary
    - 18.2 Duplicate Action Submission (Idempotency Conflict) ✅
    - 18.3 Task Already Updated on Server ✅ Extended
    - 18.4 Out-of-Order Actions ✅ New
    - 18.5 Multiple Devices Conflict ✅ Extended
    - 18.6 Task Reassigned While Offline ✅ New
    - 18.7 Task Deleted on Server ✅ New
    - 18.8 Partial Batch Sync Failure ✅
    - 18.9 Clock Skew Conflict ✅ Extended
    - 18.10 Optimistic Lock Version Conflict ✅ New
    - 18.11 Retry After App Reinstall ✅ New
    - 18.12 WorkManager Duplicate Execution ✅ Extended
    - 18.13 Network Flapping ✅ New
    - 18.14 Payload Too Large (Scale Conflict) ✅ New
    - 18.15 Data Corruption in Local DB ✅ Extended
    - 18.16 Security Conflict (Replay Attack) ✅ New

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
│      Count PENDING in journal                                 │
│      ≥ 5 → sync immediately + show banner to rider            │
│                                                               │
│  T4: Periodic fallback (safety net)                           │
│      PeriodicWorkRequest every 15 minutes                     │
│      Catches any missed T1/T2/T3 triggers                     │
│                                                               │
│  All triggers → WorkManager.enqueueUniqueWork("SYNC", KEEP)   │
│  KEEP policy → no duplicate sync jobs ever run in parallel    │
└───────────────────────────────────────────────────────────────┘
```

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

```kotlin
class ConnectivityObserver @Inject constructor(private val context: Context) {
    val isOnline: StateFlow<Boolean> = callbackFlow {
        val mgr = context.getSystemService(ConnectivityManager::class.java)
        val cb  = object : ConnectivityManager.NetworkCallback() {
            override fun onAvailable(network: Network) { trySend(true)  }
            override fun onLost(network: Network)      { trySend(false) }
        }
        mgr.registerDefaultNetworkCallback(cb)
        awaitClose { mgr.unregisterNetworkCallback(cb) }
    }.stateIn(CoroutineScope(Dispatchers.IO), SharingStarted.Eagerly, initialValue = false)
}
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

| Scenario | Handling |
|---|---|
| **App crash during action write** | Startup recovery resets IN_PROGRESS → PENDING; sync resumes on next launch |
| **Duplicate taps (e.g., "Delivered" × 3 quickly)** | Button disabled after first tap + 300ms UI debounce + backend idempotency via action_id |
| **Conflict: Rider marks Delivered, server shows Cancelled** | Backend wins; app shows "Conflict — Contact Support" screen after sync |
| **5+ unsynced actions** | Show blocking alert; restrict further offline delivery actions; pickup remains unrestricted |
| **OEM background kill (Xiaomi/Redmi/Samsung)** | WorkManager foreground service + exponential backoff + T4 periodic fallback |
| **Barcode scanner failure** | Fallback to manual shipment ID entry with backend validation |
| **Multiple riders claim same shipment** | Backend enforces unique assignment; returns `ALREADY_CLAIMED` on sync; app shows conflict |
| **Order cancelled while rider is en route** | FCM push notification → local DB update → delivery completion blocked |
| **Phone storage full** | Monitor bytes available; alert rider; block image capture if critically low |
| **Time manipulation by rider** | NTP offset: all timestamps use server-calibrated time; device clock never trusted |
| **Partial batch sync failure** | Per-record failure marking; failed records retried independently; others continue |
| **Room DB corrupted** | Catch `SQLiteException` on open → export journal via emergency HTTP → prompt reinstall |
| **App uninstalled before sync** | Backend TTL monitor: flag riders with in-flight shipments and no incoming actions for >N hours |
| **Two devices, same rider ID** | Backend invalidates older session token; app detects 401 → re-authenticate |
| **Offline version incompatibility** | API versioning; restrict operations on schema mismatch; force update on network restore |
| **UNVERIFIED_PICKUP rejected after sync** | Mark as REJECTED in DB; notify rider; task is never silently deleted |
| **NTP offset stale (very long session)** | Re-fetch server time on every app foreground event |

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
| **5-action offline restriction** | Limits maximum data loss window | Can frustrate riders in persistent deep-offline zones |
| **Action journal as sync driver** | Full audit trail; replay capability; decoupled from UI | Additional table to maintain; cleanup logic required |
| **UNVERIFIED_PICKUP for offline task creation** | Never blocks rider workflow | Phantom tasks appear until sync resolves them |
| **2-day DB retention post-sync** | Rider can review recent history | DB grows until cleanup runs; 1000+ shipments × 2 days can be significant |
| **Domain + feature modularisation (no full multi-module)** | Faster initial development | Refactoring effort when team/codebase scales |

---

## 14. Observability & Debugging

### Local (In-App Debug)
- Action journal table is fully queryable — support team can pull pending/failed actions remotely
- Sync status visible to rider via banner ("3 actions pending sync")
- Hidden debug screen (long-press app version) shows journal log, sync history, NTP offset

### Backend / Remote
- Every action carries `action_id`, `rider_id`, `timestamp` — complete audit trail
- Backend logs all rejected state transitions → alert on-call team
- Sync latency per rider: `synced_at - created_at` (track p50 / p95 / p99)
- Failed sync rate per rider cohort → flag riders stuck offline > 30 min

### Crash & ANR
- Firebase Crashlytics integration
- Custom breadcrumb logging: each state transition + sync trigger is a breadcrumb event

### Metrics to Monitor
- Pending journal entries per rider (alert ops if > 20)
- Sync success rate (target > 99.5%)
- Average time-to-sync per action
- Image upload failure rate
- State conflict rate (Delivered vs Cancelled)
- UNVERIFIED_PICKUP rejection rate (signals barcode/ID quality issue)
- DB size per rider (alert if > 50 MB)

---

## 15. AI Usage

| Tool | How Used |
|---|---|
| **Claude (Anthropic)** | Architecture brainstorming, trade-off analysis, Kotlin code scaffolding (SyncManager, StateMachine, MVI, cursor pagination) |
| **GitHub Copilot** | Boilerplate generation: DAO interfaces, Room entities, Hilt modules, WorkManager worker setup |
| **ChatGPT** | Edge case enumeration, OEM kill strategy review |

**Designed by engineer:**
Overall architecture layers, state machine transition rules, sync trigger strategy, conflict resolution policy, NTP offset approach, 3-section list structure, UNVERIFIED_PICKUP state concept, cleanup eligibility rules, module structure.

**Accelerated with AI:**
Kotlin code snippets, SQL schema drafts, WorkManager/Paging 3 boilerplate, API contract JSON structure.

**All AI output was reviewed and domain-adapted** — particularly the Mutex SyncManager guard, startup recovery logic, cursor pagination DAO query, and the UNVERIFIED_PICKUP sync resolution flow.

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

*This is the living master document. All new design decisions and clarifications are added as new sections here.*
*Platform: Android Native | Architecture: MVI + Clean + Offline-First | Version: 4.0*

---

## 18. Conflict Resolution — Complete Coverage

### 18.1 Audit Summary

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
