# API Performance Audit — Gujarat Pharmacy Council Backend

## Summary

The system downtime was caused by Apache worker exhaustion. The server was configured with a maximum of 150 concurrent worker processes (prefork MPM). Under traffic load (~500 concurrent users), all 150 workers became occupied with long-running PHP requests that weren't completing fast enough, leaving zero workers available to serve new incoming requests — effectively making the site unresponsive.

The database (RDS 56GB, 1.5GB database) was confirmed idle during the outage with only 3 active connections — the bottleneck is entirely at the application/web server layer.

## Infrastructure Details

| Component | Spec |
|-----------|------|
| EC2 Instance | m4.xlarge (4 vCPU, 16GB RAM) |
| Web Server | Apache 2.4.52 (prefork MPM) |
| PHP | 8.1.2 |
| Framework | Laravel 8.83.29 |
| Database | AWS RDS MySQL (56GB instance, 1.5GB data) |
| Queue Driver | `sync` (all jobs run inline in HTTP request) |
| Cache Driver | `file` (no Redis or Memcached) |

## Immediate Actions Taken

- Increased `MaxRequestWorkers` from 150 to 200 (safe maximum for 16GB RAM at ~58MB per worker)
- Added `MaxConnectionsPerChild=1000` to recycle workers and prevent memory leaks
- Restarted Apache to clear all stuck workers and restore service
- Implemented automated watchdog cron (runs every minute): graceful reload at 150 workers, forced restart at 180 workers

---

## CRITICAL — APIs That Block Workers for 3-10+ Seconds

These perform PDF generation, S3 upload, and/or email sending synchronously inside the HTTP request.

### Exam Submission (Highest Risk During Events)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST save-exam` | `CollegeController::saveExam()` | Dispatches `ProcessRefresherExam` job, but `QUEUE_CONNECTION=sync` means it runs inline: DB transaction → `lockForUpdate` → delete/insert exam answers → calculate results → update slot → PDF certificate generation → S3 upload → DB update. 500 users submit exams simultaneously. | Change `QUEUE_CONNECTION` to `database`. Set up a jobs table and run a Supervisor-managed queue worker. The job is already written as `ShouldQueue` — it just needs a real queue driver to run in the background. |

### Slot Booking (High Risk During Events)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST book-refresher-slot` | `CollegeController::bookRefresherCourseSlot()` | DB transaction + `lockForUpdate` + create booking + update slot counts + email via synchronous Mailgun curl. The database lock is held while the email is being sent (1-3 seconds). | Move the `directMail()` call after `DB::commit()`. The email doesn't need to be inside the transaction. Optionally queue the email as a job. |
| `POST book-offline-refresher-slot` | `CollegeController::bookOfflineRefresherCourseSlot()` | Same pattern as above. | Same fix — move email after commit, optionally queue it. |

### Certificate & Smart Card Downloads

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET validity-certy-download` | `UserController::ValidityCertificateDownload()` | PDF generation + S3 upload + DB update per request. Always regenerates. | Check if certificate already exists on S3 before regenerating. Return cached URL if available. Queue the generation if not cached. |
| `GET DownloadProvisionalCertificate/{srNo}` | `CertificateController::DownloadProvisionalCertificate()` | PDF generation + S3 upload if `file_url` is null. | Already has caching logic — no change needed unless called in bulk. |
| `GET DownloadOriginalCertificate/{srNo}` | `CertificateController::DownloadOriginalCertificate()` | PDF generation + S3 upload if not cached. | Same — already has caching logic. |
| `GET DownloadDegreeCertificate/{srNo}/{rno}` | `CertificateController::DownloadDegreeCertificate()` | PDF generation + S3 upload. | Add caching check like provisional/original certificates. |
| `POST download-good-standing` | `CertificateController::downloadGoodStanding()` | PDF generation + S3 upload. | Add caching check. Queue generation if not cached. |
| `POST DownloadSmardCard` | `MasterController::DownloadSmardCard()` | PDF + QR code generation + S3 upload — always regenerates, no caching. | Add caching: check if `smart_card_url` already exists and S3 file is valid before regenerating. |

### Certificate Issuance with Email + SMS

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST saveProvisionalDetails` | `CertificateController::saveProvisionalDetails()` | DB update + email via Mailgun curl. | Queue the email as a background job instead of sending synchronously. |
| `POST saveOriginalCertificateDetails` | `CertificateController::saveOriginalCertificateDetails()` | DB update + email via Mailgun curl + SMS via MSG91 curl — two sequential external HTTP calls. | Queue both email and SMS as background jobs. Return response immediately after DB update. |
| `POST saveDuplicateCertificateDetails` | `CertificateController::saveDuplicateCertificateDetails()` | DB updates + email via Mailgun curl. | Queue the email. |

### Letter Downloads (8 Types)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET DownloadNocLetter/{id}` | `LetterController::DownloadNocLetter()` | PDF generation + S3 upload if `file_url` is null. | Already has caching logic. No immediate change needed. |
| `GET DownloadRuhsLetter/{id}` | `LetterController::DownloadRuhsLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadNIOSLetter/{id}` | `LetterController::DownloadNIOSLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadVerificationLetter/{id}` | `LetterController::DownloadVerificationLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadReminderLetter/{id}` | `LetterController::DownloadReminderLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadCancellationLetter/{id}` | `LetterController::DownloadCancellationLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadTransferLetter/{id}` | `LetterController::DownloadTransferLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadDataflowLetter/{id}` | `LetterController::DownloadDataflowLetter()` | Same pattern. | Same — already cached. |
| `GET DownloadTrainingLetter/{id}` | `LetterController::DownloadTrainingLetter()` | Same pattern. | Same — already cached. |

### Report Downloads (14+ Types)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST DownloadApplicationRegister` | `ReportsController::DownloadApplicationRegister()` | Queries data + PDF generation + S3 upload. | Queue report generation as a background job. Return a "generating" status and let the user poll or get notified when ready. |
| `POST DownloadFeeRegister` | `ReportsController::DownloadFeeRegister()` | Same pattern. | Same — queue as background job. |
| `POST DownloadFreshApplicationRecord` | `ReportsController::DownloadFreshApplicationRecord()` | Same pattern. | Same — queue as background job. |
| `POST DownloadDegreeDetails` | `ReportsController::DownloadDegreeDetails()` | Same pattern. | Same — queue as background job. |
| `POST DownloadRegistrationValidation` | `ReportsController::DownloadRegistrationValidation()` | Same pattern. | Same — queue as background job. |
| `POST DownloadRegistrationCancellation` | `ReportsController::DownloadRegistrationCancellation()` | Same pattern. | Same — queue as background job. |
| `POST DownloadApplicationDetails` | `ReportsController::DownloadApplicationDetails()` | Same pattern. | Same — queue as background job. |
| `POST DownloadJobList` | `ReportsController::DownloadJobList()` | Same pattern. | Same — queue as background job. |
| `POST DownloadInOutRegister` | `ReportsController::DownloadInOutRegister()` | Same pattern. | Same — queue as background job. |
| `POST DownloadProvisionalOriginalCertificate` | `ReportsController::DownloadProvisionalOriginalCertificate()` | Same pattern. | Same — queue as background job. |
| `POST DownloadCollegeWiseReg` | `ReportsController::DownloadCollegeWiseReg()` | Same pattern. | Same — queue as background job. |
| `POST DownloadGovernmentReport` | `ReportsController::DownloadGovernmentReport()` | Same pattern. | Same — queue as background job. |
| `POST DownloadDDReport` | `ReportsController::DownloadDDReport()` | Same pattern. | Same — queue as background job. |
| `POST DownloadElectrolApplicationReport` | `ReportsController::DownloadElectrolApplicationReport()` | Same pattern. | Same — queue as background job. |

### Financial Report Downloads (5 Types)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST DownloadAccountLedger` | `FinancialReportsController::DownloadAccountLedger()` | Queries ledger data + PDF generation + S3 upload. | Queue as background job. |
| `POST DownloadCashBookDetail` | `FinancialReportsController::DownloadCashBookDetail()` | Same pattern. | Same — queue as background job. |
| `POST DownloadBankBookDetail` | `FinancialReportsController::DownloadBankBookDetail()` | Same pattern. | Same — queue as background job. |
| `POST DownloadJvBookDetail` | `FinancialReportsController::DownloadJvBookDetail()` | Same pattern. | Same — queue as background job. |
| `POST voucherGeneration` | `FinancialReportsController::voucherGeneration()` | Same pattern. | Same — queue as background job. |

### Scrutiny Operations

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST saveScrutinyData` | `Scrutiny/DashboardController::saveScrutinyData()` | DB updates + PDF generation + S3 upload + email + SMS — all synchronous. | Queue the PDF generation, email, and SMS as background jobs. Return response after DB update. |
| `POST saveReEntryScrutinyData` | `Scrutiny/DashboardController::saveReEntryScrutinyData()` | DB updates + PDF generation + S3 upload + email. | Same — queue PDF, email, SMS. |
| `POST saveScrutinyDegreeAddData` | `Scrutiny/DashboardController::saveScrutinyDegreeAddData()` | DB updates + email + SMS. | Queue email and SMS. |

### Payment Processing

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST payuWebhook` | `PaymentController::payuWebhook()` | Massive write operation (BillHead, BillChild, Amttrans, OtherAppDtls, Download, Applicant update, SlotMaster update, email, PDF, S3) — all without a database transaction. Webhook and browser callback can fire simultaneously for the same transaction, causing duplicate records. | Wrap all DB writes in a single `DB::transaction()`. Add `lockForUpdate` on `PaymentTransactions` row to prevent duplicate processing. Move email/PDF/S3 after the transaction. |
| `POST remainingPaymentSync` | `PaymentController::remainingPaymentSync()` | Loops through potentially hundreds of transactions, each doing 5-10 DB writes + email + PDF + S3. No transaction wrapping. | Wrap each transaction's processing in `DB::transaction()`. Queue email/PDF generation. Add batch processing with chunking instead of loading all at once. |

---

## HIGH — Heavy Database Queries (200ms-2s per request)

### Public Homepage Data (No Auth Required)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET /api/front/get-dynamic-data` | `DynamicController::getAllDyamicDataFront()` | Executes 4x `LIKE '%..%'` queries scanning 109,000+ rows in `dbo_tblapplicationdtls`, plus 2 additional count queries, plus a full course list with eager loading. Public endpoint — every visitor triggers this. No caching. | Wrap the performance stats in `Cache::remember('performance_smart_board', 300, function() { ... })` with a 5-minute TTL. These numbers don't change per-user or per-minute. |

### User Dashboard

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET /api/user/dashboard` | `User/DashboardController::index()` | Executes 15-20+ queries per request: multiple Applicant lookups, OtherAppDtls queries across different application types, BillHead lookups, SlotDetails with eager loads, Download queries, AddressChangeApplicant queries. Every logged-in user hits this. 500 users = 7,500-10,000 queries. | Cache per-user dashboard data with a short TTL (60 seconds). Consolidate multiple OtherAppDtls queries into a single query with `whereIn` on AppNm. Use eager loading to reduce query count. |

### Admin Dashboard Statistics

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET /dashboard` | `Admin/DashboardController::dashboardStatistics()` | 15+ queries: 4 count queries across large tables + 4 GROUP BY queries with date formatting + 6 pending application counts. No caching. | Cache the statistics with a 5-minute TTL. Admin dashboards don't need real-time data. |

### Refresher Course Lists (N+1 Query Pattern)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `GET get-refresher-list` | `CollegeController::getRefresherList()` | N+1 pattern: fetches all published courses (1 query), then loops through each course executing a separate query to check the user's booking status. 10 courses × 500 users = 5,000+ queries. | Replace the loop with a single query: eager load the user's bookings using `->with(['slots' => fn($q) => $q->where('user_id', $userId)])` or use a left join subquery. |
| `GET get-offline-refresher-list` | `CollegeController::getOfflineRefresherList()` | Same N+1 pattern. | Same fix — eager load or join. |

---

## MEDIUM — Race Conditions and Missing Transactions

### Slot Cancellation (No Transaction)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST cancel-referesher-slot` | `CollegeController::cancelReferesherSlot()` | Reads `available_slot`/`booked_slot`, does arithmetic in PHP, then writes back — without any transaction or lock. Two concurrent cancellations corrupt the slot counter. | Wrap in `DB::beginTransaction()` + `lockForUpdate()` on the `RefresherCourse` row, matching the pattern already used in `bookRefresherCourseSlot()`. |
| `POST cancel-offline-referesher-slot` | `CollegeController::cancelOfflineReferesherSlot()` | Same race condition. | Same fix — add transaction + lock. |

### Appointment Booking (No Transaction)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST book-slot` | `AppointmentController::bookSlot()` | Read-modify-write on `booked_members`/`ava_slots` without transaction. Two concurrent bookings can oversell the slot. | Wrap in `DB::beginTransaction()` + `lockForUpdate()` on the `SlotMaster` row. Check `ava_slots > 0` after acquiring the lock. |

### Accounting Entries (No Transaction)

| API | Method | What It Does | How To Fix |
|-----|--------|-------------|------------|
| `POST storeBankentry` | `AccountController::storeBankentry()` | Creates paired debit/credit entries as separate INSERT statements without a transaction. Failure between inserts leaves an unbalanced ledger. | Wrap all 4 creates (2x Amttrans + 2x Bankentry) in a single `DB::transaction()`. |
| `POST storeCashentry` | `AccountController::storeCashentry()` | Same pattern — paired entries without transaction. | Same fix — wrap in `DB::transaction()`. |
| `POST storeJVentry` | `AccountController::storeJVentry()` | Same pattern. | Same fix. |

---

## LOW — Cron Jobs That Could Overlap with User Traffic

### Refresher Course Reminder Cron

| Cron | Method | What It Does | How To Fix |
|------|--------|-------------|------------|
| `refreshercoursecronmsg:cron` | `CollegeController::refresherCourseCron()` | Loops through all booked users for tomorrow's courses, sends individual emails via synchronous Mailgun curl. 500 booked users = 500 sequential blocking HTTP calls. | Queue each email as a separate job. Or use Mailgun's batch sending API to send all emails in a single API call. |

### Birthday Wishes Cron

| Cron | Method | What It Does | How To Fix |
|------|--------|-------------|------------|
| `birthdaywishescron:cron` | `MigrationController::birthdayCron()` | Queries all applicants with today's birthday using `whereRaw('MONTH(BOD) = ? AND DAY(BOD) = ?')` (full table scan on 109K rows), then sends email + SMS to each one sequentially. | Add an index on `BOD` column. Queue each email/SMS as a background job instead of sending synchronously in the loop. |

---

## Middleware — Runs on Every Authenticated Request

| Middleware | What It Does | How To Fix |
|-----------|-------------|------------|
| `JsonMiddleware` (jwt.verify) | On every authenticated API call: Query 1 (`users` table for JWT auth) → Query 2 (`dbo_tblrolemaster` for role) → Query 3 (`dbo_tblrolepermission` for permissions) → string comparison for URL-based access control. 500 users × 3 queries × multiple API calls per page = thousands of queries. | Cache the role + permissions per user in the JWT token claims or in application cache with a short TTL. The role/permissions rarely change but are queried on every single request. |

---

## Root Cause Summary

Almost every problematic API shares the same pattern: **synchronous external I/O inside the HTTP request**.

- `directMail()` — a blocking curl call to Mailgun's API (found in 40+ locations across the codebase)
- `uploadStorageFileS3()` / `uploadFileS3()` — blocking S3 upload
- `PDF::loadView()` — CPU-intensive PDF rendering (found in 30+ locations)
- `MessageController::SendMsg()` — blocking curl call to MSG91 SMS API

With `QUEUE_CONNECTION=sync`, even code designed to be asynchronous (`ProcessRefresherExam implements ShouldQueue`) runs synchronously within the HTTP request.

When 500 users are active simultaneously, the combination of:
1. `get-dynamic-data` full table scans (every page load, public)
2. N+1 refresher list queries (every user on the refresher page)
3. User dashboard's 15+ queries (every logged-in user)
4. PDF/email/S3 operations blocking workers for seconds

...is enough to occupy all 200 Apache workers, causing a complete outage.

---

## Recommended Fixes (Priority Order)

### P0 — Immediate (No Code Changes)

- [x] Increase `MaxRequestWorkers` to 200
- [x] Add `MaxConnectionsPerChild=1000`
- [x] Deploy watchdog cron for auto-recovery

### P1 — High Priority

1. **Change `QUEUE_CONNECTION` from `sync` to `database`** — requires creating a jobs table (`php artisan queue:table && php artisan migrate`) and running a queue worker via Supervisor. This alone offloads all PDF generation, S3 uploads, and email sending to background processing.

2. **Cache `getAllDyamicDataFront` performance stats** — wrap the 4 LIKE queries and count queries in `Cache::remember()` with a 5-minute TTL. Eliminates ~2,000 full table scans during traffic spikes.

3. **Move `directMail()` calls to queued jobs** — replace synchronous Mailgun curl calls with Laravel's built-in mail queue. Frees workers immediately after the database operation.

4. **Move email sending after `DB::commit()` in booking methods** — currently the `lockForUpdate` is held while sending email. Move the email call after the transaction commits.

### P2 — Medium Priority

5. Fix N+1 in `getRefresherList` / `getOfflineRefresherList` with eager loading.
6. Cache user dashboard data per-user with a short TTL.
7. Wrap slot cancellation methods in transactions with `lockForUpdate`.
8. Wrap appointment booking in a transaction with `lockForUpdate`.
9. Wrap payment webhook in a transaction to prevent duplicate records.
10. Wrap accounting entries in transactions to prevent unbalanced ledger.

### P3 — Long Term

11. Add Redis as cache and queue driver.
12. Cache smart card generation — check S3 before regenerating.
13. Add database index on `dbo_tblapplicationdtls.BOD` for birthday cron.
14. Cache middleware permission checks per user.
15. Consider switching from Apache prefork to Nginx + PHP-FPM for better concurrency.
