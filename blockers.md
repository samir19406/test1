# Code-Level Fixes for 500 Concurrent Users — Problem Analysis & Solutions

## Background

When ~500 users use the system simultaneously (login + dashboard + refresher course booking/exams), the backend becomes unresponsive. Database indexes have been added (Phase 1 & 2 done), which reduced the public dashboard from 1.49s to 0.45s. But there are application-level bottlenecks that indexes alone cannot fix.

---

## Problem 1: Refresher Slot Booking Serializes All Users

**File:** `app/Http/Controllers/College/CollegeController.php` — `bookRefresherCourseSlot()` (line ~1482)

**What happens when 500 users book the same refresher course at the same time:**

1. User A hits the endpoint → `lockForUpdate()` locks the `refresher_course` row → checks duplicate → creates booking → updates available_slot → sends email via Mailgun API (1-2 sec) → returns response
2. User B is WAITING this entire time because `lockForUpdate()` means only one user can process at a time
3. User C is waiting behind User B
4. ...User 500 is waiting behind User 499

The `lockForUpdate()` itself is correct — you need it to prevent overbooking. The problem is what happens WHILE the lock is held: the code sends an email via Mailgun API synchronously. That takes 1-2 seconds per user. So 500 users × 1-2 seconds = 500-1000 seconds of serial processing. Users at the back of the queue wait 8-15 minutes.

**Solution:** Move the `directMail()` call to a queued job. The database transaction (lock → check → insert → update → commit) takes ~10ms. The email takes 1-2 seconds. By queuing the email, the lock is held for 10ms instead of 2 seconds. 500 users × 10ms = 5 seconds total instead of 15 minutes.

Same issue exists in `bookOfflineRefresherCourseSlot()` (line ~1574).

---

## Problem 2: Exam Submission Runs Synchronously (Blocks the Request)

**File:** `app/Http/Controllers/College/CollegeController.php` — `saveExam()` (line ~2160)
**File:** `.env` — `QUEUE_CONNECTION=sync`

When a user submits their refresher exam, the code calls `ProcessRefresherExam::dispatch($arrInput)`. This LOOKS like it sends the job to a background queue. But because `.env` has `QUEUE_CONNECTION=sync`, Laravel doesn't actually queue it — it runs the entire job inline, inside the HTTP request.

**What `ProcessRefresherExam` does:**

- Locks the refresher slot row (`lockForUpdate`)
- Loops through every answer and inserts into `refresher_exam_user_answers` (927K rows — the largest table in the DB)
- Calculates the score
- If passed: generates a PDF certificate, uploads to S3, sends email via Mailgun

All of this happens while the user stares at a loading spinner. With 500 users submitting exams:

- 500 PHP workers are all busy doing exam processing
- Each one is inserting rows, generating PDFs, uploading to S3, calling Mailgun
- No PHP workers are free to handle new requests
- The entire application becomes unresponsive — not just exams, everything (login, dashboard, all endpoints)

**Solution:** Change `QUEUE_CONNECTION=sync` to `QUEUE_CONNECTION=redis` in `.env` and start a queue worker (`php artisan queue:work`). Now `dispatch()` actually sends the job to Redis and returns instantly. The user gets "Exam Submitted" in milliseconds. The worker processes exams in the background one at a time without blocking any HTTP requests.

**Prerequisite:** Redis must be installed and running. Config already exists in `config/database.php`.

---

## Problem 3: User Dashboard Runs 20-25 Queries Per Load

**File:** `app/Http/Controllers/User/DashboardController.php` — `index()` (line ~29)

Every user hits this endpoint immediately after login. It currently runs 20-25 separate database queries for a single user. The same tables are queried repeatedly:

| Table | Rows | Times Queried Per User | What It Does |
|---|---|---|---|
| dbo_tblapplicationdtls | 105K | 8 times | Same user_id with slightly different filters each time. Each query is a separate round-trip to MySQL. |
| dbo_tblotherappdtls | 211K | 7 times | Same EntryNo. Checks renewal status, re-entry status, degree addition, good standing, duplicate certificate — each as a separate query. |
| EligibleApplicant | 416 | 2-3 times | Same user_id, even though it's already loaded into `$eligibleDetailsObj` at the top of the method. |
| dbo_billhead | 249K | 2-3 times | Queried separately for each application type to get receipt numbers. |

At 500 users: 500 × 22 queries = **11,000 queries** hitting the database within seconds of everyone logging in.

**Solution:** Fetch each table once per user, then filter the results in PHP memory:

| Change | Current | Recommended |
|---|---|---|
| Applicant data | 8 separate queries with different filters | One query: fetch all records for user_id, filter by applicationType/appStatus in PHP |
| Other app data | 7 separate queries with different AppNm/AppAccept | One query: fetch all records for EntryNo, filter by AppNm/AppAccept in PHP |
| Eligible data | 2-3 queries for same user_id | Reuse `$eligibleDetailsObj` already loaded at top of method |
| Bill receipts | 2-3 queries per application type | One query: fetch all BillHead for user's SrNo, match in PHP |

**Target:** 20-25 queries → 5-6 queries. At 500 users: 11,000 queries → 3,000 queries.

---

## Problem 4: Every Authenticated Request Runs 2-3 Extra Queries

**File:** `app/Http/Middleware/JsonMiddleware.php`

Before any controller code runs, the JWT middleware loads the user's permissions from the database on every single request. This means:

- User opens dashboard → 2-3 permission queries + 22 dashboard queries
- User clicks on a page → 2-3 permission queries + page queries
- User clicks on another page → 2-3 more permission queries

With 500 users each making 3-4 requests: 500 × 4 × 2.5 = **5,000 extra queries** just for permission checks. Permissions almost never change — a user's role stays the same for weeks/months.

**Solution:** After loading permissions the first time, cache them in Redis keyed by user ID with a 5-minute TTL. On subsequent requests, read from cache. Invalidate the cache entry when an admin changes that user's permissions.

---

## Problem 5: Login Runs 4 Queries (Should Be 2)

**File:** `app/Http/Controllers/User/UserController.php` — `login()` method

**Current login flow:**

| Step | Query | Purpose | Redundant? |
|---|---|---|---|
| 1 | `SELECT * FROM users WHERE email = ?` | Check if email exists | Yes — attempt() does this |
| 2 | `SELECT * FROM users WHERE email = ? AND is_email_verified = 1` | Check if verified | Yes — can check after attempt() |
| 3 | `JWTAuth::attempt()` | Same email lookup + password check | No — this is the real auth |
| 4 | Load user with role and permissions | Get user data for JWT token | No — needed for response |

Queries 1 and 2 are completely redundant. `JWTAuth::attempt()` already returns false if the user doesn't exist. You can check `is_email_verified` on the user object after a successful attempt.

At 500 logins: 500 × 4 = 2,000 queries. Should be 500 × 2 = 1,000 queries.

**Solution:** Remove queries 1 and 2. Call `JWTAuth::attempt()` first. If it succeeds, check `is_email_verified` on the returned user. If not verified, return error.

---

## Priority & Effort

| # | Fix | File(s) | Effort | Impact at 500 Users | Why This Order |
|---|---|---|---|---|---|
| 1 | .env: QUEUE_CONNECTION=redis + start worker | `.env` | 15 min | Exam + certificate processing stops blocking HTTP requests. Entire app stays responsive during exams. | One-line change, biggest impact for refresher scenario |
| 2 | Move emails to queued jobs in slot booking | `CollegeController.php` | 1-2 hours | Slot booking lock held for 10ms instead of 2 seconds. 500 bookings complete in 5 sec instead of 15 min. | Directly fixes the serialization bottleneck |
| 3 | Dashboard query consolidation | `User/DashboardController.php` | 2-3 hours | 11,000 queries → 3,000 queries. Dashboard loads 3-4× faster. | Most complex change but biggest query reduction |
| 4 | Middleware permission caching | `JsonMiddleware.php` | 30 min | Eliminates 5,000 unnecessary queries at 500 users. | Simple cache layer, affects every request |
| 5 | Login query cleanup | `User/UserController.php` | 30 min | 2,000 → 1,000 queries at 500 logins. | Quick win, low risk |

---

## Summary

Fix 1 is a one-line `.env` change. Fix 2 is moving ~10 lines of email code into a job class. Together they solve the refresher course blocking problem completely.

Fixes 3-5 are query optimization — they reduce the total database load so the server can handle 500 users without resource exhaustion.

**Total estimated effort: 5-7 hours of developer time.**
