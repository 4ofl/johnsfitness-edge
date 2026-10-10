# Legacy Database Reverse Engineering & Clean Cutover

This document outlines the technical process used to extract, decrypt, and migrate 10+ years of legacy gym records from **GymSoft / FitnessSoft** into **Johns Fitness Edge**.

---

## 1. Legacy Architecture & Encryption Challenge

The legacy software stored data across encrypted Microsoft Jet 4.0 Access databases wrapped in proprietary backup packages.

```
Legacy Backup Package
  ├── NGym_Vcnl.dl  (Jet 4.0 MDB - Main Operational & Registration Data)
  ├── wNet_cnl.dl   (Jet 4.0 MDB - Network & Machine Logs)
  └── GWincnl.dl    (Jet 4.0 MDB - Gateway Configurations)
```

### Decryption & Reverse Engineering
The database was protected with a database-level Jet engine password. Through binary analysis and configuration extraction, the encryption key was recovered, enabling full extraction via OLEDB/ODBC.

---

## 2. Data Extraction Statistics

A dedicated automated pipeline extracted all relational tables to normalized snapshots:

| Table | Legacy Purpose | Extracted Rows |
| :--- | :--- | :--- |
| `registration` | Member personal records & ID cards | 1,405 |
| `reg_particulars` | Subscription purchases & renewals | 5,232 |
| `acc_tra` | Accounting transactions & day book | 5,410 |
| `fp_verify` | Fingerprint verification logs | 243 |
| `fp_expiry` | Biometric hardware push checkpoints | 1,193 |
| `staff` | Receptionists and trainer logins | 8 |
| `bmi` | Historical weight and biometric metrics | 1,072 |

---

## 3. The Clean Slate Migration Strategy

### The Anti-Pattern: Full Ledger Import
In long-running gyms, copying 10 years of unstandardized cash logs directly into a new modern billing system creates massive operational overhead:
* Historical invoices lack modern GST tax breakdowns and tender split accounting.
* Day Book cash drawer reconciliations get distorted by past years.
* Staff searching for active members get flooded with legacy records.

### The Solution: Clean Cutover
Instead of polluting the active general ledger:
1. **1,405 Member Profiles Imported**: Full names, registration codes (`JF-1` .. `JF-1412`), normalized 10-digit mobile numbers, DOB, blood group, and contact details.
2. **Latest Membership Plan Preserved**: Evaluated each member's most recent subscription:
   * **145 Active Members**: Plan validity preserved into the future (door access recognized immediately).
   * **1,260 Inactive Members**: Last plan and expiration date recorded as inactive history. When they return, reception clicks **"Renew"** in one second.
3. **Fresh General Ledger (0 Historical Invoices)**: Payments, receipts, and Day Book cash registers start from Day 1 for clean shift reconciliation and GST compliance.
4. **Per-Person Front-Desk Biometric Enrollment**: Every member initializes in an `Unlinked` biometric state, allowing front-desk receptionists to enroll fingers on the BioMax reader cleanly as members visit.
