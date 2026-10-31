# Sanatan Puja Samagri - Loyalty & Membership System Architecture

A physical retail customer loyalty and membership Android application for **Sanatan Puja Samagri**, built with Jetpack Compose, Material 3, and Cloud Firestore. Designed to drive repeat visits to the physical store through digital membership passes, offline-friendly QR customer identification, and verified staff-managed points transactions.

---

### User Review & Critical Decisions

> [!IMPORTANT]
> The following architectural foundations have been confirmed based on your preferences:

- **Confirmed Runtime**: Native Android application in Kotlin with Jetpack Compose & Material 3 (delivers native camera barcode scanning, offline caching, and responsive Devanagari typography).
- **Confirmed Authentication**: Google Sign-In via Jetpack `CredentialManager` (100% free, zero SMS verification costs, unlimited quota on Firebase Spark plan).
- **Confirmed Language Strategy**: Starts with clean, accessible English, with built-in language switching support for Nepali (Devanagari) in app settings.
- **Zero-Budget Invariant**: Strict compliance with Firebase Free Spark quotas. All business rules and integrity checks are enforced directly in Firestore Security Rules and atomic batched writes without requiring paid Cloud Functions or SMS fees.

---

## 1. Overview & Core Concept

**Sanatan Puja Samagri** is a dedicated physical retail puja store. The digital loyalty app replaces paper cards with a reliable digital pass that members present at checkout.

- **Primary Persona**: Retail customers purchasing puja goods, dhoop, ghee, and ritual items in-store. Many users may be non-technical or older adults who require large tap targets, clear typography, and minimal friction.
- **Staff / Admin Persona**: Store attendants and managers operating behind the physical counter, awarding points for in-store bills and processing redemptions using camera barcode/QR scanning.
- **Core Value**: Zero-friction member registration, transparent real-time balance tracking, tamper-proof staff transactions, and direct physical footfall generation through festival offers.

---

## 2. User Experience & Visual Design

### Visual Identity & Theme
- **Color Palette**:
  - Primary / Saffron: `#E8871E` (Warm, sacred, high-visibility actions)
  - Deep Maroon: `#6B1A1A` (Traditional sacred backdrop and header contrast)
  - Background Cream: `#FFF8EC` (Calm, paper-like surface that avoids stark white glare)
  - Gold Accent: `#C9A24B` (Metallic borders, membership tier badges, and digital card trims)
  - On-Primary & Dark Text: `#2B1810` / `#1F140E` (High-contrast, legible typography)
- **Motifs & Styling**:
  - Subtle geometric diya, lotus, and mandala borders strictly reserved for the digital card and header accents.
  - Large touch targets (minimum 48dp x 48dp interactive areas).
  - Explicit high-contrast badge indicators for application status (`Pending Approval`, `Approved`, `Suspended`).

### Key User Flows
```
[ Customer Journey ]
Sign in with Google ──► Enter Name & Phone ──► Pending Approval Screen 
                                                      │ (Admin approves in console/app)
                                                      ▼
                      Digital Pass (QR) ◄── Approved Member Dashboard
                                                      │
[ In-Store Checkout Flow ]                             │
Store Staff scans QR Code ─────────────────────────────┘
  │
  ├─► Verify Member Name & ID (e.g., SPS-000101)
  ├─► Enter Bill Amount / Award Points  (Atomic Batch Write)
  └─► Or Enter Points to Redeem (Validated against balance)
```

---

## 3. Role & Permission Matrix

| Role | Customer Data | Points Award | Points Redeem | Staff Management | Loyalty Settings | Security Audits |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **Customer** | Read own profile & tx | ❌ None | ❌ None | ❌ None | ❌ None | ❌ None |
| **Staff** | Search by ID/Phone, Read card | ✅ `canAward` | ✅ `canRedeem` | ❌ None | ❌ None | ❌ None |
| **Admin** | Full read & approval | ✅ Allowed | ✅ Allowed + Adjust | ✅ Manage staff | ✅ Update rules | ✅ Full audit log |

---

## 4. Final Firestore Schema Specification

All collections operate with explicit data typing, mandatory field validation, and atomic batch constraints:

```
firestore-root
 ├── users/{uid}                          // Approved customer accounts
 ├── membershipRequests/{uid}              // Customer pending registration forms
 ├── staff/{uid}                          // Internal personnel roles and permission flags
 ├── pointTransactions/{txId}             // Immutable, append-only transaction ledger
 ├── loyaltySettings/main                 // Shop-wide conversion rates and counter
 ├── offers/{offerId}                     // In-store promotional festival announcements
 ├── activityLogs/{logId}                 // Administrative audit logs
 └── stats/summary                        // Aggregated counters (no full-collection scans)
```

### Detailed Document Schemas

#### 1. `membershipRequests/{uid}`
- `userId`: `String` (Must match authenticated `request.auth.uid`)
- `name`: `String` (Length between 2 and 60 characters)
- `phone`: `String` (Clean numeric string, 10 digits)
- `status`: `String` (`"pending"` | `"approved"` | `"rejected"`)
- `createdAt`: `Timestamp` (Server timestamp)
- `reviewedBy`: `String?` (Admin UID if approved/rejected)
- `reviewedAt`: `Timestamp?`
- `rejectionReason`: `String?`

#### 2. `users/{uid}`
- `userId`: `String` (Matches document ID and `request.auth.uid`)
- `name`: `String`
- `phone`: `String`
- `membershipId`: `String` (Unique sequential ID, e.g., `"SPS-000123"`)
- `status`: `String` (`"approved"` | `"suspended"`)
- `level`: `String` (`"Bronze"` | `"Silver"` | `"Gold"`)
- `pointsBalance`: `Number` (Integer >= 0)
- `pointsEarned`: `Number` (Cumulative lifetime earned)
- `pointsRedeemed`: `Number` (Cumulative lifetime redeemed)
- `joinedAt`: `Timestamp`
- `updatedAt`: `Timestamp`

#### 3. `staff/{uid}`
- `role`: `String` (`"admin"` | `"staff"`)
- `active`: `Boolean`
- `name`: `String`
- `permissions`: `Map`
  - `canSearch`: `Boolean`
  - `canAward`: `Boolean`
  - `canRedeem`: `Boolean`
- `createdAt`: `Timestamp`
- `createdBy`: `String` (Admin UID)

#### 4. `pointTransactions/{txId}` (Immutable / Append-Only)
- `txId`: `String` (Unique client-generated idempotency key)
- `userId`: `String` (Target member UID)
- `membershipId`: `String`
- `type`: `String` (`"earn"` | `"redeem"` | `"adjust_add"` | `"adjust_deduct"`)
- `points`: `Number` (Integer > 0)
- `billReference`: `String?` (Optional receipt or bill number)
- `reason`: `String` (Mandatory explanation, especially for adjustments)
- `staffId`: `String` (Must match `request.auth.uid`)
- `balanceBefore`: `Number` (Must match user's current `pointsBalance`)
- `balanceAfter`: `Number` (Calculated post-write balance)
- `createdAt`: `Timestamp` (Server timestamp)

#### 5. `loyaltySettings/main`
- `pointsPerUnit`: `Number` (e.g., 1 point per 100 Rs spent)
- `minRedeemPoints`: `Number` (e.g., minimum 50 points to redeem)
- `maxPointsPerTx`: `Number` (e.g., maximum 500 points per transaction)
- `nextMembershipNumber`: `Number` (Atomic sequential counter for `SPS-XXXXXX`)
- `updatedAt`: `Timestamp`

---

## 5. Security & Verification Design (`firestore.rules`)

1. **Default Deny**: All unauthenticated access and unknown collections are completely closed.
2. **Role Verification via Firestore**:
   - `isAdmin()`: `request.auth != null && exists(/databases/$(database)/documents/staff/$(request.auth.uid)) && get(/databases/$(database)/documents/staff/$(request.auth.uid)).data.role == "admin" && get(/databases/$(database)/documents/staff/$(request.auth.uid)).data.active == true`
   - `isStaffWith(perm)`: Checks active staff document and verifies `permissions[perm] == true`.
3. **Atomic Batched Points Integrity**:
   - Any write to `pointTransactions` MUST be accompanied in the same batch by an update to `users/{userId}`.
   - Rules verify:
     - `getAfter(/databases/$(database)/documents/users/$(request.resource.data.userId)).data.pointsBalance == request.resource.data.balanceAfter`
     - `request.resource.data.balanceBefore == get(/databases/$(database)/documents/users/$(request.resource.data.userId)).data.pointsBalance`
     - For redemptions: `request.resource.data.points <= request.resource.data.balanceBefore`
     - Append-only enforcement: `allow update, delete: if false;` on all transactions and activity logs.
4. **Customer Protection**:
   - Customers can only read their own `users/{uid}` and `pointTransactions` where `userId == request.auth.uid`.
   - Customers can NEVER write to their `pointsBalance`, `membershipId`, or `status`.

---

## 6. Step-by-Step Firebase Console Configuration

Before deploying code, perform these one-time configurations in the Firebase Console:

1. **Enable Google Sign-In Provider**:
   - Go to **Firebase Console** -> **Authentication** -> **Sign-in method**.
   - Click **Add new provider** -> Select **Google**.
   - Enable it, specify the support email, and save.
2. **Create the First Administrator Account**:
   - In the app or Firebase Console Auth tab, sign in with your admin Google account once to generate your Auth UID.
   - Go to **Cloud Firestore** -> Click **Start collection** -> Collection ID: `staff`.
   - Document ID: `<YOUR_ADMIN_AUTH_UID>` (Enter your exact Auth UID as the document ID).
   - Add fields:
     - `role`: string `"admin"`
     - `active`: boolean `true`
     - `name`: string `"Owner / Chief Admin"`
     - `permissions`: map
       - `canSearch`: boolean `true`
       - `canAward`: boolean `true`
       - `canRedeem`: boolean `true`
     - `createdAt`: server timestamp
3. **Initialize `loyaltySettings/main`**:
   - In Firestore, create collection `loyaltySettings` -> Document ID: `main`.
   - Add fields:
     - `pointsPerUnit`: number `1`
     - `minRedeemPoints`: number `20`
     - `maxPointsPerTx`: number `1000`
     - `nextMembershipNumber`: number `101`

---

## 7. Phased Implementation Roadmap

- **Phase 1: Foundation, Auth & Customer Enrollment**
  - Firebase Auth & Google Sign-In with Credential Manager integration.
  - Customer membership request flow with validation.
  - "Pending Approval" state holding screen.
  - Admin/Staff role resolution & customer dashboard scaffold.
- **Phase 2: Digital Card, Points Engine & Scanner**
  - Premium Hindu-aesthetic digital membership pass with QR code generation.
  - Camera-based barcode/QR code scanner for staff.
  - Atomic batched earn/redeem transactions with full audit history.
  - Language switcher between English and Nepali (Devanagari).
- **Phase 3: Shop Announcements & Store Analytics**
  - In-store festival offers and member-exclusive announcements.
  - Lightweight summary analytics without scanning full collections.
  - End-to-end security test verification suite.
