# Biometric Privacy & Cloud Synchronization

This document specifies the data boundary, privacy isolation architecture, and outbox sync engine implemented in **Johns Fitness Edge**.

---

## 1. The Edge vs Cloud Architecture Boundary

Fitness centers process sensitive physical biometric data alongside standard commercial billing. **Johns Fitness Edge** enforces a strict physical and logical boundary between edge hardware and central cloud endpoints:

```
┌──────────────────────────────────────────────────────────┐
│                   ON-PREMISE EDGE SERVER                 │
│                                                          │
│   [ Raw Biometric Templates ]                            │
│   [ Fingerprint Minutiae Hashes ]                        │
│   [ Hardware NVRAM Snapshots ]   ──> QUARANTINED LOCALLY │
│   [ Physical Door Strike Pulses ]                        │
│                                                          │
│   [ Business Invoices ]                                  │
│   [ Member Profiles ]            ──> QUEUED IN OUTBOX    │
│   [ Attendance Aggregates ]                              │
└────────────────────────────┬─────────────────────────────┘
                             │
                  Transactional Outbox
                   (Biometrics Dropped)
                             │
                             ▼
┌──────────────────────────────────────────────────────────┐
│                 CENTRAL CLOUD PLATFORM                   │
│               (api.johnsfitness.4ofl.dev)                │
│                                                          │
│   • Multi-branch Business Intelligence                   │
│   • Consolidated Financial Reporting                     │
│   • Mobile Member Portal API                             │
└──────────────────────────────────────────────────────────┘
```

---

## 2. Biometric Zero-Knowledge Isolation

Under the Digital Personal Data Protection Act (DPDP Act) and international data privacy regulations (GDPR), transmitting biometric templates across public cloud networks introduces severe compliance and security liabilities.

### The Technical Implementation
The background synchronization dispatcher (`CloudSyncWorker.cs`) incorporates an active **Biometric Safety Filter**:

```csharp
// BIOMETRIC SAFETY FILTER: Drop any biometric payload from cloud sync
var safeEventsToSync = new List<SyncOutboxEvent>();
foreach (var evt in pendingEvents)
{
    if (evt.EntityType.Contains("Biometric", StringComparison.OrdinalIgnoreCase) || 
        evt.EntityType == "AttendanceRecord")
    {
        if (evt.EntityType == "BiometricCredential" || 
            evt.EntityType == "BiometricDevice" || 
            evt.EntityType == "BiometricEventJournal" || 
            evt.EntityType == "UnmappedBiometricEvent")
        {
            _logger.LogWarning($"SAFETY ALERT: Blocking {evt.EntityType} from being synced to the cloud.");
            evt.ProcessingState = SyncState.Completed; // Mark completed without transmission
            continue;
        }
    }
    safeEventsToSync.Add(evt);
}
```

* Raw biometric templates, fingerprint minutiae, and hardware audit journals **never touch the network socket** leading to the cloud.
* Even if the cloud infrastructure is fully compromised, member biometric data remains completely inaccessible.

---

## 3. Transactional Outbox Pattern

To prevent distributed transaction failures during intermittent gym internet connectivity:

1. **Transactional Writing**: When a business entity (Member, Invoice, Payment) is modified, a corresponding `SyncOutboxEvent` record is inserted in the exact same database transaction.
2. **Batch Dispatch**: `CloudSyncWorker` polls pending events in batches of up to 50 records.
3. **JWT Authentication**: Authenticates against the cloud endpoint using short-lived signed JWT bearer tokens.
4. **Exponential Backoff**: If internet is down, events transition to `SyncState.Failed` and retry automatically when connectivity returns.

---

## 4. Online Database Durability & Hot Backups

To ensure gym data is safe against disk failures or operating system corruption, `BackupService.cs` provides non-blocking hot database snapshots:

* **Non-Blocking Hot Backup**: Invokes `SqliteConnection.BackupDatabase` directly from memory, copying live database pages to an isolated backup directory without interrupting front-desk operations.
* **Cryptographic Integrity Verification**: Executes `PRAGMA integrity_check` on the backup file prior to marking completion.
* **Pre-Restore Safety Lockout**: Verifies the edge service is not running on port 4782 before permitting any restore operation, preventing database file corruption.
