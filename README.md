# BuildStock — Vijay Balaji Agency

[![CI](https://github.com/Hem1234567/Inventory-management-System-bala/actions/workflows/ci.yml/badge.svg)](https://github.com/Hem1234567/Inventory-management-System-bala/actions/workflows/ci.yml)

Repository: **Inventory-management-System-bala** — every push runs the GitHub Actions verification workflow (format, analyze, tests, APK).

Offline-first building-materials management and billing for **Vijay Balaji Agency**, Chennai.
Flutter (Android) · Riverpod · GoRouter · Drift/SQLite · Firebase Auth + Cloud Firestore.

> The application never ships sample products, customers, invoices or statistics.
> Every dashboard figure and document is derived from records the business enters.

---

## 1. What is implemented

| Area | Status |
| --- | --- |
| Business profile, logo/signature references, UPI ID, invoice prefix | Implemented |
| Products, categories, brands, units, SKU/barcode, GST rate, price lists | Implemented |
| Inventory ledger, stock projection, adjustments, low-stock detection | Implemented |
| Stores and godowns | Implemented (transfers schema present, transfer UI pending) |
| Customers/suppliers, balances, ledgers, credit limits | Implemented |
| Retail/wholesale billing, GST/non-GST, discounts, partial payments | Implemented |
| Exact Vijay Balaji Agency A4 estimate/bill renderer | Implemented |
| Invoice preview, structured snapshots, cloud retrieval by bill number | Implemented |
| PDF download, print, share, PNG image export | Implemented |
| A4, 58 mm and 80 mm thermal PDFs | Implemented |
| UPI payment QR and invoice reference QR | Implemented (no gateway) |
| Purchases, supplier payments, stock-in | Implemented |
| Expenses, cash book, day book, receivables/payables | Implemented |
| Staff, roles, attendance, payroll generation and salary payments | Implemented |
| Drivers, vehicles, deliveries, partial delivery, proof metadata | Implemented (assignment/status; photo capture is local-ref based) |
| Reports (sales, GST, stock value, outstanding, low stock) + CSV export | Implemented |
| Durable outbox, exponential backoff, idempotent writes, incremental pull | Implemented |
| Firestore security rules (single file) | Implemented |
| English + Tamil UI and document font | Implemented |
| Unit/widget tests (26) | Implemented |
| Image hosting upload pipeline | **Not wired** — metadata table and local references exist; provider upload is documented as the next step |
| Stock transfer UI, returns/credit-note UI, quotations UI | Repository/schema present, UI not yet built |

This README states the boundary honestly so a feature is never claimed complete
when its UI or verification is missing.

---

## 2. Architecture

Feature-first clean architecture:

```
lib/
  app/                 theme, router, shell, preferences
  core/
    audit/             immutable audit writer
    database/          Drift schema, migrations, providers
    domain/            shared enums and status strings
    errors/            typed AppFailure
    firebase/          initialization and auth wrapper
    qr/                UPI and invoice-reference payloads
    security/          permission catalog and role defaults
    session/           device identity and shop binding
    sync/              outbox, backend, engine, local mirror, controller
    utils/             money, quantity, dates, ids, json, validators
    widgets/           design-system components and forms
  features/            dashboard, sales, invoices, products, inventory,
                       customers, suppliers, purchases, accounts, stores,
                       staff, drivers, deliveries, reports, settings, sync
  l10n/                English and Tamil ARB files
```

Rules enforced in code:

* UI never imports `cloud_firestore`; only `FirestoreSyncBackend` and the
  invoice-retrieval path in `SalesRepository` touch it.
* Money is **integer paise**; quantities are **integer milli-units**; tax rates
  are **basis points**. No business amount is a `double`.
* Every business write and its outbox entry share one SQLite transaction.
* Finalized invoices are immutable. Corrections use cancellation and returns.
* Stock is an append-only ledger; the `inventories` row is a projection.
* Firestore rules independently enforce roles, immutability and idempotency.

### Local database

Drift, schema version **1**, 41 tables, indices on search fields, foreign keys
enabled, WAL journal mode. Migrations are sequential (`_runMigration`) and new
versions must append a step rather than edit an old one.

Configuration seeded on first launch (editable, not business records):

* Units: Nos, Bag, Ton, Cft, Cum, Kg, Load, Ltr, Sqft, Pc
* Categories: Cement, M-Sand, Sand, Jelly, Blue Metal, Bricks, AAC Blocks, Other
* Expense categories: Rent, Electricity, Transport, Vehicle Fuel, Vehicle
  Maintenance, Staff Salary, Driver Salary, Telephone, Office Expenses,
  Loading and Unloading, Other

No product, customer, supplier, staff, invoice or payment is seeded.

### Invoice snapshots

`Sales` stores `sellerSnapshotJson`, `buyerSnapshotJson` and per-line product
name/SKU/HSN/unit/rate/tax snapshots. The document is always regenerated from
these snapshots, so changing a product price or the business profile cannot
alter an issued bill.

Invoice numbers are device-aware:
`VBA-2026-DEVXXX-000001`, with the sequence scoped by document type, device code
and year in `number_sequences` inside the same transaction as the invoice.

### Synchronization

* `sync_queue` is the durable outbox (`pending → syncing → synced | failed |
  conflict`).
* Each operation has a stable `operationId`. Documents use client-generated IDs,
  so retries overwrite the same Firestore document rather than duplicate it.
* Firestore writes run in a transaction and fail with a conflict when the remote
  `version` is newer than the local `baseVersion`.
* Pull is incremental per collection using an ISO `updated_at` cursor.
* Applying a remote document never overwrites an entity with a pending local
  operation; it is marked `conflict` and shown in Sync Center.
* Retry uses 30 s → 2 m → 8 m → 30 m → 2 h → 6 h backoff.
* Connectivity, app start and app resume trigger a run; connectivity alone is
  never treated as proof that Firestore is reachable.

---

## 3. Prerequisites

* Flutter **3.47.9** or newer (Dart 3.12) — this project was built and analyzed
  with that SDK.
* Android SDK **36** and build-tools 36.
* JDK **17–24** for Gradle 8.14. JDK 25 is not supported by that Gradle version.
* A Firebase project (the supplied one is `inventory-management-be3d1`).

```bash
flutter --version
flutter doctor
```

---

## 4. Generate code and run

Drift uses code generation. The generated database and localization files are committed so a fresh clone analyzes immediately; regenerate them after changing a table or ARB file.

```bash
flutter pub get
dart run build_runner build
flutter analyze
flutter test
flutter run
```

`flutter gen-l10n` runs automatically during `pub get`/build; the generated
localizations land in `lib/l10n/generated/`.

### Verify

```bash
dart format --output=none --set-exit-if-changed lib test
flutter analyze
flutter test
flutter build apk --release
```

---

## 5. Firebase setup

### 5.1 Web (already configured)

The supplied web app configuration is embedded in
`lib/core/firebase/firebase_runtime.dart`. It is safe to ship: Firebase web API
keys identify the project, they are not service-account secrets.

### 5.2 Android (required for cloud sync on phones)

A web app ID cannot authenticate an Android client. Register an Android app in
the same Firebase project:

1. Firebase console → Project settings → **Add app** → Android.
2. Package name: `com.vijaybalaji.buildstock` (see `android/app/build.gradle.kts`).
3. Download `google-services.json` if you prefer the Gradle path, **or** copy the
   generated `mobilesdk_app_id` and pass it at build time:

```bash
flutter build apk \
  --dart-define=FIREBASE_ANDROID_APP_ID=1:800582767727:android:xxxxxxxxxxxx
```

Until this is supplied, the app starts in a fully functional **offline mode**;
the setup and Sync Center screens explain the missing step. No fake credentials
are invented.

4. Enable **Email/Password** authentication.
5. Create Firestore in production mode.
6. Keep the supplied SHA-1/SHA-256 fingerprints if you later add Google sign-in.

### 5.3 Deploy rules and indexes

`firestore.rules` is the single required rules file and starts with
`rules_version = '2';`.

```bash
firebase deploy --only firestore:rules,firestore:indexes
```

The rules:

* deny everything not explicitly listed;
* scope every document to `shops/{shopId}` and an active membership document;
* make `stock_movements`, `payments`, `salary_payments` and `audit_logs`
  append-only;
* prevent financial edits to finalized invoices and payroll;
* require a deterministic document ID and an idempotency key on payments;
* allow the owner to bootstrap `shops/{uid}` + `users/{uid}` only for their own
  UID, and staff to join through a single-use invite addressed to their email.

### 5.4 First cloud shop

Sign in on the first device with the owner account, save the business profile,
then run **Sync Center → Sync now**. The app creates the shop document and the
owner membership; subsequent devices sign in with invited accounts.

### 5.5 Current Firebase plan constraint

Firebase Cloud Storage requires Blaze. This app does **not** use Firebase
Storage. Images are intended to live on an external host; Firestore stores only
metadata and URLs. The upload pipeline is not wired in this build (see
limitations), so invoice rendering uses the local file reference or text only.

---

## 6. Invoice document

The renderer reproduces the supplied paper format in English:

* `ESTIMATE` / `BILL` heading, two contact numbers top-right;
* centered `VIJAY BALAJI AGENCY`, dealer line, wholesale/retail line, address;
* divider, date and bill number;
* buyer name, phone, address, GSTIN;
* bordered table: S.No, Description, Qty, Rate, Amount;
* Subtotal, Discount, Taxable, CGST, SGST, IGST, Grand Total, Paid, Balance;
* amount in words, payment mode, UPI QR, authorized signature, terms.

Editable in **Settings → Business profile**: name, phones, description, address,
logo/signature/stamp references, GSTIN, PAN, state code, UPI ID, bank details,
invoice prefix. Seller fields are snapshotted onto each finalized invoice.

Tamil text uses the embedded Hind Madurai font in PDFs. The PDF library does not
perform full complex-script shaping; for Tamil thermal output the recommended
fallback is bitmap rendering, which is not yet automated. On-screen Tamil
rendering uses Flutter's text engine and shapes correctly.

---

## 7. UPI QR (no gateway)

`UpiQr.paymentUri` builds a standard UPI intent:

```
upi://pay?pa=<upiId>&pn=<payee>&am=<amount>&cu=INR&tn=<invoice>&tr=<invoiceId>
```

The QR is rendered locally with `qr_flutter` (screen) and `pdf` (print). Displaying
or scanning the QR **never** marks an invoice paid. Staff must use **Record
payment** and confirm receipt. There is no Razorpay/Stripe/Paytm integration.

The invoice reference QR contains only
`buildstock://invoice/<invoiceId>` — no customer name, phone or amount.

---

## 8. Building Android

```bash
# JDK 17-24 must be active for Gradle 8.14
export JAVA_HOME=/path/to/jdk-21   # PowerShell: $env:JAVA_HOME = 'D:\jdk21'

flutter build apk --release \
  --dart-define=FIREBASE_ANDROID_APP_ID=1:800582767727:android:xxxx
```

Release signing: create `android/key.properties` (git-ignored):

```properties
storePassword=...
keyPassword=...
keyAlias=upload
storeFile=/absolute/path/upload-keystore.jks
```

`android/app/build.gradle.kts` falls back to debug signing when the file is
absent, so CI can build an unsigned/debug artifact; a production release must
provide the keystore.

Required Android permissions are declared in `AndroidManifest.xml`: internet,
camera, images, notifications, Bluetooth (thermal printers).

---

## 9. Tests

```bash
flutter test
```

Covered: money parsing/formatting/words, quantity precision, GST/discount/rounding
arithmetic, invoice numbering and device codes, role permissions, and the A4
invoice widget for both bill and estimate.

Not yet covered by automated tests: database transactions, sync engine against a
fake backend, PDF byte generation, and end-to-end multi-device flows. The
`test/` structure is ready for them; CI runs everything present.

---

## 10. Project conventions

* All new monetary UI input is parsed with `Money.tryParseRupees` and stored as
  `int` paise.
* All quantities use `Quantity.tryParseMilli`.
* New tables require a schema version bump and a new migration case.
* New business writes must call `Outbox.enqueue` in the same transaction.
* New Firestore collections require rules, index definitions and a `SyncCollections`
  entry.
* Never put a secret in `lib/` or commit `key.properties`, `google-services.json`
  or service-account JSON.

---

## 11. Known limitations / next steps

1. **Android Firebase app ID** must be registered; until then cloud sync is
   unavailable and offline mode is used.
2. **Image hosting upload** is not wired. `image_metadata` rows, local paths and
   checksums exist; add an ImgBB/S3/Cloudinary adapter and a queue worker.
3. Stock transfer, return/credit-note and quotation screens exist in the data
   layer but not the UI.
4. PDF Tamil shaping uses the font's glyphs without HarfBuzz; verify with the
   printer/business before relying on Tamil thermal bills.
5. Automated integration tests for sync/multi-device conflicts are pending.
6. Firestore rules should be exercised in the Firebase emulator before
   production deployment.

---

## 12. License and ownership

Built for Vijay Balaji Agency. Source delivered to the business; no third-party
sample data is included.
#   b a l u - s h o p - 0 4  
 