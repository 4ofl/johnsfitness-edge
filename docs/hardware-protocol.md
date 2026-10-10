# BioMax Hardware Relay & Socket Protocol

This document details the hardware integration, TCP/IP socket protocol, and native driver interop powering the physical access control layer in **Johns Fitness Edge**.

---

## 1. Hardware Architecture Overview

The facility operates an on-premise **BioMax biometric access terminal** installed at the entrance turnstile, interfaced with an electronic door strike:

```
[ Reception Edge Server ]
         │  (Local Gym Network)
         ▼
[ Network Switch ]
         │  (TCP Socket / Port 5005)
         ▼
[ BioMax Biometric Terminal ] ──(Door Relay Trigger)──▶ [ Turnstile / Electronic Door Strike ]
         ▲
         │
    [ Optical Fingerprint Sensor / Biometric Scanner ]
```

### Key Technical Specifications
* **Terminal**: BioMax N-Series Biometric Access Terminal
* **Transport**: TCP/IP raw binary socket (Port `5005`)
* **Driver Architecture**: Native 32-bit Win32 C library (`FK623Attend.dll`)
* **Relay Output**: Dry contact relay trigger
* **Pulse Duration**: 5,000ms hardware auto-reset pulse

---

## 2. 32-Bit Native Interop on 64-Bit Windows

The vendor SDK (`FK623Attend.dll`) is compiled exclusively as a **32-bit (x86) unmanaged C dynamic library**. Running a standard 64-bit .NET runtime results in `BadImageFormatException`.

### The Edge Solution
Instead of maintaining an external IPC bridge or slow inter-process proxy, **Johns Fitness Edge** is compiled and executed as a dedicated **32-bit .NET 8 runtime process** (`win-x86`).

```csharp
internal static class FK623Native
{
    private const string DllName = "FK623Attend.dll";

    [DllImport(DllName, CallingConvention = CallingConvention.StdCall)]
    public static extern int FK_ConnectNet(
        int nMachineNo,
        [MarshalAs(UnmanagedType.LPStr)] string pIpAddress,
        int nPort,
        int nTimeOut,
        int nProtocolType,
        int nNetPassword,
        int nLicense);

    [DllImport(DllName, CallingConvention = CallingConvention.StdCall)]
    public static extern void FK_DisConnect(int nMachineNo);

    [DllImport(DllName, CallingConvention = CallingConvention.StdCall)]
    public static extern int FK_SetDoorStatus(int nMachineNo, int nDoorStatus);

    [DllImport(DllName, CallingConvention = CallingConvention.StdCall)]
    public static extern int FK_GetGeneralLogData(
        int nMachineNo,
        ref int pEnrollNumber,
        ref int pVerifyMode,
        ref int pInOutMode,
        ref int pYear,
        ref int pMonth,
        ref int pDay,
        ref int pHour,
        ref int pMinute,
        ref int pSecond);

    [DllImport(DllName, CallingConvention = CallingConvention.StdCall)]
    public static extern int FK_EmptyGeneralLogData(int nMachineNo);
}
```

---

## 3. Remote Door Strike Trigger Protocol

The system allows front-desk staff to trigger an instant door release (e.g., admitting visitors, personal training trials, or staff) directly from the reception UI:

### Command Flow: `DOOR_COMMNAD (Value = 3)`
1. Frontend makes authenticated `POST /api/biometrics/door/unlock`.
2. Controller delegates to `IBiomaxDeviceAdapter.TriggerDoorUnlockAsync()`.
3. The adapter acquires a lease under `BiomaxGlobalLock.Instance`.
4. Executes native driver command:
   ```csharp
   const int DOOR_COMMAND_OPEN = 3;
   int resultCode = FK623Native.FK_SetDoorStatus(deviceMachineNumber, DOOR_COMMAND_OPEN);
   ```
5. Result is validated against error codes (`1 = Success`).
6. The terminal energizes its relay circuit for 5 seconds and resets automatically without requiring an active polling reset loop from the server.

---

## 4. Attendance Polling & Access Synchronization

A background worker service (`BiometricPollingBackgroundService`) continuously maintains state with the terminal:

* **Real-time Attendance Ingestion**: Polls new check-in timestamps periodically using incremental log pointers (`LastProcessedSourceRecordIdentity`).
* **Offline Event Recovery**: If network drops temporarily, logs remain queued in terminal NVRAM (up to 100,000 logs) and back-fill sequentially upon reconnection.
* **Thread Safety via Global Lock**: Door unlock requests take priority over polling loops via the internal `BiomaxGlobalLock` mutex.

---

## 5. Live Remote Enrollment Protocol

Front-desk staff can enroll member fingerprints or facial credentials directly from the web dashboard without navigating the terminal's physical keypad:

```
[ Reception UI ]          [ Edge Service ]          [ BioMax Terminal ]
       |                         |                           |
       |--- POST /face/start --->|                           |
       |                         |--- Acquire Global Lock ---|
       |                         |--- Init Capture Session ->|
       |<-- { sessionId: ... } --|                           |
       |                         |                           |
       |--- GET /face/status --->|                           |
       |    (polling progress)   |--- Query Status Handle -->|
       |<-- { status: Captured } |                           |
       |                         |                           |
       |--- POST /face/commit -->|                           |
       |                         |--- Commit to NVRAM ------>|
       |                         |--- Save Template to DB ---|
       |<-- { success: true } ---|                           |
```

1. **Session Initialization**: `POST /api/biometrics/enrollment/face/start` assigns an internal enrollment identifier and puts the terminal sensor into capture mode.
2. **Real-time Polling**: Receptionist UI monitors sensor progress (`NotStarted` -> `Scanning` -> `Captured`).
3. **Commit Phase**: `POST /api/biometrics/enrollment/face/{sessionId}/commit` extracts the raw template, stores a biometric vault snapshot, and links the credential to the member profile.

---

## 6. Biometric Vault Snapshots & Fleet Restoration

To protect against hardware theft, terminal failure, or accidental factory reset:
* **Automated Snapshots**: `BiomaxBiometricVaultBackgroundService` periodically extracts all enrolled biometric templates from terminal memory into local encrypted SQLite storage.
* **1-Click Fleet Restoration**: `POST /api/biomax/vault/restore-fleet` pushes the entire credential archive back onto any replacement terminal in minutes, restoring full gym turnstile operation without requiring 1,405 members to re-enroll.
