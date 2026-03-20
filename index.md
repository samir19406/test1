# Database Index Report — GSPC Backend

## Database Overview

| Metric | Value |
|---|---|
| Total Tables | 143 |
| Tables with data (rows > 0) | 99 |
| Tables with ONLY primary key | 52 (of the active ones) |
| Tables with NO indexes at all | 40+ (mostly empty/legacy) |
| Total DB Size | 1,194 MB |
| Buffer Pool | 46 GB (entire DB fits in memory 38×) |

---

## Current Index Status — Active Tables (1,000+ rows)

Tables sorted by row count. These are the ones that matter for performance.

| # | Table | Rows | Size (MB) | Current Indexes | Missing Indexes? |
|---|---|---|---|---|---|
| 1 | refresher_exam_user_answers | 927,644 | 151.39 | PRIMARY, college_id | ⚠️ Yes |
| 2 | dbo_amttrans | 920,534 | 132.70 | PRIMARY only | 🔴 Critical |
| 3 | dbo_billchild | 728,397 | 93.64 | PRIMARY, CrYear, CrYear_2, CrYear_3 | ⚠️ Yes |
| 4 | dbo_tbluploaddocs | 474,038 | 85.61 | PRIMARY only | 🔴 Critical |
| 5 | dbo_tblinoutregister | 294,502 | 58.61 | PRIMARY only | ⚠️ Yes |
| 6 | dbo_billhead | 248,940 | 51.59 | PRIMARY, BillSrNo | ⚠️ Yes |
| 7 | dbo_tblotherappdtls | 210,755 | 60.14 | PRIMARY, EntryNo, IX_1, IX_2 | ⚠️ Yes |
| 8 | dbo_tbldownloads | 180,474 | 31.56 | PRIMARY, uq_day_unique | 🔴 Critical |
| 9 | dbo_tbltxns | 155,627 | 271.77 | PRIMARY only | ⚠️ Yes |
| 10 | dbo_tblapplicationdtls | 104,917 | 140.78 | PRIMARY, IX_1, LastVDate, StudF, StudL, StudM, Validity date | 🔴 Critical (no user_id) |
| 11 | dbo_tblscrutiny | 97,296 | 19.56 | PRIMARY only | ⚠️ Yes |
| 12 | users | 79,428 | 13.55 | PRIMARY, uq_email, uq_mobile | ✅ OK |
| 13 | slot_details | 62,476 | 4.52 | PRIMARY only | 🔴 Critical |
| 14 | refresher_course_slot_booking | 40,328 | 16.52 | PRIMARY, refresher_course_id, uniq_user_course | ✅ OK |
| 15 | dbo_tblapplicationdtls_profile | 35,088 | 5.52 | PRIMARY only | ⚠️ Yes |
| 16 | dbo_refresherchild | 23,160 | 2.52 | PRIMARY only | ⚠️ Yes |
| 17 | dbo_bankentry | 16,241 | 1.52 | PRIMARY only | ⚠️ Yes |
| 18 | pro_to_original | 14,064 | 4.52 | PRIMARY only | ⚠️ Yes |
| 19 | dbo_addrchange | 12,568 | 3.52 | PRIMARY only | ⚠️ Yes |
| 20 | dbo_tblorgapproval | 11,756 | 4.52 | PRIMARY only | ⚠️ Yes |
| 21 | contact_us | 11,696 | 2.52 | PRIMARY only | 🟢 Low priority |
| 22 | college_admitted_student_list | 7,785 | 1.52 | PRIMARY only | 🟢 Low priority |
| 23 | forgot_password | 7,126 | 1.52 | PRIMARY only | 🟢 Low priority |
| 24 | dbo_tblappbookings | 6,981 | 0.34 | PRIMARY only | 🟢 Low priority |
| 25 | renewalendyearmsg | 6,958 | 1.52 | PRIMARY only | 🟢 Low priority |
| 26 | dbo_tblgoodstanddtls | 5,339 | 3.52 | PRIMARY only | 🟢 Low priority |

---

## Indexes To Add — Priority Order

### Priority 1 — Critical (Run immediately, biggest impact)

These indexes affect the user dashboard (hit by every logged-in user) and the public dashboard (hit by anyone).

| # | Table | Rows | Index Name | Columns | Queries That Benefit | Expected Speedup |
|---|---|---|---|---|---|---|
| 1 | dbo_tblapplicationdtls | 104,917 | idx_app_userid | user_id | User dashboard runs 8 queries on this table filtering by user_id. Currently 0.6-0.7s per query (full scan). | 0.7s → 0.005s per query |
| 2 | dbo_tblapplicationdtls | 104,917 | idx_app_userid_type_status | user_id, applicationType, appStatus | Dashboard territory queries filter by all 3 columns. Composite index covers them in one lookup. | Eliminates 4 full scans per dashboard load |
| 3 | dbo_tbldownloads | 180,474 | idx_downloads_srno_type | SrNo, type, sub_type | Dashboard loads provisional certificate URL. 180K rows with no index on SrNo. | 0.2s → 0.001s |
| 4 | slot_details | 62,476 | idx_slot_userid_status | user_id, status | Dashboard checks appointment status. 62K rows, only has primary key. | 0.05s → 0.001s |
| 5 | dbo_billhead | 248,940 | idx_bill_appnm_paydt | AppNm, PayDt | Every report endpoint filters by AppNm + PayDt. 249K rows. | Reports 50-70% faster |
| 6 | dbo_billhead | 248,940 | idx_bill_created_at | created_at | Public dashboard runs 4 GROUP BY queries filtering by created_at. | Public dashboard 60% faster |
| 7 | dbo_billhead | 248,940 | idx_bill_entryno_appsrno | EntryNo, AppSrNo | Dashboard receipt lookups. User dashboard queries this per application type. | 0.1-0.15s → 0.001s |

**SQL to run:**

```sql
-- Priority 1: Critical indexes (run these first)
CREATE INDEX idx_app_userid ON dbo_tblapplicationdtls (user_id);
CREATE INDEX idx_app_userid_type_status ON dbo_tblapplicationdtls (user_id, applicationType, appStatus);
CREATE INDEX idx_downloads_srno_type ON dbo_tbldownloads (SrNo, type, sub_type);
CREATE INDEX idx_slot_userid_status ON slot_details (user_id, status);
CREATE INDEX idx_bill_appnm_paydt ON dbo_billhead (AppNm, PayDt);
CREATE INDEX idx_bill_created_at ON dbo_billhead (created_at);
CREATE INDEX idx_bill_entryno_appsrno ON dbo_billhead (EntryNo, AppSrNo);
```

---

### Priority 2 — High (Run same day, affects reports and other app queries)

| # | Table | Rows | Index Name | Columns | Queries That Benefit | Expected Speedup |
|---|---|---|---|---|---|---|
| 8 | dbo_tblotherappdtls | 210,755 | idx_other_appnm_entdt | AppNm, EntDt | Public dashboard pending counts (6 queries). Report endpoints filter by AppNm + date. | Dashboard pending section 50% faster |
| 9 | dbo_tblotherappdtls | 210,755 | idx_other_entryno_accept_appsrno | EntryNo, AppAccept, AppSrNo | User dashboard queries this 7 times per user with these filters. | 7 queries per user become instant |
| 10 | dbo_amttrans | 920,534 | idx_amttrans_refno | RefNo | Financial reports and migration endpoints filter by RefNo. Largest table, 133MB. | Financial reports 60-80% faster |
| 11 | dbo_amttrans | 920,534 | idx_amttrans_currdate | CurrDate | Financial reports filter by date on 920K rows. | Date-range reports much faster |
| 12 | dbo_billchild | 728,397 | idx_billchild_billsrno | BillSrNo | Joined with billhead in all report downloads. 728K rows, no index on join column. | Report PDF generation 50% faster |
| 13 | dbo_tbluploaddocs | 474,038 | idx_uploaddocs_srno | SrNo | Document list loading for applications. 474K rows, only primary key. | Document loading 70% faster |

**SQL to run:**

```sql
-- Priority 2: High impact indexes
CREATE INDEX idx_other_appnm_entdt ON dbo_tblotherappdtls (AppNm, EntDt);
CREATE INDEX idx_other_entryno_accept_appsrno ON dbo_tblotherappdtls (EntryNo, AppAccept, AppSrNo);
CREATE INDEX idx_amttrans_refno ON dbo_amttrans (RefNo);
CREATE INDEX idx_amttrans_currdate ON dbo_amttrans (CurrDate);
CREATE INDEX idx_billchild_billsrno ON dbo_billchild (BillSrNo);
CREATE INDEX idx_uploaddocs_srno ON dbo_tbluploaddocs (SrNo);
```

---

### Priority 3 — Medium (Run within the week)

| # | Table | Rows | Index Name | Columns | Queries That Benefit | Expected Speedup |
|---|---|---|---|---|---|---|
| 14 | dbo_tbltxns | 155,627 | idx_txns_orderid | order_id | Payment status lookups. 155K rows, only primary key. | Payment checks faster |
| 15 | dbo_tblscrutiny | 97,296 | idx_scrutiny_srno | SrNo | Scrutiny data lookups by application SrNo. 97K rows. | Scrutiny pages faster |
| 16 | dbo_tblinoutregister | 294,502 | idx_inout_entdt | EntDt | In/out register filtered by date. 294K rows. | In/out reports faster |
| 17 | dbo_tblapplicationdtls_profile | 35,088 | idx_profile_srno | SrNo | Profile pic lookups by application. | Profile loading faster |
| 18 | dbo_addrchange | 12,568 | idx_addrchange_srno | SrNo | Dashboard loads address change apps by SrNo. | Minor improvement |
| 19 | pro_to_original | 14,064 | idx_pro_to_original_srno | SrNo | Pro-to-original list lookups. | Minor improvement |
| 20 | dbo_tblelgblappl | 416 | idx_eligible_userid | user_id | Dashboard eligible check. Small table but queried every dashboard load. | Minor improvement |
| 21 | refresher_exam_user_answers | 927,644 | idx_exam_userid | user_id | Exam answer lookups per user. Largest table by rows. | Exam result loading faster |

**SQL to run:**

```sql
-- Priority 3: Medium impact indexes
CREATE INDEX idx_txns_orderid ON dbo_tbltxns (order_id);
CREATE INDEX idx_scrutiny_srno ON dbo_tblscrutiny (SrNo);
CREATE INDEX idx_inout_entdt ON dbo_tblinoutregister (EntDt);
CREATE INDEX idx_profile_srno ON dbo_tblapplicationdtls_profile (SrNo);
CREATE INDEX idx_addrchange_srno ON dbo_addrchange (SrNo);
CREATE INDEX idx_pro_to_original_srno ON pro_to_original (SrNo);
CREATE INDEX idx_eligible_userid ON dbo_tblelgblappl (user_id);
CREATE INDEX idx_exam_userid ON refresher_exam_user_answers (user_id);
```

---

## Benchmark Queries

Run these BEFORE and AFTER adding indexes to measure the improvement.

### Before/After Priority 1 indexes:

```sql
SET @uid = 12;
SET @srno = 202502402;

-- Test A: Dashboard applicant lookup (expect 0.7s → 0.005s)
SELECT * FROM dbo_tblapplicationdtls WHERE user_id = @uid ORDER BY created_at DESC LIMIT 1;

-- Test B: Territory query (expect 0.65s → 0.005s)
SELECT * FROM dbo_tblapplicationdtls WHERE user_id = @uid AND applicationType = 'transfer-territory' AND appStatus != 'Initial';

-- Test C: Download lookup (expect 0.2s → 0.001s)
SELECT * FROM dbo_tbldownloads WHERE SrNo = @srno AND type = 'certificate' AND sub_type = 'provisional' ORDER BY created_at DESC LIMIT 1;

-- Test D: Slot lookup (expect 0.05s → 0.001s)
SELECT * FROM slot_details WHERE user_id = @uid AND status = 1 LIMIT 1;

-- Test E: Bill receipt lookup (expect 0.15s → 0.001s)
SELECT BillSrNo FROM dbo_billhead WHERE EntryNo = @srno AND AppSrNo = '1' LIMIT 1;

-- Test F: EXPLAIN (should show type:ref instead of type:ALL after index)
EXPLAIN SELECT * FROM dbo_tblapplicationdtls WHERE user_id = @uid ORDER BY created_at DESC LIMIT 1;
```

### Before/After Priority 2 indexes:

```sql
-- Test G: Other app lookup (expect 0.09s → 0.001s)
SELECT * FROM dbo_tblotherappdtls WHERE EntryNo = @srno AND AppAccept = 4 AND AppSrNo = '11' ORDER BY RNo DESC LIMIT 1;

-- Test H: Amttrans by RefNo (expect slow → fast on 920K rows)
SELECT * FROM dbo_amttrans WHERE RefNo = '12345' LIMIT 1;

-- Test I: Billchild join column (expect slow → fast on 728K rows)
SELECT COUNT(*) FROM dbo_billchild WHERE BillSrNo = 100;

-- Test J: Upload docs (expect slow → fast on 474K rows)
SELECT * FROM dbo_tbluploaddocs WHERE SrNo = @srno;
```

---

## Summary

| Priority | Indexes | Tables Affected | Total Rows Covered | Impact |
|---|---|---|---|---|
| Priority 1 (Critical) | 7 indexes | 4 tables | 597K rows | Fixes user dashboard (20-25 queries), public dashboard, receipt lookups |
| Priority 2 (High) | 6 indexes | 4 tables | 2.58M rows | Fixes reports, financial reports, document loading |
| Priority 3 (Medium) | 8 indexes | 8 tables | 1.4M rows | Fixes scrutiny, in/out, profile, exam lookups |
| **Total** | **21 indexes** | **16 tables** | **4.58M rows** | |

**Estimated index creation time:** Under 30 seconds per index (tables are in memory). Total ~10 minutes for all 21 indexes.

**Estimated disk space for indexes:** ~200-300 MB additional (small compared to 46 GB buffer pool).

**Risk:** Zero for SELECT performance. CREATE INDEX briefly locks the table (few seconds per table). Run during low-traffic hours if concerned.
