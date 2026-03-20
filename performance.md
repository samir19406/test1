# Performance Fix Plan — GSPC Backend

## Problem Statement

When ~500 users log in simultaneously, the backend becomes slow/unresponsive. The database is only 1.5GB, so the issue is not data volume — it is excessive query count, missing indexes, and inefficient configuration.

---

## Current State Summary

| Metric | Value |
|---|---|
| Total DB Size | ~1.5 GB |
| Largest Table | dbo_amttrans (920K rows, 133 MB) |
| Queries per Login | 4 (should be 1-2) |
| Queries per Dashboard Load | 20-25 (should be 5-6) |
| Queries per Middleware Check | 2-3 (runs on every authenticated request) |
| Cache Driver | file (slow under concurrency) |
| Session Driver | file (slow under concurrency) |
| Queue Driver | sync (blocks request until job finishes) |
| DB Connection Timeout | 1 second (too aggressive) |
| Tables Missing Indexes | Almost all except users, primary keys |

With 500 concurrent users: ~15,000-20,000 queries hit the database within seconds, most doing full table scans.

---

## Phase 1 — Database Indexes (Immediate, No Code Changes)

These can be run directly on MySQL. No deployment needed. Biggest bang for the buck.

| # | Table | Index To Add | Columns | Why |
|---|---|---|---|---|
| 1 | dbo_tblapplicationdtls | idx_app_userid | user_id | Dashboard queries this table 5+ times per user filtering by user_id. 105K rows, zero index on user_id = full table scan every time. This is the single biggest bottleneck. |
| 2 | dbo_tblapplicationdtls | idx_app_userid_type_status | user_id, applicationType, appStatus | Dashboard filters by all three columns together in multiple queries. Composite index avoids scanning 105K rows repeatedly. |
| 3 | dbo_tblotherappdtls | idx_other_entryno_appnm_accept | EntryNo, AppNm, AppAccept | Dashboard queries this table 6+ times per user with these exact filters. 211K rows with no covering index for this pattern. |
| 4 | dbo_tblotherappdtls | idx_other_appnm_entdt | AppNm, EntDt | Reports and dashboard statistics filter by AppNm + date range. Used in dashboardStatistics (public endpoint) and all report endpoints. |
| 5 | dbo_billhead | idx_bill_appnm_paydt | AppNm, PayDt | Every report endpoint filters by AppNm and PayDt. 249K rows, no index on these columns. Reports will timeout under load without this. |
| 6 | dbo_billhead | idx_bill_created_at | created_at | dashboardStatistics queries this table 4 times filtering by created_at. This endpoint is public (no auth) so anyone can trigger it. |
| 7 | dbo_tbldownloads | idx_downloads_srno_type | SrNo, type, sub_type | Dashboard loads provisional certificate URL by filtering on all three columns. 180K rows, only has primary key. |
| 8 | slot_details | idx_slot_userid_status | user_id, status | Dashboard queries appointment data by user_id + status. 62K rows, only has primary key index. |
| 9 | dbo_tblelgblappl | idx_eligible_userid | user_id | Dashboard queries eligible applicant by user_id. Small table (416 rows) but still benefits from index. |
| 10 | dbo_tbluploaddocs | idx_uploaddocs_srno | SrNo | 474K rows with no index beyond primary key. Queried when loading document lists for applications. |
| 11 | dbo_amttrans | idx_amttrans_refno | RefNo | 920K rows. Used in migration/sync endpoints and financial reports. Full table scan on largest table. |
| 12 | dbo_amttrans | idx_amttrans_currdate | CurrDate | Financial reports and migration scripts filter by date on this 920K row table. |
| 13 | dbo_billchild | idx_billchild_billsrno | BillSrNo | 728K rows. Joined with billhead in reports. No index means full scan on second largest table. |
| 14 | refresher_course_slot_booking | idx_refresher_slot_userid | user_id | Dashboard and user-facing refresher course endpoints filter by user_id. |

**Expected Impact:** 60-80% reduction in query execution time. Full table scans on 100K-900K row tables become index lookups returning in milliseconds.

---

## Phase 2 — Environment Configuration Changes (.env)

No code changes. Just update .env values and restart the application.

| # | Setting | Current Value | Recommended Value | Why |
|---|---|---|---|---|
| 1 | CACHE_DRIVER | file | redis | File-based cache uses filesystem locks. Under 500 concurrent requests, processes queue up waiting for file locks. Redis handles concurrent reads/writes natively. |
| 2 | SESSION_DRIVER | file | redis | Same problem as cache. Each request reads/writes a session file. 500 concurrent users = 500 file lock contentions. |
| 3 | QUEUE_CONNECTION | sync | redis | Sync means jobs (certificate generation, email sending) run inside the HTTP request. The user waits until the job finishes. Redis queue processes jobs in the background via a worker. |
| 4 | APP_DEBUG | true | false | Debug mode collects stack traces, query logs, and memory snapshots on every request. This adds CPU and memory overhead on every single request in production. Also exposes sensitive data in error responses. |
| 5 | LOG_LEVEL | debug | error | Debug logging writes to disk on every request. Under 500 concurrent users, the log file becomes a bottleneck (filesystem writes). Only log errors in production. |

**Expected Impact:** 20-30% improvement in request throughput. Redis eliminates filesystem contention. Disabling debug reduces per-request overhead.

**Prerequisite:** Redis must be installed and running on the server. The Redis config already exists in config/database.php.

---

## Phase 3 — Database Configuration (config/database.php)

One line change in config file.

| # | Setting | Current Value | Recommended Value | Why |
|---|---|---|---|---|
| 1 | PDO::ATTR_TIMEOUT | 1 (second) | 5 (seconds) | Under load, MySQL connection establishment can take >1 second. With a 1-second timeout, connections fail immediately during peak traffic, causing 500 errors. 5 seconds gives the connection pool breathing room. |

**Expected Impact:** Eliminates random 500 errors during peak load caused by connection timeouts.

---

## Phase 4 — Middleware Optimization (JsonMiddleware.php)

Affects every single authenticated API request.

| # | Change | File | Current Behavior | Recommended Behavior | Why |
|---|---|---|---|---|---|
| 1 | Cache user permissions | app/Http/Middleware/JsonMiddleware.php | Every request loads user → role → permissions from DB (2-3 queries) | Cache permissions in Redis for 5 minutes per user | With 500 users, every click/page load triggers 2-3 extra queries before the controller even runs. That's 1,000-1,500 unnecessary queries per page load across all users. Permissions rarely change — caching for 5 minutes is safe. |

**Expected Impact:** Eliminates 2-3 queries per request. At 500 concurrent users making 3-4 requests each, this removes ~3,000-6,000 queries from the database.

---

## Phase 5 — Login Optimization (UserController.php)

| # | Change | File | Current Behavior | Recommended Behavior | Why |
|---|---|---|---|---|---|
| 1 | Consolidate login queries | app/Http/Controllers/User/UserController.php (login method) | 4 separate queries: (1) check email exists, (2) check email verified, (3) JWTAuth::attempt, (4) load user with role/permissions | 2 queries: (1) JWTAuth::attempt, (2) load user with role/permissions + email verification check in same query | 500 logins × 4 queries = 2,000 queries. Should be 500 logins × 2 queries = 1,000 queries. The first two queries are redundant — JWTAuth::attempt already checks if the user exists. |

**Expected Impact:** 50% reduction in login query volume.

---

## Phase 6 — Dashboard Optimization (User/DashboardController.php)

This is the most impactful code change. Every user hits this endpoint after login.

| # | Change | Current Behavior | Recommended Behavior | Why |
|---|---|---|---|---|
| 1 | Fetch all user applicants in one query | Queries dbo_tblapplicationdtls 5 separate times for the same user_id with different filters | One query to fetch all applicant records for the user, then filter in PHP memory | 5 queries × 500 users = 2,500 queries. Should be 1 query × 500 users = 500 queries. The table is 105K rows — even with an index, 5 round-trips to MySQL add latency. |
| 2 | Fetch all OtherAppDtls in one query | Queries dbo_tblotherappdtls 6+ separate times for the same EntryNo with different AppNm/AppAccept filters | One query to fetch all other app records for the user's SrNo, then filter in PHP memory | 6 queries × 500 users = 3,000 queries. Should be 1 query × 500 users = 500 queries. Each query hits a 211K row table. |
| 3 | Reuse already-fetched applicant data | Queries Applicant again for fresh app data even though applicantDetailsObj already has the same data | Use the already-loaded applicantDetailsObj instead of querying again | Completely unnecessary duplicate query. Already have the data in memory. |
| 4 | Batch BillHead receipt lookups | Each application type triggers a separate BillHead query for receipt number | Fetch all BillHead records for the user's SrNo in one query, then match in PHP | Eliminates 2-4 queries per dashboard load depending on user's application types. |
| 5 | Remove redundant territory queries | 4 separate queries for transfer-territory status with different appStatus filters | Fetch all territory applicants once (already done at top of method), filter the collection in PHP | The territory list is already fetched at line 1. Lines at the bottom query the same table again with slightly different filters. |
| 6 | Avoid re-querying EligibleApplicant | Queries EligibleApplicant 2-3 times for the same user_id | Use the already-loaded eligibleDetailsObj from the top of the method | Already have this data. No need to query again. |

**Expected Impact:** Dashboard goes from 20-25 queries to 5-6 queries per user. At 500 users: from ~12,000 queries to ~3,000 queries.

---

## Phase 7 — Rate Limiting & Protection

| # | Change | File | Why |
|---|---|---|---|
| 1 | Add throttle middleware to login routes | routes/api.php | No rate limiting on login/signup/password-reset. A bot or brute-force attack can flood these endpoints with thousands of requests, consuming all DB connections and making the app unresponsive for real users. |
| 2 | Add auth middleware to dashboardStatistics | routes/api.php | This endpoint is currently public (no authentication). It runs 12+ aggregate queries including COUNT and GROUP BY on large tables. Anyone can call it repeatedly and overwhelm the database. |
| 3 | Remove or protect migration routes | routes/api.php | ~20 migration/sync endpoints are publicly accessible. They modify/delete database records and some do full table scans. If crawled or accidentally triggered, they can lock tables and slow everything down. |

**Expected Impact:** Prevents external abuse from consuming database resources that should be reserved for real users.

---

## Summary — Expected Total Impact

| Phase | Effort | Query Reduction | Throughput Improvement |
|---|---|---|---|
| Phase 1: DB Indexes | 30 minutes (run SQL) | Every query faster by 10-100x | 60-80% faster query execution |
| Phase 2: Redis + Config | 15 minutes (.env change) | Eliminates filesystem contention | 20-30% more requests/second |
| Phase 3: PDO Timeout | 1 minute (config change) | Prevents connection failures | Eliminates random 500 errors |
| Phase 4: Middleware Cache | 30 minutes (code change) | -3,000 to -6,000 queries at 500 users | Significant per-request speedup |
| Phase 5: Login Fix | 30 minutes (code change) | -1,000 queries at 500 logins | 50% faster login |
| Phase 6: Dashboard Fix | 2-3 hours (code change) | -9,000 queries at 500 users | 75% fewer dashboard queries |
| Phase 7: Rate Limiting | 30 minutes (route change) | Prevents abuse-driven load | Protects against external overload |

**Do Phase 1 + 2 first.** They require no code changes, can be done in under an hour, and will likely resolve the immediate problem for 500 concurrent users.

---

## Benchmarking & Testing Plan

Run these tests BEFORE and AFTER each phase to measure the actual improvement. Record the times in the results table at the bottom.

### Test 1 — Dashboard Statistics (Public, No Auth)

This is the heaviest public endpoint. Runs 12+ aggregate queries on billhead (249K rows), otherappdtls (211K rows), and refresher tables.

```bash
time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  "https://gujaratpharmacycouncil.co.in/api/dashboard?start_date=2025-01-01&end_date=2025-12-31&year=2025"
```

| When | Expected Time |
|---|---|
| Before any fix (baseline) | ~1.5s (confirmed) |
| After Phase 1 (indexes) | < 400ms |
| After Phase 2 (Redis) | < 300ms |

### Test 2 — Migration Duplicate Scan (Public, No Auth)

Runs GROUP BY + HAVING on amttrans (920K rows) and billchild (728K rows). Full table scans.

```bash
time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  "https://gujaratpharmacycouncil.co.in/api/amt-duplicate"

time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  "https://gujaratpharmacycouncil.co.in/api/billchild-duplicate"
```

| When | Expected Time |
|---|---|
| Before any fix (baseline) | Record this |
| After Phase 1 (indexes) | 50-70% faster |

### Test 3 — User Login (Public, No Auth)

Tests the login flow which currently runs 4 queries. Use a test account.

```bash
time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  -X POST "https://gujaratpharmacycouncil.co.in/api/generic/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"[test_user_email]","password":"[base64_encoded_password]"}'
```

| When | Expected Time |
|---|---|
| Before any fix (baseline) | Record this |
| After Phase 1 (indexes) | Slightly faster (users table already indexed) |
| After Phase 5 (login fix) | 40-50% faster |

### Test 4 — User Dashboard (Auth Required)

This is the most critical test. Every user hits this after login. Runs 20-25 queries. Requires a valid JWT token.

```bash
# Step 1: Get token
TOKEN=$(curl -s -X POST "https://gujaratpharmacycouncil.co.in/api/generic/login" \
  -H "Content-Type: application/json" \
  -d '{"email":"[test_user_email]","password":"[base64_encoded_password]"}' \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['data']['token'])")

# Step 2: Hit dashboard
time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  -H "Authorization: Bearer $TOKEN" \
  "https://gujaratpharmacycouncil.co.in/api/user/dashboard"
```

| When | Expected Time |
|---|---|
| Before any fix (baseline) | Record this |
| After Phase 1 (indexes) | 50-60% faster |
| After Phase 4 (middleware cache) | Additional 20-30% faster |
| After Phase 6 (dashboard fix) | < 300ms |

### Test 5 — Reports Endpoint (Auth Required)

Tests report generation which joins multiple large tables.

```bash
time curl -s -o /dev/null -w "HTTP %{http_code} | Time: %{time_total}s\n" \
  -X POST -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"app_name":"fresh-registration","start_date":"2025-01-01","end_date":"2025-12-31"}' \
  "https://gujaratpharmacycouncil.co.in/api/bo/reports/applicationRegister"
```

| When | Expected Time |
|---|---|
| Before any fix (baseline) | Record this |
| After Phase 1 (indexes) | 50-70% faster |

### Test 6 — Concurrent Load Test

This simulates the actual problem — 500 users hitting the system at once. Run this from a machine with good network to the server.

Requires `ab` (Apache Bench) or `hey` tool.

```bash
# Install hey (if not available)
# brew install hey (macOS) or go install github.com/rakyll/hey@latest

# Test 1: 100 concurrent requests to public dashboard
hey -n 500 -c 100 \
  "https://gujaratpharmacycouncil.co.in/api/dashboard?start_date=2025-01-01&end_date=2025-12-31&year=2025"

# Test 2: 50 concurrent requests to user dashboard (with token)
hey -n 200 -c 50 \
  -H "Authorization: Bearer $TOKEN" \
  "https://gujaratpharmacycouncil.co.in/api/user/dashboard"
```

Key metrics to record from `hey` output:

| Metric | Before Fix | After Phase 1 | After Phase 2 | After All Phases |
|---|---|---|---|---|
| Average response time | | | | |
| P95 response time | | | | |
| Requests/sec | | | | |
| Slowest request | | | | |
| Error count (non-200) | | | | |

### Test 7 — MySQL Connection Check

Run this on the database during peak hours to check if you're hitting connection limits.

```sql
SHOW VARIABLES LIKE 'max_connections';
SHOW STATUS LIKE 'Threads_connected';
SHOW STATUS LIKE 'Max_used_connections';
SHOW STATUS LIKE 'Aborted_connects';
SHOW STATUS LIKE 'Threads_running';
```

| Metric | Healthy Value | Problem Value |
|---|---|---|
| Threads_connected | < 80% of max_connections | > 90% of max_connections |
| Max_used_connections | < 80% of max_connections | Close to or equal to max_connections |
| Aborted_connects | Low / stable | Increasing rapidly |
| Threads_running | < 30 | > 50 (queries piling up) |

### Test 8 — Slow Query Check

Enable slow query logging before testing, then review after.

```sql
-- Enable before testing
SET GLOBAL slow_query_log = 'ON';
SET GLOBAL long_query_time = 0.5;

-- After testing, check slowest queries
SELECT * FROM mysql.slow_log ORDER BY query_time DESC LIMIT 20;

-- Disable after testing
SET GLOBAL slow_query_log = 'OFF';
```

---

## Results Tracking Table

Fill this in as you apply each phase. Run all tests before starting, then after each phase.

| Test | Baseline | After Phase 1 | After Phase 2 | After Phase 3 | After Phase 4 | After Phase 5 | After Phase 6 | After Phase 7 |
|---|---|---|---|---|---|---|---|---|
| Test 1: Dashboard Stats | 1.49s | | | | | | | |
| Test 2: amt-duplicate | | | | | | | | |
| Test 2: billchild-duplicate | | | | | | | | |
| Test 3: Login | | | | | | | | |
| Test 4: User Dashboard | | | | | | | | |
| Test 5: Reports | | | | | | | | |
| Test 6: Concurrent avg | | | | | | | | |
| Test 6: Concurrent P95 | | | | | | | | |
| Test 6: Requests/sec | | | | | | | | |
| Test 6: Errors | | | | | | | | |
| Test 7: Threads_connected | | | | | | | | |
| Test 7: Max_used_connections | | | | | | | | |


---

## Complete API Endpoint Inventory

All endpoints from `routes/api.php` with authentication requirement and performance weight assessment.

**Performance Weight Legend:**
- 🟢 Light — 1-2 queries, small tables, fast response
- 🟡 Medium — 3-6 queries, or hits medium tables (10K-100K rows)
- 🔴 Heavy — 7+ queries, or full scans on large tables (100K+ rows), or PDF/Excel generation, or external API calls
- ⚫ Critical — 10+ queries, or GROUP BY on 200K+ rows, or modifies/deletes data on large tables without auth

---

### Public Routes (No Authentication)

These endpoints have NO auth middleware. Anyone on the internet can call them.

| # | Method | Endpoint | Performance Weight | Notes |
|---|---|---|---|---|
| | | **Migration / Sync (PUBLIC, DANGEROUS)** | | |
| 1 | GET | /api/migrate-doc-parent | ⚫ Critical | Full scan on upload docs (474K rows). Modifies data. No auth. |
| 2 | GET | /api/migrate-billhead | ⚫ Critical | Full scan + migration on billhead (249K rows). Modifies data. No auth. |
| 3 | GET | /api/migrate-doc | ⚫ Critical | Full scan on upload docs. Modifies data. No auth. |
| 4 | GET | /api/migrate-app | ⚫ Critical | Full scan on applicationdtls (105K rows). Modifies data. No auth. |
| 5 | GET | /api/migrate-download | ⚫ Critical | GROUP BY + DELETE on downloads (180K rows). No auth. |
| 6 | GET | /api/amt-duplicate | ⚫ Critical | GROUP BY + HAVING on amttrans (920K rows, 133MB). Full table scan. No auth. |
| 7 | GET | /api/billchild-duplicate | ⚫ Critical | GROUP BY + HAVING on billchild (728K rows). Full table scan. No auth. |
| 8 | GET | /api/amt-single | ⚫ Critical | Single account migration on amttrans (920K rows). Modifies data. No auth. |
| 9 | GET | /api/date-change | ⚫ Critical | Date migration on large tables. Modifies data. No auth. |
| 10 | GET | /api/amt-trans | ⚫ Critical | Full account migration on amttrans (920K rows). Modifies data. No auth. |
| 11 | GET | /api/amt-final | ⚫ Critical | Final amttrans processing (920K rows). Modifies data. No auth. |
| 12 | GET | /api/amt-final-check-entry | ⚫ Critical | Check entries on amttrans (920K rows). No auth. |
| 13 | GET | /api/amt-final-check-entry-total-check | ⚫ Critical | Total check on amttrans (920K rows). No auth. |
| 14 | GET | /api/remaining-bill-child | ⚫ Critical | Scan billchild (728K rows). No auth. |
| 15 | GET | /api/remaining-bill-child-entry-save | ⚫ Critical | Modifies billchild (728K rows). No auth. |
| 16 | GET | /api/bill-child-entry-date-change | ⚫ Critical | Modifies billchild dates. No auth. |
| 17 | GET | /api/migrate-clg-name | ⚫ Critical | Full scan + update on applicationdtls (105K rows). No auth. |
| 18 | GET | /api/migrate-deg-clg-name | ⚫ Critical | Full scan + update on degree tables. No auth. |
| 19 | GET | /api/remaining-payment-sync | ⚫ Critical | Payment sync, hits billhead (249K rows) + external PayU API. No auth. |
| 20 | GET | /api/order-date-sync | ⚫ Critical | Date sync on billhead (249K rows). Modifies data. No auth. |
| 21 | GET | /api/undertaking-complain | ⚫ Critical | Bulk data processing + WhatsApp API calls. No auth. |
| 22 | GET | /api/undertaking-complain-2 | ⚫ Critical | Same as above, different batch. No auth. |
| 23 | GET | /api/undertaking-complain-3 | ⚫ Critical | Same as above, different batch. No auth. |
| 24 | GET | /api/undertaking-complain-4 | ⚫ Critical | Same as above, different batch. No auth. |
| 25 | GET | /api/pci-code-migration | ⚫ Critical | PCI code migration on applicationdtls. Modifies data. No auth. |
| 26 | GET | /api/store-renewal-data | ⚫ Critical | Stores renewal data + sends WhatsApp messages. No auth. |
| 27 | GET | /api/get-renewal-data | 🔴 Heavy | Reads renewal data. No auth. |
| 28 | GET | /api/add-emailto-renewal-data | ⚫ Critical | Modifies renewal records. No auth. |
| 29 | GET | /api/refresher-slot-update-migration | ⚫ Critical | Updates refresher slot data. Modifies data. No auth. |
| 30 | GET | /api/offline-refresher-slot-update-migration | ⚫ Critical | Updates offline refresher slots. Modifies data. No auth. |
| 31 | POST | /api/profile-pic-upload-renewal-migration | ⚫ Critical | Uploads profile pics in bulk. S3 + DB writes. No auth. |
| 32 | GET | /api/profile-pic-get-file-details | 🔴 Heavy | Reads file details for migration. No auth. |
| 33 | GET | /api/profile-pic-sent-migration | ⚫ Critical | Sends profile pic migration messages. No auth. |
| 34 | GET | /api/renewal-upload-doc-reminder | ⚫ Critical | Sends bulk WhatsApp reminders. No auth. |
| | | **Authentication (PUBLIC)** | | |
| 35 | POST | /api/signup | 🟡 Medium | 2-3 queries (check exists + insert). No rate limit. |
| 36 | POST | /api/login | 🟡 Medium | 2-3 queries. Legacy auth endpoint. No rate limit. |
| 37 | GET | /api/sendForgotPassword/{email} | 🟡 Medium | 1 query + email send (Mailgun API). No rate limit. |
| 38 | POST | /api/validateForgotPassword | 🟢 Light | 1-2 queries. Token validation. |
| 39 | POST | /api/resetForgotPassword | 🟢 Light | 1-2 queries. Password update. |
| 40 | POST | /api/user/register | 🟡 Medium | 3-4 queries (validate + insert user + role). No rate limit. |
| 41 | POST | /api/user/register-with-gno | 🟡 Medium | 3-4 queries (validate GNO + insert). No rate limit. |
| 42 | POST | /api/generic/login | 🔴 Heavy | 4 queries (email check + verified check + attempt + load user/role/permissions). No rate limit. |
| 43 | POST | /api/admin/login | 🔴 Heavy | 3-4 queries + OTP generation + WhatsApp API call. Returns JWT before OTP verify. No rate limit. |
| 44 | POST | /api/admin/verifyotp | 🟡 Medium | 2-3 queries. OTP verification. |
| 45 | GET | /api/emailverified/{email} | 🟢 Light | 1 query + update. Email verification link. |
| 46 | POST | /api/sendForgotPasswordEmail | 🟡 Medium | 1 query + Mailgun API call. No rate limit. |
| 47 | POST | /api/resetpassword | 🟢 Light | 1-2 queries. Password reset. |
| 48 | POST | /api/sendAppMail | 🟡 Medium | 1 query + Mailgun API call. No rate limit. |
| | | **Payment Callbacks (PUBLIC)** | | |
| 49 | POST | /api/payment-callback | 🔴 Heavy | Payment verification + 4-6 queries (billhead, amttrans, billchild inserts). External PayU API verify. |
| 50 | POST | /api/payment-callback-mobile | 🔴 Heavy | Same as above, mobile variant. |
| 51 | POST | /api/payment-webhook | 🔴 Heavy | Webhook processing + DB writes. |
| | | **QR Scan (PUBLIC)** | | |
| 52 | POST | /api/scan-qr | 🟡 Medium | 2-3 queries. Looks up applicant by QR data. |
| 53 | POST | /api/scan-qr-other | 🟡 Medium | 2-3 queries. Looks up other app by QR data. |
| | | **Dashboard Statistics (PUBLIC)** | | |
| 54 | GET | /api/dashboard | ⚫ Critical | 12+ queries: COUNT on users, 4× GROUP BY on billhead (249K rows), 3× COUNT on otherappdtls (211K rows), 6× COUNT pending apps on otherappdtls, 1 refresher query. All with date range filters on unindexed columns. Confirmed 1.49s baseline. |
| | | **Export (PUBLIC)** | | |
| 55 | GET | /api/dglocker-csv | 🔴 Heavy | CSV export. Scans applicationdtls (105K rows) + joins. No auth. |
| | | **College Auth (PUBLIC)** | | |
| 56 | POST | /api/college-register | 🟡 Medium | 3-4 queries + email verification. |
| 57 | POST | /api/verify-college-email | 🟢 Light | 1-2 queries. Token check + update. |
| 58 | POST | /api/college-login | 🟡 Medium | 2-3 queries + OTP via WhatsApp API. |
| 59 | POST | /api/college-login-otp-verify | 🟡 Medium | 2-3 queries. OTP check + JWT issue. |
| 60 | GET | /api/college-list | 🟢 Light | 1 query. List colleges. |
| | | **PayU Hash (PUBLIC)** | | |
| 61 | GET | /api/generate-payu-hashstring-mobile | 🟢 Light | Hash generation, no DB query. But exposes PayU merchant key logic publicly. |
| | | **Front Website (PUBLIC)** | | |
| 62 | GET | /api/front/get-dynamic-data | 🟡 Medium | 3-4 queries (banners + notices + updates + gallery). |
| 63 | POST | /api/front/save-contact-us | 🟢 Light | 1 insert query. |
| | | **Job Portal (PUBLIC)** | | |
| 64 | GET | /api/job/arealist | 🟢 Light | 1 query. Area list. |
| 65 | GET | /api/job/jobtype | 🟢 Light | 1 query. Job types. |
| 66 | GET | /api/job/departmentlist | 🟢 Light | 1 query. Departments. |
| 67 | POST | /api/job/adddedepartment | 🟢 Light | 1 insert. No auth — anyone can add departments. |
| 68 | POST | /api/job/editdepartment | 🟢 Light | 1 update. No auth — anyone can edit departments. |
| 69 | GET | /api/job/companyjoblist | 🟡 Medium | 2-3 queries with relations. |
| 70 | POST | /api/job/addjobbycompany | 🟡 Medium | 2-3 queries. No auth — anyone can post jobs. |
| 71 | POST | /api/job/updatestatusjob | 🟢 Light | 1 update. No auth. |
| 72 | DELETE | /api/job/deleteJob/{id} | 🟢 Light | 1 delete. No auth — anyone can delete jobs. |
| 73 | POST | /api/job/download-job-list | 🔴 Heavy | PDF generation + joins. No auth. |
| 74 | POST | /api/job/export-job-list | 🔴 Heavy | Excel export + joins. No auth. |
| 75 | GET | /api/job/studentlist | 🟡 Medium | 2-3 queries. No auth. |
| 76 | POST | /api/job/addstudent | 🟢 Light | 1 insert. No auth. |
| 77 | POST | /api/job/updatestudent | 🟢 Light | 1 update. No auth. |
| 78 | POST | /api/job/updatestudentstatus | 🟢 Light | 1 update. No auth. |
| 79 | POST | /api/job/viewstudent | 🟢 Light | 1 query. No auth. |
| 80 | DELETE | /api/job/deleteStudent/{id} | 🟢 Light | 1 delete. No auth. |
| | | **AI Routes (PUBLIC)** | | |
| 81 | GET | /api/ai/get-published-upcoming-refresher-courses | 🟢 Light | 1-2 queries. Refresher course list. |
| 82 | POST | /api/ai/get-user-details-refresher-course | 🟡 Medium | 3-4 queries. User + refresher data lookup. |
| 83 | POST | /api/ai/get-user-fresh-application-check | 🟡 Medium | 2-3 queries. Application status check. |
| 84 | POST | /api/ai/email-phone-update | 🟡 Medium | 2-3 queries. Updates user contact info. No auth. |
| 85 | POST | /api/ai/faq-enquiry | 🟢 Light | 1-2 queries. FAQ lookup. |
| 86 | POST | /api/ai/get-user-other-application-check | 🟡 Medium | 2-3 queries. Other app status check. |
| | | **Misc Public** | | |
| 87 | GET | /api/get-user-data/{id} | 🟡 Medium | 2-3 queries. Returns user data by ID. No auth. |
| 88 | GET | /api/test-mail | 🟢 Light | Test email send. Mailgun API. No auth. |
| 89 | POST | /api/change-password | 🟢 Light | 1-2 queries. Password change without auth — security risk. |
| 90 | GET | /api/re-entry-certy-new | 🔴 Heavy | Re-entry certificate generation. PDF + S3. No auth. |
| 91 | GET | /api/refresher-exist-status | 🟡 Medium | 2-3 queries. Refresher status check. No auth. |
| 92 | GET | /api/refresher-certy-change-status | 🔴 Heavy | Certificate generation + S3 upload. Modifies data. No auth. |


---

### Protected Routes (JWT Authentication Required)

These endpoints require a valid JWT token via `jwt.verify` middleware. Each request also triggers 2-3 middleware queries for permission check (see Phase 4).

| # | Method | Endpoint | Performance Weight | Notes |
|---|---|---|---|---|
| | | **User Routes (`/api/user/`)** | | |
| 1 | GET | /api/user/dashboard | ⚫ Critical | 20-25 queries per load. Hits applicationdtls (105K) 5×, otherappdtls (211K) 6×, billhead (249K), downloads (180K), slot_details (62K), eligible table. Single biggest bottleneck for 500 users. |
| 2 | GET | /api/user/validity-certy-download | 🔴 Heavy | PDF generation + S3 upload + 3-4 queries. |
| 3 | GET | /api/user/refresher-booking-details | 🟡 Medium | 2-3 queries. Refresher slot lookup by user. |
| 4 | GET | /api/user/offline-refresher-booking-details | 🟡 Medium | 2-3 queries. Offline refresher slot lookup. |
| 5 | GET | /api/user/collages | 🟢 Light | 1 query. College list. |
| 6 | GET | /api/user/hospitals | 🟢 Light | 1 query. Hospital list. |
| 7 | GET | /api/user/states | 🟢 Light | 1 query. State list. |
| 8 | POST | /api/user/create-application | 🔴 Heavy | 5-8 queries (validate + insert app + docs + billhead + payment setup). S3 uploads. |
| 9 | POST | /api/user/create-territory-application | 🔴 Heavy | 5-8 queries. Similar to create-application for territory transfer. |
| 10 | POST | /api/user/unverified-to-pending | 🟡 Medium | 2-3 queries. Status update. |
| 11 | GET | /api/user/get-initial-application-data | 🟡 Medium | 3-5 queries. Loads user + applicant + related data. |
| 12 | GET | /api/user/get-document-list | 🟡 Medium | 2-3 queries. Document list for application. |
| 13 | GET | /api/user/other-document-list | 🟡 Medium | 2-3 queries. Other document list. |
| 14 | POST | /api/user/generate-payu-hashstring | 🟡 Medium | 2-3 queries + hash generation. |
| 15 | POST | /api/user/check-tnx-status | 🟡 Medium | 2-3 queries + external PayU API call. |
| 16 | POST | /api/user/save-document | 🟡 Medium | 2-3 queries + S3 upload. |
| 17 | POST | /api/user/save-document-renew-exist | 🟡 Medium | 2-3 queries + S3 upload. |
| 18 | POST | /api/user/save-document-other | 🟡 Medium | 2-3 queries + S3 upload. |
| 19 | POST | /api/user/change-app-doc-status | 🟢 Light | 1-2 queries. Status update. |
| 20 | GET | /api/user/get-available-slots | 🟡 Medium | 2-3 queries. Slot availability check on slot_details (62K rows). |
| 21 | POST | /api/user/book-slot | 🟡 Medium | 3-4 queries. Slot booking + update counts. |
| 22 | GET | /api/user/downloads | 🟡 Medium | 2-3 queries on downloads (180K rows). No index on user filter columns. |
| 23 | GET | /api/user/single-download | 🔴 Heavy | Receipt PDF generation. Queries billhead + billchild + amttrans. |
| 24 | GET | /api/user/single-download-new-fees | 🔴 Heavy | New fees receipt PDF. Same large table joins. |
| 25 | GET | /api/user/single-download-renew-receipt | 🔴 Heavy | Renewal receipt PDF. Joins on billhead (249K) + billchild (728K). |
| 26 | GET | /api/user/single-download-certificates | 🔴 Heavy | Certificate PDF generation + S3. |
| 27 | POST | /api/user/create-eligible-application | 🟡 Medium | 3-4 queries. Eligible app creation. |
| 28 | POST | /api/user/create-address-change-application | 🟡 Medium | 3-4 queries. Address change app. |
| 29 | GET | /api/user/councils | 🟢 Light | 1 query. Council list. |
| 30 | GET | /api/user/download-smart-card/{srNo} | 🔴 Heavy | PDF generation + 3-4 queries + S3. |
| 31 | GET | /api/user/app-billing-amount | 🟡 Medium | 2-3 queries. Billing amount lookup. |
| 32 | POST | /api/user/pro-to-original | 🟡 Medium | 2-3 queries. Provisional to original request. |
| 33 | POST | /api/user/change-password | 🟢 Light | 1-2 queries. Password update. |
| 34 | GET | /api/user/check-appointment-status | 🟡 Medium | 2-3 queries on slot_details (62K rows). |
| 35 | GET | /api/user/get-slot-details | 🟡 Medium | 2-3 queries. Slot details. |
| 36 | POST | /api/user/book-refresher-slot | 🔴 Heavy | 4-6 queries. Slot booking + validation + update counts + WhatsApp notification. |
| 37 | POST | /api/user/book-offline-refresher-slot | 🔴 Heavy | 4-6 queries. Same as above for offline. |
| 38 | POST | /api/user/refresher-attendance | 🟡 Medium | 2-3 queries. Attendance marking. |
| 39 | GET | /api/user/get-refresher-quiz-data | 🟡 Medium | 2-3 queries. Quiz data load. |
| 40 | POST | /api/user/save-exam | 🟡 Medium | 3-4 queries. Save exam answers + calculate score. |
| 41 | POST | /api/user/submit-feedback | 🟢 Light | 1-2 queries. Feedback insert. |
| 42 | GET | /api/user/get-attended-refresher-list | 🟡 Medium | 2-3 queries. Attended list. |
| 43 | GET | /api/user/get-attended-offline-refresher-list | 🟡 Medium | 2-3 queries. Offline attended list. |
| 44 | GET | /api/user/get-refresher-list | 🟡 Medium | 2-3 queries. Available refresher list. |
| 45 | GET | /api/user/get-offline-refresher-list | 🟡 Medium | 2-3 queries. Available offline list. |
| 46 | POST | /api/user/cancel-referesher-slot | 🟡 Medium | 2-3 queries. Cancel + update slot count. |
| 47 | POST | /api/user/cancel-offline-referesher-slot | 🟡 Medium | 2-3 queries. Cancel offline slot. |
| 48 | POST | /api/user/profile-pic-upload | 🟡 Medium | 1-2 queries + S3 upload. |
| | | **Admin Dashboard (`/api/dashboard/`)** | | |
| 49 | GET | /api/dashboard/get-file-details | 🔴 Heavy | 6-10 queries. Loads full applicant file with docs, payments, history. Hits applicationdtls + otherappdtls + billhead + uploaddocs. |
| 50 | GET | /api/dashboard/get-user-details-by-regno | 🔴 Heavy | 4-6 queries. Applicant lookup by reg number across applicationdtls (105K) + otherappdtls (211K). |
| 51 | GET | /api/dashboard/search | 🔴 Heavy | Search across applicationdtls (105K) + otherappdtls (211K). Potentially full table scan if search term is broad. |
| | | **Master Data (`/api/bo/master/`)** | | |
| 52 | POST | /api/bo/master/applications | 🟢 Light | 1 insert. Create application type. |
| 53 | POST | /api/bo/master/update-applications | 🟢 Light | 1 update. |
| 54 | GET | /api/bo/master/applications | 🟢 Light | 1 query. List application types. |
| 55 | DELETE | /api/bo/master/applications | 🟢 Light | 1 delete. |
| 56 | POST | /api/bo/master/documents | 🟢 Light | 1 insert. |
| 57 | GET | /api/bo/master/documents | 🟢 Light | 1 query. |
| 58 | POST | /api/bo/master/update-documents | 🟢 Light | 1 update. |
| 59 | DELETE | /api/bo/master/documents | 🟢 Light | 1 delete. |
| 60 | POST | /api/bo/master/map-documents | 🟢 Light | 1-2 queries. Map app to doc. |
| 61 | POST | /api/bo/master/edit-map-documents | 🟢 Light | 1-2 queries. |
| 62 | GET | /api/bo/master/documents-master | 🟢 Light | 1 query. |
| 63 | POST | /api/bo/master/hospitals | 🟢 Light | 1 insert. |
| 64 | POST | /api/bo/master/update-hospitals | 🟢 Light | 1 update. |
| 65 | GET | /api/bo/master/hospitals | 🟢 Light | 1 query. |
| 66 | DELETE | /api/bo/master/hospitals | 🟢 Light | 1 delete. |
| 67 | POST | /api/bo/master/colleges | 🟢 Light | 1 insert. |
| 68 | POST | /api/bo/master/update-colleges | 🟢 Light | 1 update. |
| 69 | GET | /api/bo/master/colleges | 🟢 Light | 1 query. |
| 70 | GET | /api/bo/master/single-colleges | 🟢 Light | 1 query. |
| 71 | DELETE | /api/bo/master/colleges | 🟢 Light | 1 delete. |
| 72 | POST | /api/bo/master/college-approval | 🟢 Light | 1-2 queries. |
| 73 | GET | /api/bo/master/college-approval | 🟢 Light | 1 query. |
| 74 | GET | /api/bo/master/single-college-approval | 🟢 Light | 1 query. |
| 75 | POST | /api/bo/master/single-college-approval-delete | 🟢 Light | 1 delete. |
| 76 | POST | /api/bo/master/boards | 🟢 Light | 1 insert. |
| 77 | POST | /api/bo/master/update-boards | 🟢 Light | 1 update. |
| 78 | GET | /api/bo/master/boards | 🟢 Light | 1 query. |
| 79 | DELETE | /api/bo/master/boards | 🟢 Light | 1 delete. |
| 80 | POST | /api/bo/master/councils | 🟢 Light | 1 insert. |
| 81 | GET | /api/bo/master/councils | 🟢 Light | 1 query. |
| 82 | POST | /api/bo/master/update-councils | 🟢 Light | 1 update. |
| 83 | DELETE | /api/bo/master/councils | 🟢 Light | 1 delete. |
| 84 | POST | /api/bo/master/university | 🟢 Light | 1 insert. |
| 85 | POST | /api/bo/master/update-university | 🟢 Light | 1 update. |
| 86 | GET | /api/bo/master/university | 🟢 Light | 1 query. |
| 87 | DELETE | /api/bo/master/university | 🟢 Light | 1 delete. |
| 88 | POST | /api/bo/master/exam-authority | 🟢 Light | 1 insert. |
| 89 | POST | /api/bo/master/update-exam-authority | 🟢 Light | 1 update. |
| 90 | GET | /api/bo/master/exam-authority | 🟢 Light | 1 query. |
| 91 | DELETE | /api/bo/master/exam-authority | 🟢 Light | 1 delete. |
| 92 | POST | /api/bo/master/banks | 🟢 Light | 1 insert. |
| 93 | POST | /api/bo/master/update-banks | 🟢 Light | 1 update. |
| 94 | GET | /api/bo/master/banks | 🟢 Light | 1 query. |
| 95 | DELETE | /api/bo/master/banks | 🟢 Light | 1 delete. |
| 96 | POST | /api/bo/master/bill-item | 🟢 Light | 1 insert. |
| 97 | POST | /api/bo/master/update-bill-item | 🟢 Light | 1 update. |
| 98 | GET | /api/bo/master/bill-item | 🟢 Light | 1 query. |
| 99 | DELETE | /api/bo/master/bill-item | 🟢 Light | 1 delete. |
| 100 | POST | /api/bo/master/bill-master | 🟢 Light | 1 insert. |
| 101 | GET | /api/bo/master/bill-master | 🟢 Light | 1 query. |
| 102 | DELETE | /api/bo/master/bill-master | 🟢 Light | 1 delete. |
| 103 | GET | /api/bo/master/users-list | 🔴 Heavy | Loads all users (79K rows) with roles. Potentially full table scan if no pagination. |
| 104 | POST | /api/bo/master/linkwith-gno | 🟡 Medium | 2-3 queries. Link user with GNO. |
| 105 | POST | /api/bo/master/download-smart-card | 🔴 Heavy | PDF generation + 3-4 queries + S3. |
| 106 | POST | /api/bo/master/user-change-email | 🟡 Medium | 2-3 queries. Admin email change. |
| 107 | POST | /api/bo/master/user-change-password | 🟢 Light | 1-2 queries. |
| 108 | POST | /api/bo/master/gno-wise-details | 🟡 Medium | 2-3 queries. GNO lookup. |
| 109 | POST | /api/bo/master/profile-pic-upload | 🟡 Medium | 1-2 queries + S3 upload. |
| 110 | GET | /api/bo/master/states | 🟢 Light | 1 query. |

| | | **Reports (`/api/bo/reports/`)** | | |
| 111 | GET | /api/bo/reports/get-globalval | 🟢 Light | 1 query. Global value lookup. |
| 112 | POST | /api/bo/reports/pci-register-csv | 🔴 Heavy | CSV export. Joins applicationdtls (105K) + otherappdtls (211K) + billhead (249K). |
| 113 | GET | /api/bo/reports/record-register-csv | 🔴 Heavy | CSV export. Full scan on applicationdtls (105K) with joins. |
| 114 | POST | /api/bo/reports/applicationRegister | 🔴 Heavy | 3-5 queries. Joins billhead (249K) + applicationdtls (105K) with date filters on unindexed columns. |
| 115 | POST | /api/bo/reports/download-application-register | ⚫ Critical | Same as above + PDF generation. Loads full result set into memory for PDF. |
| 116 | POST | /api/bo/reports/feeRegister | 🔴 Heavy | 3-5 queries. Joins billhead (249K) + billchild (728K) with date filters. |
| 117 | POST | /api/bo/reports/download-fee-register | ⚫ Critical | Same as above + PDF generation. Joins two of the largest tables. |
| 118 | POST | /api/bo/reports/freshApplicationRecord | 🔴 Heavy | 3-5 queries on applicationdtls (105K) with date + status filters. |
| 119 | POST | /api/bo/reports/download-fresh-application-record | ⚫ Critical | Same + PDF generation. |
| 120 | POST | /api/bo/reports/pharmacistDetails | 🔴 Heavy | 3-5 queries. Pharmacist data across applicationdtls + otherappdtls. |
| 121 | POST | /api/bo/reports/degreeDetails | 🔴 Heavy | 3-5 queries. Degree data joins. |
| 122 | POST | /api/bo/reports/download-degree-details | ⚫ Critical | Same + PDF generation. |
| 123 | POST | /api/bo/reports/registrationValidation | 🔴 Heavy | 3-5 queries on applicationdtls with date filters. |
| 124 | POST | /api/bo/reports/download-registration-validation | ⚫ Critical | Same + PDF generation. |
| 125 | POST | /api/bo/reports/statisticsValidation | ⚫ Critical | 8-10 queries. Multiple COUNT + GROUP BY on applicationdtls (105K) + otherappdtls (211K) + billhead (249K). |
| 126 | POST | /api/bo/reports/registrationCancellation | 🔴 Heavy | 3-5 queries on otherappdtls (211K). |
| 127 | POST | /api/bo/reports/download-registration-cancellation | ⚫ Critical | Same + PDF generation. |
| 128 | POST | /api/bo/reports/applicationDetails | 🔴 Heavy | 3-5 queries. Application detail joins. |
| 129 | POST | /api/bo/reports/download-application-details | ⚫ Critical | Same + PDF generation. |
| 130 | POST | /api/bo/reports/inOut-Register | 🟡 Medium | 2-3 queries. In/out register data. |
| 131 | POST | /api/bo/reports/download-inout-register | 🔴 Heavy | Same + PDF generation. |
| 132 | POST | /api/bo/reports/provisional-original-certificate | 🔴 Heavy | 3-5 queries on downloads (180K) + applicationdtls. |
| 133 | POST | /api/bo/reports/download-provisional-original-certificate | ⚫ Critical | Same + PDF generation. |
| 134 | POST | /api/bo/reports/collegeWiseReg | 🔴 Heavy | 3-5 queries. GROUP BY college on applicationdtls (105K). |
| 135 | POST | /api/bo/reports/download-collegewise-registration | ⚫ Critical | Same + PDF generation. |
| 136 | POST | /api/bo/reports/govReport | 🔴 Heavy | 3-5 queries. Government report aggregation. |
| 137 | POST | /api/bo/reports/download-government | ⚫ Critical | Same + PDF generation. |
| 138 | POST | /api/bo/reports/ddReport | 🔴 Heavy | 3-5 queries. DD report. |
| 139 | POST | /api/bo/reports/download-dd-report | ⚫ Critical | Same + PDF generation. |
| 140 | GET | /api/bo/reports/colleges | 🟢 Light | 1 query. College list (master data). |
| 141 | GET | /api/bo/reports/bill-master | 🟢 Light | 1 query. Bill master list. |
| 142 | POST | /api/bo/reports/pci-register | 🔴 Heavy | PCI register report. Joins large tables. |
| 143 | POST | /api/bo/reports/record-register | 🔴 Heavy | Record register report. Joins large tables. |
| 144 | POST | /api/bo/reports/get-user-details-by-regnolist | 🔴 Heavy | Batch lookup by reg number list. Multiple queries on applicationdtls (105K). |
| 145 | POST | /api/bo/reports/export-user-details-by-regnolist | ⚫ Critical | Same + Excel export. Loads all matching records into memory. |
| 146 | POST | /api/bo/reports/get-receipt-certy-list | 🔴 Heavy | Receipt + certificate list. Joins billhead + downloads. |
| | | **Financial Reports (`/api/bo/financial-reports/`)** | | |
| 147 | POST | /api/bo/financial-reports/accountLedger | 🔴 Heavy | 4-8 queries. Joins amttrans (920K, 133MB) + account tables with date filters. Largest table in DB. |
| 148 | POST | /api/bo/financial-reports/download-account-ledger | ⚫ Critical | Same + PDF generation on 920K row table data. |
| 149 | GET | /api/bo/financial-reports/accountnames | 🔴 Heavy | Same as accountLedger (routes to same method). |
| 150 | POST | /api/bo/financial-reports/cashBookDetail | 🔴 Heavy | 4-8 queries. Cash book joins on amttrans (920K) + bankentry. |
| 151 | POST | /api/bo/financial-reports/download-cashbook-details | ⚫ Critical | Same + PDF generation. |
| 152 | POST | /api/bo/financial-reports/bankBookDetail | 🔴 Heavy | 4-8 queries. Bank book joins on amttrans (920K) + bankentry. |
| 153 | POST | /api/bo/financial-reports/download-bankbook-details | ⚫ Critical | Same + PDF generation. |
| 154 | POST | /api/bo/financial-reports/jvBookDetail | 🔴 Heavy | 4-6 queries. JV book on amttrans (920K). |
| 155 | POST | /api/bo/financial-reports/download-jvbook-details | ⚫ Critical | Same + PDF generation. |
| 156 | POST | /api/bo/financial-reports/trialBalance | 🔴 Heavy | 4-8 queries. Aggregation across amttrans (920K) + all account tables. |
| 157 | POST | /api/bo/financial-reports/profitLoss | 🔴 Heavy | 4-8 queries. P&L aggregation across amttrans (920K). |
| 158 | GET | /api/bo/financial-reports/bill-master | 🟢 Light | 1 query. Bill master list. |
| | | **Certificates (`/api/bo/certificates/`)** | | |
| 159 | POST | /api/bo/certificates/updateRegDate | 🟢 Light | 1-2 queries. Date update. |
| 160 | GET | /api/bo/certificates/provisionalCertificate | 🟡 Medium | 2-3 queries. Provisional cert details. |
| 161 | POST | /api/bo/certificates/provisionalCertificate | 🔴 Heavy | 4-6 queries + PDF generation + S3 upload. Certificate save + generate. |
| 162 | GET | /api/bo/certificates/download-provisional-certificate/{srNo} | 🔴 Heavy | PDF generation + S3. |
| 163 | GET | /api/bo/certificates/provisionalCertificateList | 🟡 Medium | 2-3 queries. List with filters. |
| 164 | POST | /api/bo/certificates/originalCertificate | 🔴 Heavy | 4-6 queries + PDF generation + S3 upload. |
| 165 | GET | /api/bo/certificates/download-original-certificate/{srNo} | 🔴 Heavy | PDF generation + S3. |
| 166 | GET | /api/bo/certificates/originalCertificateList | 🟡 Medium | 2-3 queries. |
| 167 | GET | /api/bo/certificates/degreeAdditionCertificate | 🟡 Medium | 2-3 queries. |
| 168 | POST | /api/bo/certificates/degreeAdditionCertificate | 🔴 Heavy | 4-6 queries + PDF + S3. |
| 169 | GET | /api/bo/certificates/download-degree-additional-certificate/{srNo}/{rno} | 🔴 Heavy | PDF generation + S3. |
| 170 | GET | /api/bo/certificates/get-duplicate-certificate-list | 🟡 Medium | 2-3 queries. |
| 171 | GET | /api/bo/certificates/change-duplicate-certificate-status | 🟢 Light | 1-2 queries. Status update. |
| 172 | GET | /api/bo/certificates/nameChangeCertificate | 🟡 Medium | 2-3 queries. |
| 173 | POST | /api/bo/certificates/nameChangeCertificate | 🔴 Heavy | 4-6 queries + PDF + S3. |
| 174 | POST | /api/bo/certificates/goodStandingCertificate | 🔴 Heavy | 4-6 queries + PDF + S3. |
| 175 | GET | /api/bo/certificates/download-good-standing | 🔴 Heavy | PDF generation + S3. |
| 176 | GET | /api/bo/certificates/goodStandingCertificate | 🟡 Medium | 2-3 queries. |

| | | **Applications (`/api/applications/`)** | | |
| 177 | POST | /api/applications/save-duplicate-certificate | 🟡 Medium | 2-3 queries. Save duplicate cert request. |
| 178 | GET | /api/applications/search-applicant-app | 🔴 Heavy | Search on applicationdtls (105K) with multiple filters. Potentially full scan. |
| 179 | POST | /api/applications/change-election-status | 🟢 Light | 1-2 queries. Status update. |
| 180 | GET | /api/applications/delete-registered-app | ⚫ Critical | Deletes application records. Uses GET for delete operation. |
| 181 | GET | /api/applications/get-file-details | 🔴 Heavy | 6-10 queries. Full applicant file load (same as dashboard/get-file-details). |
| 182 | POST | /api/applications/update-penalty-status | 🟡 Medium | 2-3 queries. Penalty update. |
| 183 | POST | /api/applications/update-penalty-under-jan-vishwas-status | 🟡 Medium | 2-3 queries. |
| 184 | GET | /api/applications/get-user-details-by-regno | 🔴 Heavy | 4-6 queries across applicationdtls + otherappdtls. |
| 185 | GET | /api/applications/get-eligible-file-details | 🔴 Heavy | 6-10 queries. Eligible applicant full file. |
| 186 | POST | /api/applications/profile-pic-upload | 🟡 Medium | 1-2 queries + S3 upload. |
| 187 | POST | /api/applications/eligible-profile-pic-upload | 🟡 Medium | 1-2 queries + S3 upload. |
| 188 | POST | /api/applications/update-file-details | 🟡 Medium | 3-5 queries. Update applicant details. |
| 189 | POST | /api/applications/update-eligible-file-details | 🟡 Medium | 3-5 queries. |
| 190 | GET | /api/applications/document-list | 🟡 Medium | 2-3 queries. |
| 191 | GET | /api/applications/eligible-document-list | 🟡 Medium | 2-3 queries. |
| 192 | GET | /api/applications/download-zip-link | 🔴 Heavy | Generates zip of documents from S3. Multiple S3 calls. |
| 193 | GET | /api/applications/eligible-download-zip-link | 🔴 Heavy | Same for eligible docs. |
| 194 | POST | /api/applications/upload-document | 🟡 Medium | 1-2 queries + S3 upload. |
| 195 | POST | /api/applications/upload-eligible-document | 🟡 Medium | 1-2 queries + S3 upload. |
| 196 | GET | /api/applications/search-reentry-app | 🔴 Heavy | Search on otherappdtls (211K) for re-entry apps. |
| 197 | GET | /api/applications/check-duplicate/{id} | 🟡 Medium | 2-3 queries. Duplicate check. |
| 198 | GET | /api/applications/search-eligible-app | 🔴 Heavy | Search on eligible applicants. |
| 199 | GET | /api/applications/search-other-app | 🔴 Heavy | Search on otherappdtls (211K). |
| 200 | GET | /api/applications/search-address-change-app | 🔴 Heavy | Search on otherappdtls (211K). |
| 201 | GET | /api/applications/search-additionaldegree-app | 🔴 Heavy | Search on otherappdtls (211K). |
| 202 | GET | /api/applications/save-eligible | 🟡 Medium | 2-3 queries. Save eligible status. |
| 203 | GET | /api/applications/save-eligible-other | 🟡 Medium | 2-3 queries. |
| 204 | POST | /api/applications/hold-unhold-applicant | 🟡 Medium | 2-3 queries. Hold/unhold. |
| 205 | POST | /api/applications/change-hold-unhold-applicant | 🟡 Medium | 2-3 queries. |
| 206 | GET | /api/applications/all-hold-applicant | 🟡 Medium | 2-3 queries. List held applicants. |
| 207 | GET | /api/applications/pro-to-original-list | 🟡 Medium | 2-3 queries. Pro-to-original list. |
| 208 | GET | /api/applications/pro-to-original-reject | 🟢 Light | 1-2 queries. Reject request. |
| 209 | GET | /api/applications/get-application | 🟡 Medium | 2-3 queries. In/out register app. |
| 210 | POST | /api/applications/store-application | 🟡 Medium | 2-3 queries. Store in/out app. |
| 211 | GET | /api/applications/verify-document | 🟡 Medium | 2-3 queries. Doc verification. |
| 212 | GET | /api/applications/verify-eligible-document | 🟡 Medium | 2-3 queries. |
| 213 | GET | /api/applications/search | 🔴 Heavy | General search across applicationdtls (105K) + otherappdtls (211K). |
| 214 | POST | /api/applications/other-status-application | 🟡 Medium | 2-3 queries. Other app status change. |
| 215 | POST | /api/applications/change-other-status-application | 🟡 Medium | 2-3 queries. |
| 216 | GET | /api/applications/approve-other-application | 🟡 Medium | 2-3 queries. Approve other app. |
| 217 | GET | /api/applications/hospitals | 🟢 Light | 1 query. Master list. |
| 218 | GET | /api/applications/colleges | 🟢 Light | 1 query. Master list. |
| 219 | GET | /api/applications/boards | 🟢 Light | 1 query. Master list. |
| 220 | GET | /api/applications/states | 🟢 Light | 1 query. Master list. |
| 221 | GET | /api/applications/councils | 🟢 Light | 1 query. Master list. |
| 222 | GET | /api/applications/get-closed-apps | 🟡 Medium | 2-3 queries. Closed apps list. |
| 223 | GET | /api/applications/approve-fresh-all-application | ⚫ Critical | Bulk approve. Updates multiple records in applicationdtls (105K). |
| 224 | GET | /api/applications/get-user-details-by-regno-fileno | 🔴 Heavy | 4-6 queries across large tables. |
| 225 | GET | /api/applications/delete-with-refund | ⚫ Critical | Deletes app + processes refund. Uses GET for destructive operation. |
| 226 | POST | /api/applications/delete-prefresh-application | ⚫ Critical | Deletes pre-fresh application records. |
| 227 | POST | /api/applications/download-electrol-application-report | ⚫ Critical | PDF report generation. Joins large tables. |
| 228 | GET | /api/applications/get-scrutiny-verification | 🟡 Medium | 2-3 queries. Scrutiny verification list. |
| 229 | GET | /api/applications/user-history | 🔴 Heavy | 4-6 queries. Full user history across multiple tables. |

| | | **Scrutiny (`/api/scrutiny/`)** | | |
| 230 | GET | /api/scrutiny/hospitals | 🟢 Light | 1 query. Master list. |
| 231 | GET | /api/scrutiny/get-scrutiny-application-list | 🔴 Heavy | 3-5 queries. Scrutiny list with joins on applicationdtls. |
| 232 | GET | /api/scrutiny/get-scrutiny-reentry-list | 🔴 Heavy | 3-5 queries on otherappdtls (211K). |
| 233 | GET | /api/scrutiny/get-scrutiny-degreeadd-list | 🔴 Heavy | 3-5 queries on otherappdtls (211K). |
| 234 | GET | /api/scrutiny/get-scrutiny-data | 🟡 Medium | 2-3 queries. Single scrutiny record. |
| 235 | GET | /api/scrutiny/get-scrutiny-data-by-rno | 🟡 Medium | 2-3 queries. Lookup by reg number. |
| 236 | GET | /api/scrutiny/get-file-details | 🔴 Heavy | 6-10 queries. Full file details. |
| 237 | GET | /api/scrutiny/get-file-details-reentry | 🔴 Heavy | 6-10 queries. Re-entry file details. |
| 238 | POST | /api/scrutiny/save-scrutiny-data | 🟡 Medium | 3-4 queries. Save scrutiny + update status. |
| 239 | POST | /api/scrutiny/save-scrutiny-degreeadd-data | 🟡 Medium | 3-4 queries. |
| 240 | POST | /api/scrutiny/save-reentry-scrutiny-data | 🟡 Medium | 3-4 queries. |
| 241 | POST | /api/scrutiny/save-degree-add | 🟡 Medium | 3-4 queries. |
| 242 | GET | /api/scrutiny/get-file-details-by-fileno | 🔴 Heavy | 6-10 queries. Same as dashboard index. |
| 243 | GET | /api/scrutiny/document-list | 🟡 Medium | 2-3 queries. |
| 244 | GET | /api/scrutiny/download-zip-link | 🔴 Heavy | Zip generation from S3. |
| 245 | POST | /api/scrutiny/send-scrutiny-verification | 🟡 Medium | 2-3 queries + notification. |
| 246 | GET | /api/scrutiny/get-scrutiny-verification | 🟡 Medium | 2-3 queries. |
| 247 | GET | /api/scrutiny/get-original-pending | 🟡 Medium | 2-3 queries. |
| 248 | POST | /api/scrutiny/update-scrutiny-verification | 🟡 Medium | 2-3 queries. |
| 249 | POST | /api/scrutiny/add-scrutiny-verification | 🟡 Medium | 2-3 queries. |
| 250 | POST | /api/scrutiny/set-scrutiny-hold | 🟡 Medium | 2-3 queries. |
| 251 | POST | /api/scrutiny/scrutiny-app-transfer | 🟡 Medium | 2-3 queries. Transfer app between scrutiny users. |
| | | **Appointments (`/api/bo/appointment/`)** | | |
| 252 | POST | /api/bo/appointment/add-new-slot | 🟡 Medium | 2-3 queries. Create slot. |
| 253 | POST | /api/bo/appointment/update-slot | 🟡 Medium | 2-3 queries. Update slot. |
| 254 | GET | /api/bo/appointment/appointment | 🟡 Medium | 2-3 queries. List appointments. |
| 255 | DELETE | /api/bo/appointment/appointment | 🟢 Light | 1 delete. |
| 256 | POST | /api/bo/appointment/download-slot-details | 🔴 Heavy | PDF/Excel generation of slot data. Joins slot_details (62K). |
| 257 | GET | /api/bo/appointment/get-slot-details | 🟡 Medium | 2-3 queries on slot_details (62K). |
| 258 | GET | /api/bo/appointment/get-all-slot-details | 🔴 Heavy | All slot details. Full scan on slot_details (62K). |
| 259 | GET | /api/bo/appointment/datewise-appointment-graph | 🟡 Medium | 2-3 queries. GROUP BY date on slot_details. |
| 260 | GET | /api/bo/appointment/datewise-appointment-list | 🟡 Medium | 2-3 queries. Date-wise list. |
| | | **In/Out Register (`/api/in-out/`)** | | |
| 261 | GET | /api/in-out/get-globalval | 🟢 Light | 1 query. Global value. |
| 262 | GET | /api/in-out/get-inward | 🟡 Medium | 2-3 queries. Inward register list. |
| 263 | GET | /api/in-out/get-outward | 🟡 Medium | 2-3 queries. Outward register list. |
| 264 | POST | /api/in-out/store-inward | 🟡 Medium | 2-3 queries. Store inward entry. |
| 265 | POST | /api/in-out/store-outward | 🟡 Medium | 2-3 queries. Store outward entry. |
| 266 | DELETE | /api/in-out/deleteInOut/{id} | 🟢 Light | 1 delete. |
| | | **Letters (`/api/letters/`)** | | |
| 267 | GET | /api/letters/get-noc-letters | 🟡 Medium | 2-3 queries. NOC letter list. |
| 268 | POST | /api/letters/storenoc-letter | 🟡 Medium | 2-3 queries. Store NOC letter. |
| 269 | GET | /api/letters/download-noc-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 270 | GET | /api/letters/get-ruhs-letter | 🟡 Medium | 2-3 queries. |
| 271 | POST | /api/letters/store-ruhs-letter | 🟡 Medium | 2-3 queries. |
| 272 | GET | /api/letters/download-ruhs-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 273 | GET | /api/letters/get-nios-letter | 🟡 Medium | 2-3 queries. |
| 274 | POST | /api/letters/store-nios-letter | 🟡 Medium | 2-3 queries. |
| 275 | GET | /api/letters/download-nios-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 276 | GET | /api/letters/get-verification-letter | 🟡 Medium | 2-3 queries. |
| 277 | POST | /api/letters/store-verification-letter | 🟡 Medium | 2-3 queries. |
| 278 | GET | /api/letters/download-verification-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 279 | GET | /api/letters/get-reminder-letter | 🟡 Medium | 2-3 queries. |
| 280 | POST | /api/letters/store-reminder-Letter | 🟡 Medium | 2-3 queries. |
| 281 | GET | /api/letters/download-reminder-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 282 | GET | /api/letters/get-cancellation-letter | 🟡 Medium | 2-3 queries. |
| 283 | POST | /api/letters/store-cancellation-letter | 🟡 Medium | 2-3 queries. |
| 284 | GET | /api/letters/download-cancellation-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 285 | GET | /api/letters/get-transfer-letter | 🟡 Medium | 2-3 queries. |
| 286 | POST | /api/letters/store-transfer-letter | 🟡 Medium | 2-3 queries. |
| 287 | GET | /api/letters/download-transfer-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 288 | GET | /api/letters/get-training-letter | 🟡 Medium | 2-3 queries. |
| 289 | POST | /api/letters/store-training-letter | 🟡 Medium | 2-3 queries. |
| 290 | GET | /api/letters/download-training-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 291 | GET | /api/letters/get-dataflow-letter | 🟡 Medium | 2-3 queries. |
| 292 | POST | /api/letters/store-dataflow-letter | 🟡 Medium | 2-3 queries. |
| 293 | GET | /api/letters/download-dataflow-letter/{id} | 🔴 Heavy | PDF generation + S3. |
| 294 | GET | /api/letters/hospitals | 🟢 Light | 1 query. Master list. |
| 295 | GET | /api/letters/colleges | 🟢 Light | 1 query. Master list. |
| 296 | GET | /api/letters/boards | 🟢 Light | 1 query. Master list. |
| 297 | GET | /api/letters/councils | 🟢 Light | 1 query. Master list. |
| 298 | GET | /api/letters/university | 🟢 Light | 1 query. Master list. |

| | | **Dynamic Content (`/api/dynamic/`)** | | |
| 299 | POST | /api/dynamic/change-banner | 🟡 Medium | 1-2 queries + S3 upload. |
| 300 | GET | /api/dynamic/get-banner-image | 🟢 Light | 1 query. |
| 301 | POST | /api/dynamic/add-notice | 🟡 Medium | 1-2 queries + S3 upload. |
| 302 | POST | /api/dynamic/change-notice-status | 🟢 Light | 1 update. |
| 303 | GET | /api/dynamic/get-all-notice | 🟢 Light | 1 query. |
| 304 | GET | /api/dynamic/delete-notice | 🟢 Light | 1 delete. Uses GET for delete. |
| 305 | POST | /api/dynamic/add-latest-update | 🟡 Medium | 1-2 queries + S3 upload. |
| 306 | POST | /api/dynamic/change-latest-update-status | 🟢 Light | 1 update. |
| 307 | GET | /api/dynamic/get-all-latest-update | 🟢 Light | 1 query. |
| 308 | GET | /api/dynamic/delete-latest-update | 🟢 Light | 1 delete. Uses GET for delete. |
| 309 | POST | /api/dynamic/add-gallery-images | 🟡 Medium | 1-2 queries + S3 upload. |
| 310 | POST | /api/dynamic/change-image-status | 🟢 Light | 1 update. |
| 311 | GET | /api/dynamic/get-all-gallery | 🟢 Light | 1 query. |
| 312 | GET | /api/dynamic/delete-gallery-images | 🟢 Light | 1 delete. Uses GET for delete. |
| 313 | GET | /api/dynamic/get-contact-us-data | 🟢 Light | 1 query. |
| | | **User Management (`/api/user-management/`)** | | |
| 314 | POST | /api/user-management/create-user | 🟡 Medium | 2-3 queries. Create/edit admin user. |
| 315 | GET | /api/user-management/list-roles | 🟢 Light | 1 query. |
| 316 | GET | /api/user-management/list-users | 🟡 Medium | 2-3 queries. Users with roles. |
| 317 | DELETE | /api/user-management/delete-user/{id} | 🟢 Light | 1 delete. |
| | | **Role Management (`/api/role-management/`)** | | |
| 318 | GET | /api/role-management/list-roles | 🟢 Light | 1 query. |
| 319 | POST | /api/role-management/save-role-permissions | 🟡 Medium | 2-3 queries. Save permissions. |
| 320 | POST | /api/role-management/update-role-data | 🟡 Medium | 2-3 queries. |
| 321 | GET | /api/role-management/get-role-data-by-id/{id} | 🟢 Light | 1-2 queries. |
| 322 | POST | /api/role-management/update-user-menu-data | 🟡 Medium | 2-3 queries. |
| | | **Account (`/api/account/`)** | | |
| 323 | GET | /api/account/get-voucher | 🔴 Heavy | Voucher generation. Queries amttrans (920K) + account tables. |
| 324 | POST | /api/account/download-jventry-details | 🔴 Heavy | PDF generation of JV entries. |
| 325 | GET | /api/account/get-account-group | 🟢 Light | 1 query. Account groups. |
| 326 | POST | /api/account/store-account-group | 🟢 Light | 1 insert. |
| 327 | DELETE | /api/account/account-group | 🟢 Light | 1 delete. |
| 328 | GET | /api/account/get-account | 🟢 Light | 1-2 queries. Account list. |
| 329 | POST | /api/account/create-account | 🟢 Light | 1 insert. |
| 330 | DELETE | /api/account/account | 🟢 Light | 1 delete. |
| 331 | GET | /api/account/get-bankentry | 🟡 Medium | 2-3 queries. Bank entries with joins. |
| 332 | POST | /api/account/store-bankentry | 🟡 Medium | 3-5 queries. Store bank entry + amttrans inserts. |
| 333 | GET | /api/account/get-cashentry | 🟡 Medium | 2-3 queries. Cash entries. |
| 334 | POST | /api/account/store-cashentry | 🟡 Medium | 3-5 queries. Store cash entry + amttrans inserts. |
| 335 | GET | /api/account/get-jventry | 🟡 Medium | 2-3 queries. JV entries. |
| 336 | POST | /api/account/store-jventry | 🟡 Medium | 3-5 queries. Store JV entry + amttrans inserts. |
| 337 | GET | /api/account/get-billing | 🟡 Medium | 2-3 queries. Billing data. |
| 338 | GET | /api/account/get-allrefrance-number | 🔴 Heavy | Loads all reference numbers from amttrans (920K). Potentially full scan. |

| | | **College Management (`/api/college/`)** | | |
| 339 | GET | /api/college/college-login-approval | 🟡 Medium | 2-3 queries. College login approval list. |
| 340 | POST | /api/college/add-pci-college-approval | 🟡 Medium | 2-3 queries + S3 upload. |
| 341 | POST | /api/college/update-pci-college-approval | 🟡 Medium | 2-3 queries + S3 upload. |
| 342 | GET | /api/college/list-pci-college-approval | 🟡 Medium | 2-3 queries. PCI approval list. |
| 343 | GET | /api/college/list-pci-college-approval-pending | 🟡 Medium | 2-3 queries. Pending approvals. |
| 344 | POST | /api/college/pci-approval-reupload-request | 🟡 Medium | 2-3 queries. |
| 345 | POST | /api/college/change-pci-college-approval-status | 🟡 Medium | 2-3 queries. Status change. |
| 346 | POST | /api/college/upload-college-admitted-list | 🔴 Heavy | Excel parsing + bulk insert. Reads Excel file + inserts many rows. |
| 347 | POST | /api/college/reupload-college-admitted-list | 🔴 Heavy | Delete old + re-parse Excel + bulk insert. |
| 348 | GET | /api/college/college-admitted-list-master | 🟡 Medium | 2-3 queries. Admitted list master. |
| 349 | GET | /api/college/college-admitted-list-master-pending | 🟡 Medium | 2-3 queries. |
| 350 | GET | /api/college/college-admitted-list | 🟡 Medium | 2-3 queries. Student list. |
| 351 | POST | /api/college/admitted-list-change-status | 🟡 Medium | 2-3 queries. |
| 352 | POST | /api/college/upload-passout-list | 🔴 Heavy | Excel parsing + bulk insert. |
| 353 | POST | /api/college/reupload-passout-list | 🔴 Heavy | Delete old + re-parse + bulk insert. |
| 354 | GET | /api/college/passout-list-master | 🟡 Medium | 2-3 queries. |
| 355 | GET | /api/college/passout-list-master-pending | 🟡 Medium | 2-3 queries. |
| 356 | GET | /api/college/passout-list | 🟡 Medium | 2-3 queries. |
| 357 | POST | /api/college/passout-list-change-status | 🟡 Medium | 2-3 queries. |
| 358 | POST | /api/college/register | 🟡 Medium | 3-4 queries. Register user from college. |
| 359 | POST | /api/college/edit-college | 🟡 Medium | 2-3 queries + S3 upload. |
| 360 | GET | /api/college/college-list | 🟡 Medium | 2-3 queries. |
| 361 | GET | /api/college/get-college-details | 🟡 Medium | 2-3 queries. |
| 362 | DELETE | /api/college/delete-college/{id} | 🟢 Light | 1 delete. |
| 363 | GET | /api/college/colleges | 🟢 Light | 1 query. Master list. |
| 364 | POST | /api/college/add-update-refresher-course | 🟡 Medium | 3-4 queries. Create/update refresher course + slots. |
| 365 | POST | /api/college/add-update-offline-refresher-course | 🟡 Medium | 3-4 queries. |
| 366 | GET | /api/college/refresher-course-list | 🟡 Medium | 2-3 queries with relations. |
| 367 | GET | /api/college/refresher-offline-course-list | 🟡 Medium | 2-3 queries. |
| 368 | GET | /api/college/refresher-course-by-id | 🟡 Medium | 2-3 queries. |
| 369 | GET | /api/college/offline-refresher-course-by-id | 🟡 Medium | 2-3 queries. |
| 370 | GET | /api/college/booked-slots-list | 🔴 Heavy | 3-5 queries. Booked slots with user details + joins. |
| 371 | GET | /api/college/offline-booked-slots-list | 🔴 Heavy | 3-5 queries. Same for offline. |
| 372 | POST | /api/college/attendance-offline-refresher | 🟡 Medium | 2-3 queries. Mark attendance. |
| 373 | POST | /api/college/add-update-refresher-quiz-data | 🟡 Medium | 2-3 queries. Quiz CRUD. |
| 374 | DELETE | /api/college/delete-quiz/{id} | 🟢 Light | 1 delete. |
| 375 | GET | /api/college/get-refresher-quiz-data | 🟡 Medium | 2-3 queries. |
| 376 | POST | /api/college/change-lecture-status | 🟢 Light | 1-2 queries. |
| 377 | POST | /api/college/change-refresher-course-status | 🟢 Light | 1-2 queries. |
| 378 | POST | /api/college/change-offline-refresher-course-status | 🟢 Light | 1-2 queries. |
| 379 | POST | /api/college/change-quiz-status | 🟢 Light | 1-2 queries. |
| 380 | GET | /api/college/dashboard-stats | 🔴 Heavy | 5-8 queries. College dashboard aggregation. COUNT on multiple tables. |
| 381 | POST | /api/college/refresher-certy-change-status | 🔴 Heavy | Certificate generation + PDF + S3 + queue job. |
| 382 | POST | /api/college/offline-refresher-certy-change-status | 🔴 Heavy | Same for offline certificates. |
| 383 | DELETE | /api/college/delete-refresher-course/{id} | 🟢 Light | 1 delete. |
| 384 | DELETE | /api/college/delete-offline-refresher-course/{id} | 🟢 Light | 1 delete. |
| 385 | GET | /api/college/refresher-user-details | 🟡 Medium | 2-3 queries. User details for refresher. |
| 386 | POST | /api/college/generate-comember-certificate | 🔴 Heavy | PDF generation + S3 upload + 3-4 queries. |
| 387 | GET | /api/college/get-refresher-member-certificate-list | 🟡 Medium | 2-3 queries. |
| 388 | POST | /api/college/generate-comember-offlinecertificate | 🔴 Heavy | PDF generation + S3 upload + 3-4 queries. |
| 389 | GET | /api/college/get-offlinerefresher-member-certificate-list | 🟡 Medium | 2-3 queries. |

---

### Endpoint Summary

| Category | Total | 🟢 Light | 🟡 Medium | 🔴 Heavy | ⚫ Critical |
|---|---|---|---|---|---|
| Public (No Auth) | 92 | 24 | 28 | 12 | 28 |
| Protected (Auth) | 389 | 82 | 168 | 101 | 38 |
| **Total** | **481** | **106** | **196** | **113** | **66** |

### Top 10 Most Dangerous Endpoints (Performance + Security)

| # | Endpoint | Why |
|---|---|---|
| 1 | GET /api/dashboard | PUBLIC, 12+ queries, GROUP BY on 249K + 211K row tables. Confirmed 1.49s. Anyone can hammer this. |
| 2 | GET /api/user/dashboard | 20-25 queries per user. 500 users = 12,000+ queries. Biggest bottleneck. |
| 3 | GET /api/amt-duplicate | PUBLIC, GROUP BY on amttrans (920K rows, 133MB). Full table scan. No auth. |
| 4 | GET /api/billchild-duplicate | PUBLIC, GROUP BY on billchild (728K rows). Full table scan. No auth. |
| 5 | POST /api/generic/login | PUBLIC, 4 queries per login, no rate limit. 500 logins = 2,000 queries. |
| 6 | POST /api/bo/reports/download-fee-register | Joins billhead (249K) + billchild (728K) + PDF generation. Memory intensive. |
| 7 | POST /api/bo/financial-reports/download-account-ledger | Queries amttrans (920K, 133MB) + PDF generation. Largest table. |
| 8 | GET /api/amt-trans | PUBLIC, full account migration on 920K rows. Modifies data. No auth. |
| 9 | GET /api/store-renewal-data | PUBLIC, bulk data insert + WhatsApp API calls. No auth. |
| 10 | POST /api/bo/reports/statisticsValidation | 8-10 queries with COUNT + GROUP BY on three 100K+ row tables. |
