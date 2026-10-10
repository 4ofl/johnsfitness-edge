# Personal Training & Coaching Management

This document specifies the personal training (PT) module, trainer allocation workflows, and session lifecycle management in **Johns Fitness Edge**.

---

## 1. Personal Training Domain Model

Personal training operates as a dedicated subsystem integrated with member access and trainer payroll:

* **Plan Type**: Plans are categorized via `PlanKind.PersonalTraining` (distinguished from general gym floor memberships).
* **Prerequisite Rule**: Personal training can only be assigned to a member whose general membership status is **Active** (`member.Status == MemberStatus.Active`).
* **Payment Requirement**: A trainer can only be linked to a member after the personal training invoice is marked **Paid** (`request.PaymentStatus == InvoiceStatus.Paid`).

---

## 2. Trainer Allocation & Session Tracking

When a personal training package is purchased:
* A `PersonalTrainingAssignment` record is created linking `MemberId`, `StaffUserId` (Trainer), and `MembershipId`.
* Tracks package parameters:
  * Total Allocated Sessions (e.g. 12, 24, or 36 sessions)
  * Start Date and Package Expiry Date
  * Assigned Trainer Name and Specialty
  * Commission Structure / Trainer Payout Metadata

### Session Check-In Linking
When a personal training client verifies their biometric scan at the turnstile, the attendance engine records both general floor attendance and decrements the active personal training session counter, providing automated quota validation.

---

## 3. PT Renewal Lifecycles & Automated Reminders

* **Expiry Monitoring**: Packages approaching completion (either by date or when remaining sessions reach $\le 2$) appear on the front-desk PT renewal dashboard.
* **Renewal Flow**: Receptionists initiate renewal with 1 click, preserving trainer continuity while generating a new invoice.
* **WhatsApp PT Reminders**: Client-side WhatsApp generator creates personalized coaching renewal alerts with session balances directly to the member's phone.
