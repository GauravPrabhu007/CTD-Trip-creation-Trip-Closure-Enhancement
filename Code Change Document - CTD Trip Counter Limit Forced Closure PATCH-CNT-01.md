# Code Change Document — CTD Trip Counter Limit Forced Closure (PATCH-CNT-01)

**Final — For ABAP Implementation**

---

## Document Information

| Field | Detail |
|---|---|
| **Document Title** | CTD Trip Counter Limit Forced Closure — PATCH-CNT-01 |
| **Change ID** | PATCH-CNT-01 / PATCH-CNT-02 |
| **System** | RD2 |
| **Function Groups** | `ZLOG_CTD` |
| **Prepared For** | SAP ABAP Technical Team |
| **Date** | June 2026 |
| **Status** | **Final — Ready for Implementation** |
| **Patch Format** | `-` = Remove · `+` = Add |
| **Supersedes** | PATCH-DICT-01 (`ZCOUNTER` NUMC 2 → 4) in `Code Change Document - CTD Trip Production Bug Fixes.md` — **not required** if this change is deployed |
| **Related Documents** | PATCH-CLS-01/02/03 (area-based trip closure at loading confirmation) |
| **Azure Story** | Counter overflow / shipment overwrite on `ZSCE_CTD_ITM` |

---

## 0. ABAP Team — Quick Start

| Item | Detail |
|---|---|
| **Problem** | After counter `90`, `ADD 10` → `100` overflows `ZCOUNTER` NUMC(2) → ITM `MODIFY` overwrites existing shipment |
| **Fix** | Force-close open trip when `MAX( counter ) >= 90` **before** 10th leg; new trip + counter `10` |
| **Dictionary change** | **None** — `ZCOUNTER` stays NUMC(2) |
| **Objects to change** | `Z_SCE_TRIP_STLMNT` (primary), `Z_SCE_TRIP_COUNTER` (safety net) |
| **Do NOT use** | `Z_SCE_TRIP_COMPLETE` for counter closure (no area match at mid-route) |
| **Additive with** | Area-based closure (TPN + loading confirmation) — both may apply |

---

## 1. Background

### 1.1 Problem

| Step | Counter on trip | Issue |
|---|---|---|
| Leg 1 … 9 | `10, 20, … 90` | OK |
| Leg 10 | `90 + 10 = 100` | `ZCOUNTER` is **NUMC(2)** — value `100` overflows/truncates |
| Persist | `MODIFY zsce_ctd_itm` key includes `counter` | **Overwrites** existing ITM row (blank/duplicate counter) |

**Root cause FM:** `Z_SCE_TRIP_COUNTER` — `SELECT MAX( counter )` then `ADD 10 TO e_counter`.

### 1.2 Agreed solution (functional sign-off)

| Rule | Confirmation |
|---|---|
| Max legs per trip | **9** — users confirmed trips shall not exceed 9 legs |
| Threshold | Force-close when **`MAX( ZSCE_CTD_ITM-COUNTER ) >= 90`** before assigning **10th** leg |
| Dictionary extension | **Not required** — avoid `ZCOUNTER` length change |
| vs area closure | **Additional** — counter closure is independent of base-area closure |
| `ZTRSTLMNT-COUNTER` | **Out of scope** — separate from CTD ITM counter logic |

### 1.3 Alternative not taken

| Approach | Status |
|---|---|
| PATCH-DICT-01: extend `ZCOUNTER` to NUMC(4) | **Superseded** by this document per user decision |

---

## 2. Business Rules

### 2.1 Two independent closure triggers

| Trigger | When | Mechanism | Area match? |
|---|---|---|---|
| **Area-based** | Truck at **base** (YTTS / `ZTPPAREA`) | `Z_SCE_TRIP_COMPLETE` in `Z_LOG_TRSTLMNT_UPD_TD` | **Yes** |
| **Counter-based** | Open trip already has **9 legs** (`MAX >= 90`) | Force-close HDR → `03` in `Z_SCE_TRIP_STLMNT` | **No** |

Both may apply at the same loading confirmation in sequence:

```
1. Area-based closure (if applicable)
2. Z_SCE_TRIP_STLMNT
   a. Counter-based forced closure (if MAX >= 90)
   b. Z_SCE_TRIP_COUNTER
   c. Z_SCE_TRIP_UPD
```

### 2.2 Counter sequence (unchanged increment logic)

| Leg # | Counter assigned |
|---|---|
| 1 | 10 |
| 2 | 20 |
| 3 | 30 |
| 4 | 40 |
| 5 | 50 |
| 6 | 60 |
| 7 | 70 |
| 8 | 80 |
| 9 | 90 |
| **10** | **New trip** → **10** (after forced close of prior trip) |

### 2.3 Forced closure rule

```
IF open trip (status 01 or 02)
   AND MAX( ZSCE_CTD_ITM-COUNTER ) >= 90 for that trip
THEN
   UPDATE ZSCE_CTD_HDR SET trip_status = '03'
   Start new trip: status '01', new trip_no, counter = 10 for current shipment
ENDIF
```

> **Note:** Leg 9 still receives counter `90` on the **current** trip. Forced close runs when adding leg **10**.

---

## 3. Affected Objects

| Object | Change | Mandatory |
|---|---|---|
| `Z_SCE_TRIP_STLMNT` | PATCH-CNT-01 — counter limit check + force close | **Yes** |
| `Z_SCE_TRIP_COUNTER` | PATCH-CNT-02 — safety net, no `ADD 10` when `>= 90` | **Yes** |
| `Z_SCE_TRIP_UPD` | No change | — |
| `Z_SCE_TRIP_COMPLETE` | No change | — |
| `Z_LOG_TRSTLMNT_UPD_TD` | No change (area closure already present) | — |
| `ZCOUNTER` domain | **No change** | — |
| `ZSCE_CTD_ITM` / `ZTRSTLMNT` | **No dictionary change** | — |

---

## 4. Implementation — PATCH-CNT-01 (`Z_SCE_TRIP_STLMNT`)

**FG:** `ZLOG_CTD`  
**Location:** Inside `LOOP AT it_trstlmnt`, **after** trip status / `trip_no` assignment and HDR area from master, **before** `APPEND lw_trip_hdr TO lt_trip_hdr`.

### 4.1 Constants and data (add to FM DATA section)

```diff
+  CONSTANTS: lc_ctd_cnt_limit TYPE zcounter VALUE '90'.
+
+  DATA: lv_max_counter TYPE zcounter,
+        lv_old_trip_no   TYPE ztrip_no.
```

### 4.2 Counter limit check and forced trip rotation

Insert **after** HDR area block (`" EOC CD:xxxx"` for master area) and **before** `APPEND lw_trip_hdr TO lt_trip_hdr`:

```diff
       " EOC CD:xxxx

+      " BOC CD:xxxx - Forced trip close when counter limit reached (max 9 legs)
+      IF lw_trip_hdr-trip_status = '02'.
+        CLEAR lv_max_counter.
+        SELECT MAX( counter )
+          FROM zsce_ctd_itm CLIENT SPECIFIED
+          INTO lv_max_counter
+          WHERE mandt    = sy-mandt
+            AND lifnr    = lw_trstlmnt-lifnr
+            AND truck_no = lw_trstlmnt-vehl_no(15)
+            AND trip_no  = lw_trip_hdr-trip_no.
+        IF sy-subrc = 0 AND lv_max_counter >= lc_ctd_cnt_limit.
+
+          lv_old_trip_no = lw_trip_hdr-trip_no.
+
+          CALL FUNCTION 'ENQUEUE_EZSCE_CTD_HDR'
+            EXPORTING
+              mode_zsce_ctd_hdr = 'E'
+              mandt             = sy-mandt
+              lifnr             = lw_trstlmnt-lifnr
+              truck_no          = lw_trstlmnt-vehl_no(15)
+              trip_no           = lv_old_trip_no
+            EXCEPTIONS
+              OTHERS            = 0.
+
+          UPDATE zsce_ctd_hdr CLIENT SPECIFIED
+            SET trip_status   = '03'
+                modified_by   = sy-uname
+                modified_date = sy-datum
+                modified_time = sy-uzeit
+            WHERE mandt    = sy-mandt
+              AND lifnr    = lw_trstlmnt-lifnr
+              AND truck_no = lw_trstlmnt-vehl_no(15)
+              AND trip_no  = lv_old_trip_no
+              AND trip_status IN ('01', '02').
+
+          CALL FUNCTION 'DEQUEUE_EZSCE_CTD_HDR'
+            EXPORTING
+              mode_zsce_ctd_hdr = 'E'
+              lifnr             = lw_trstlmnt-lifnr
+              truck_no          = lw_trstlmnt-vehl_no(15)
+              trip_no           = lv_old_trip_no
+            EXCEPTIONS
+              OTHERS            = 0.
+
+          CONCATENATE sy-datum sy-uzeit INTO lw_trip_no.
+          lw_trip_hdr-trip_no     = lw_trip_no.
+          lw_trip_hdr-trip_status = '01'.
+
+        ENDIF.
+      ENDIF.
+      " EOC CD:xxxx - Forced trip close when counter limit reached

       APPEND lw_trip_hdr TO lt_trip_hdr.
```

### 4.3 Behaviour after patch

| Prior trip state | `MAX( counter )` | Action on current loading |
|---|---|---|
| Open (`02`) | `< 90` | Same trip (`02`), counter `MAX + 10` |
| Open (`02`) | `>= 90` | Old trip → `03`; new trip (`01`), new `trip_no`, counter `10` |
| New (`01`) | n/a | Counter `10` (existing counter FM logic) |
| Closed by area closure before STLMNT | `03` | New trip (`01`) — existing logic |

---

## 5. Implementation — PATCH-CNT-02 (`Z_SCE_TRIP_COUNTER`)

**FG:** `ZLOG_CTD`  
**Location:** After `SELECT MAX( counter )` / before `ADD 10 TO e_counter`.

### 5.1 Code delta

```diff
+  CONSTANTS: lc_ctd_cnt_limit TYPE zcounter VALUE '90'.

   SELECT MAX( counter ) INTO e_counter FROM zsce_ctd_itm ...
   IF sy-subrc EQ 0.
+    IF e_counter >= lc_ctd_cnt_limit.
+      lw_return-type    = 'E'.
+      lw_return-id      = 'ZLOG'.
+      lw_return-message = 'Trip counter limit reached; trip must be rotated'(xxx).
+      APPEND lw_return TO et_return.
+      RETURN.
+    ENDIF.
     ADD 10 TO e_counter.
   ENDIF.
```

> Replace message `(xxx)` with actual text element before transport.  
> **Primary** rotation is in PATCH-CNT-01; this block is a **safety net** against overflow if STLMNT rotation is bypassed.

---

## 6. Execution Flow (combined with area closure)

```
Loading Confirmation (OIGSV status = '2')
        │
        ▼
Z_LOG_TRSTLMNT_UPD_TD
  → Area-based: Z_SCE_TRIP_COMPLETE (if area = HDR base)
        │
        ▼
Z_SCE_TRIP_STLMNT
  → Determine trip status (01 new / 02 open)
  → [NEW] IF status 02 AND MAX(counter) >= 90
        → Force close old trip (03)
        → New trip (01), new trip_no
  → Z_SCE_TRIP_COUNTER → 10 (new) or MAX+10 (open, MAX < 90)
  → Z_SCE_TRIP_UPD → MODIFY ZSCE_CTD_HDR / ZSCE_CTD_ITM
```

---

## 7. Transport Object List

| Seq | Object | FG | Patch | Mandatory |
|---|---|---|---|---|
| 1 | `Z_SCE_TRIP_STLMNT` | `ZLOG_CTD` | PATCH-CNT-01 | Yes |
| 2 | `Z_SCE_TRIP_COUNTER` | `ZLOG_CTD` | PATCH-CNT-02 | Yes |
| 3 | `Z_SCE_TRIP_UPD` | `ZLOG_CTD` | — | No change |
| 4 | `Z_SCE_TRIP_COMPLETE` | `ZLOG_CTD` | — | No change |
| 5 | `Z_LOG_TRSTLMNT_UPD_TD` | `ZLOG_TD` | — | No change |
| 6 | `ZCOUNTER` domain | SE11 | — | **No change** |

---

## 8. Deployment Sequence

```
Step 1 — ABAP: Z_SCE_TRIP_STLMNT       (PATCH-CNT-01)
Step 2 — ABAP: Z_SCE_TRIP_COUNTER      (PATCH-CNT-02)
Step 3 — Transport to QA
Step 4 — Execute UAT C01–C07
Step 5 — Transport to PRD
```

> Replace `CD:xxxx` and message number `(xxx)` before transport.

---

## 9. UAT Test Plan

| ID | Scenario | Setup | Expected result |
|---|---|---|---|
| **C01** | 9th leg on open trip | 8 ITM rows (max counter `80`); add 9th loading | New ITM counter = `90`; trip status `02`; **no** forced close |
| **C02** | 10th leg — counter forced close | Open trip with max counter `90`; 10th loading (any location) | Prior trip `03`; new trip `01`; new ITM counter = `10`; **no overwrite** of leg 9 row |
| **C03** | 10th leg after area closure | Trip closed at base on 7th leg; 8th loading | New trip from area closure path; counter = `10`; C02 path not needed |
| **C04** | Mid-route 10th leg | 9 legs, truck not at base | Forced close without area match; new trip created |
| **C05** | Never reaches 90 | Trip with ≤ 8 legs | Normal `MAX + 10`; no forced close |
| **C06** | Counter FM safety net | Simulate STLMNT skip (negative test) | `Z_SCE_TRIP_COUNTER` returns error; no counter `100` / no overwrite |
| **C07** | Port + counter combined | Port truck: 9 legs then reload at port | Area closure on reload if `AP` match; or counter close on 10th leg if still open |

### 9.1 Post-test SQL

```sql
-- Counters on a trip — should never exceed 90
SELECT lifnr, truck_no, trip_no, counter, shnumber
  FROM zsce_ctd_itm
 WHERE truck_no = '<test_truck>'
 ORDER BY trip_no, counter.

-- Trip headers — expect new trip_no after 10th leg
SELECT lifnr, truck_no, trip_no, trip_status, area
  FROM zsce_ctd_hdr
 WHERE truck_no = '<test_truck>'
 ORDER BY trip_no DESC.
```

---

## 10. Risks & Mitigations

| # | Risk | Mitigation |
|---|---|---|
| R1 | Mid-route trip split affects costing | User confirmed max 9 legs; accept new trip ID at leg 10 |
| R2 | Area + counter both close same loading | Area closure first → new trip; counter check on **new** trip (MAX < 90) |
| R3 | Enqueue failure on force close | Use same enqueue pattern as `Z_SCE_TRIP_UPD`; log error if UPDATE fails |
| R4 | Counter FM still reaches 100 | PATCH-CNT-02 safety net |
| R5 | PATCH-DICT-01 also deployed | **Do not deploy both** — choose counter guard OR domain extension |

---

## 11. Out of Scope

- `ZCOUNTER` domain extension (PATCH-DICT-01)
- `ZTRSTLMNT` counter logic
- Changes to area-based closure (PATCH-CLS)
- Changes to `Z_SCE_TRIP_COMPLETE`

---

## 12. Sign-off

| Role | Name | Date |
|---|---|---|
| Functional Lead | | |
| SAP ABAP Developer | | |
| QA Lead | | |

---

*End of Document — PATCH-CNT-01 / PATCH-CNT-02*
