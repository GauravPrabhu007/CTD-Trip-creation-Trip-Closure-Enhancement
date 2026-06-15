# Code Change Document — CTD Port Plant ITM Area & Trip Closure Delta

---

## Document Information

| Field | Detail |
|---|---|
| **Document Title** | Code Change Document — CTD Port Plant ITM Area & Trip Closure Delta |
| **System** | RD2 |
| **Function Groups** | `ZLOG_CTD`, `ZLOG_TD` |
| **Prepared For** | SAP ABAP Technical Team |
| **Date** | June 2026 |
| **Status** | Ready for Development |
| **Patch Format** | `-` = Remove · `+` = Add · `#` = Comment only |
| **Related Document** | `Code Change Document - CTD Trip Production Bug Fixes.md` (separate scope; HDR-from-master may already be applied) |

---

## 1. Background

### 1.1 Business context — Adani Port (`AP`)

Port-based trucks (e.g. Adani Port) operate **without YTTS** in the dispatch cycle. CTD Master is maintained with **Area `AP`** as the truck base location.

| Table / field | Source | Example |
|---|---|---|
| `ZSCE_CTD_HDR-AREA` | `ZSCE_CTD_MST-AREA` | `AP` |
| `ZSCE_CTD_ITM-AREA` | YTTS → **`ZTPPAREA` fallback** (new) | Outbound: `JG` · Return load at port: `AP` |

### 1.2 Problems addressed

| # | Problem | Root cause |
|---|---------|------------|
| **P1** | `ZSCE_CTD_ITM-AREA` blank for port plant legs | ITM area derived only from `YTTSTX0001`; no YTTS at port |
| **P2** | Open trips never close for port trucks | `Z_SCE_TRIP_COMPLETE` called only from YTTS-TPN (`MYSTXF01`); no TPN at port plant |
| **P3** | Empty return to port leaves trip open | No shipment / no YTTS event at empty gate-in; closure must occur on **next loading** |

### 1.3 Trip closure model (port + plant)

| Scenario | Primary closure path | Loading-confirmation check (new) |
|---|---|---|
| **Plant base truck** | YTTS-TPN (`UPDATE_SHNUMBER_TPN_R`) | Extra safety net — close open trip if current shipment area matches HDR area |
| **Port truck (`AP`)** | Not available (no YTTS TPN) | **Only** closure path — runs at loading confirmation when area matches |
| **Empty return to port** | Nothing runs | Next port loading triggers area check → close if match |
| **No open trip** | — | `Z_SCE_TRIP_COMPLETE` no-op → `Z_SCE_TRIP_STLMNT` creates new trip (unchanged) |

> **Business confirmation:** Same-base multi-loading on one open trip does **not** occur. Plant safety-net closure at loading is therefore safe.

---

## 2. Area Derivation Rules (after change)

### 2.1 HDR vs ITM — different sources (by design)

| Field | Priority | Notes |
|---|---|---|
| **`ZSCE_CTD_HDR-AREA`** | `ZSCE_CTD_MST-AREA` | Base area (`AP` for Adani Port trucks). Set on new trip (`01`) only via existing `Z_SCE_TRIP_UPD` logic. |
| **`ZSCE_CTD_ITM-AREA`** | 1. `YTTSTX0001-AREA` · 2. `ZTPPAREA` via `OIGS-TPLST` · 3. blank | Leg / location area for the shipment line. |

### 2.2 `ZTPPAREA` configuration (functional prerequisite)

Maintain via T-code **`ZTPPAREA`**:

| TPLST | AREA | Purpose |
|---|---|---|
| `ABC5` | `AP` | Adani Port base — return loading & trip closure |
| `JGC5` | `JG` | Outbound leg to Jamnagar (example) |
| `PGC5` | `PG` | Outbound leg to Patalganga (example) |
| … | … | Existing port-to-plant mappings |

> On **outbound** loading, `OIGS-TPLST` = destination TPLST (e.g. `JGC5`) → ITM area `JG` ≠ HDR `AP` → trip stays open.  
> On **return/reload** at port, `OIGS-TPLST` = port TPLST (e.g. `ABC5`) → area `AP` = HDR `AP` → trip closes.

### 2.3 Trip closure area (loading confirmation)

Same derivation as ITM for the current shipment:

```
lw_closure_area = YTTSTX0001-AREA          (if YTTS linked)
               OR ZTPPAREA-AREA for OIGS-TPLST   (if YTTS area blank)
```

`Z_SCE_TRIP_COMPLETE` closes only when `lw_closure_area = ZSCE_CTD_HDR-AREA` and trip status is `01` or `02`.

---

## 3. Affected Objects

| FM | FG | Role | Change |
|---|---|---|---|
| `Z_SCE_TRIP_STLMNT` | `ZLOG_CTD` | Trip settlement — builds hdr/itm | **PATCH-ITM-01/02/03** — ITM area `ZTPPAREA` fallback |
| `Z_LOG_TRSTLMNT_UPD_TD` | `ZLOG_TD` | Loading confirmation caller | **PATCH-CLS-01/02** — trip complete before STLMNT |
| `Z_SCE_TRIP_COMPLETE` | `ZLOG_CTD` | Closes open trip | **No change** |
| `Z_SCE_TRIP_UPD` | `ZLOG_CTD` | Persists hdr/itm | **No change** |
| `MYSTXF01` | — | YTTS-TPN closure | **No change** (remains primary for plant) |

| Config / table | Role |
|---|---|
| `ZTPPAREA` | TPLST → Area (incl. `ABC5 → AP`) |
| `ZSCE_CTD_MST` | Truck base area (`AP`) |
| `ZLOG_EXEC_VAR` | `ZSCE_CTD_TPN_ACTIVATE` gates closure call |
| `OIGS` | `TPLST` for `ZTPPAREA` lookup |
| `YTTSTX0001` | YTTS area (plant trucks) |

---

## 4. Execution Flow (after change)

```
Loading Confirmation (OIGSV status = '2')
        │
        ▼
Z_LOG_TRSTLMNT_UPD_TD
  → builds lt_trstlmnt
  → [NEW] PATCH-CLS: derive lw_closure_area (YTTS → ZTPPAREA)
  → [NEW] CALL Z_SCE_TRIP_COMPLETE (if ZSCE_CTD_TPN_ACTIVATE active)
  → CALL Z_SCE_TRIP_STLMNT
        │
        ▼
Z_SCE_TRIP_STLMNT
  → HDR area from ZSCE_CTD_MST (existing)
  → ITM area: YTTSTX0001 → [NEW] ZTPPAREA fallback
  → CALL Z_SCE_TRIP_COUNTER → Z_SCE_TRIP_UPD
        │
        ▼
ZSCE_CTD_HDR / ZSCE_CTD_ITM updated
```

**Parallel path (unchanged) — plant trucks:**

```
YTTS Receive + TPN (MYSTXF01 / UPDATE_SHNUMBER_TPN_R)
  → Z_SCE_TRIP_COMPLETE (if ZSCE_CTD_TPN_ACTIVATE active)
```

---

## 5. Code Delta Patches

---

### 5.1 ITM Area — `ZTPPAREA` fallback in `Z_SCE_TRIP_STLMNT`

**Object:** `Z_SCE_TRIP_STLMNT` | **FG:** `ZLOG_CTD`

#### PATCH-ITM-01 — Add local types and data (top of FM, with existing DATA block)

```diff
+  TYPES: BEGIN OF lty_oigs_tplst,
+           shnumber TYPE oig_shnum,
+           tplst    TYPE tplst,
+         END OF lty_oigs_tplst.
+
+  DATA: lt_oigs_tplst TYPE STANDARD TABLE OF lty_oigs_tplst,
+        lw_oigs_tplst TYPE lty_oigs_tplst,
+        lt_ztpparea   TYPE STANDARD TABLE OF ztpparea,
+        lw_ztpparea   TYPE ztpparea.
```

#### PATCH-ITM-02 — Bulk-fetch `OIGS-TPLST` and `ZTPPAREA` before main loop

**Location:** After `lw_cnt = lines( lt_yttstx0001 )` and before `LOOP AT it_trstlmnt`.

```diff
+  " BOC CD:xxxx - Bulk read OIGS-TPLST and ZTPPAREA for ITM area fallback
+  REFRESH: lt_oigs_tplst, lt_ztpparea.
+  IF it_trstlmnt IS NOT INITIAL.
+    SELECT shnumber tplst
+      FROM oigs CLIENT SPECIFIED
+      INTO TABLE lt_oigs_tplst
+      FOR ALL ENTRIES IN it_trstlmnt
+      WHERE mandt    = sy-mandt
+        AND shnumber = it_trstlmnt-tknum.
+    IF sy-subrc = 0.
+      SORT lt_oigs_tplst BY shnumber.
+      DELETE ADJACENT DUPLICATES FROM lt_oigs_tplst COMPARING shnumber.
+      SELECT tplst area
+        FROM ztpparea CLIENT SPECIFIED
+        INTO TABLE lt_ztpparea
+        FOR ALL ENTRIES IN lt_oigs_tplst
+        WHERE mandt = sy-mandt
+          AND tplst = lt_oigs_tplst-tplst.
+      IF sy-subrc = 0.
+        SORT lt_ztpparea BY tplst.
+      ENDIF.
+    ENDIF.
+  ENDIF.
+  " EOC CD:xxxx
```

#### PATCH-ITM-03 — `ZTPPAREA` fallback after YTTS ITM area block

**Location:** Immediately after the existing YTTS block (`"end on 20.02.2020"`), before `lw_trip_itm-lifnr = ...`.

```diff
       ENDIF.
       "end on 20.02.2020
+
+      " BOC CD:xxxx - ITM area fallback from ZTPPAREA when no YTTS
+      IF lw_trip_itm-area IS INITIAL.
+        READ TABLE lt_oigs_tplst INTO lw_oigs_tplst
+          WITH KEY shnumber = lw_trstlmnt-tknum
+          BINARY SEARCH.
+        IF sy-subrc = 0.
+          READ TABLE lt_ztpparea INTO lw_ztpparea
+            WITH KEY tplst = lw_oigs_tplst-tplst
+            BINARY SEARCH.
+          IF sy-subrc = 0.
+            lw_trip_itm-area = lw_ztpparea-area.
+          ENDIF.
+        ENDIF.
+      ENDIF.
+      " EOC CD:xxxx
+
       lw_trip_itm-lifnr = lw_trstlmnt-lifnr.
```

**ITM area priority after PATCH-ITM-03:**

| Priority | Source |
|---|---|
| 1 | `YTTSTX0001-AREA` (existing logic) |
| 2 | `ZTPPAREA-AREA` via `OIGS-TPLST` |
| 3 | blank (no error raised) |

---

### 5.2 Trip Closure — loading confirmation check in `Z_LOG_TRSTLMNT_UPD_TD`

**Object:** `Z_LOG_TRSTLMNT_UPD_TD` | **FG:** `ZLOG_TD`

#### PATCH-CLS-01 — Add local types and data (with existing DATA block near line ~210)

```diff
+  TYPES: BEGIN OF lty_yttstx0001_cls,
+           area       TYPE yarea,
+           truck_no   TYPE ytruck_no,
+           shnumber   TYPE tknum,
+           trk_purpos TYPE ytrk_purps,
+         END OF lty_yttstx0001_cls.
+
+  DATA: lt_yttstx0001_cls TYPE STANDARD TABLE OF lty_yttstx0001_cls,
+        lw_yttstx0001_cls TYPE lty_yttstx0001_cls,
+        lt_trstlmnt_cls   TYPE STANDARD TABLE OF ztrstlmnt,
+        lw_trstlmnt_cls   TYPE ztrstlmnt,
+        lw_closure_area   TYPE yarea,
+        lw_trip_complete  TYPE bapiret2,
+        lt_trip_complete  TYPE bapiret2_t,
+        lw_ytts_cnt       TYPE i,
+        lw_tpn_active     TYPE zactive_flag.
+
+  CONSTANTS: lc_ctd_tpn TYPE rvari_vnam VALUE 'ZSCE_CTD_TPN_ACTIVATE'.
```

#### PATCH-CLS-02 — Call `Z_SCE_TRIP_COMPLETE` before `Z_SCE_TRIP_STLMNT`

**Location:** Inside block `lw_oigsv_2-oig_sstsf = '2'`, immediately **before** `CALL FUNCTION 'Z_SCE_TRIP_STLMNT'` (~line 1365).

```diff
              "EOC by Ketan Bhor on 07.10.2021 17:43:58 for CD: 8056115
+
+              " BOC CD:xxxx - CTD trip complete check at loading confirmation
+              CLEAR lw_tpn_active.
+              SELECT SINGLE active
+                FROM zlog_exec_var CLIENT SPECIFIED
+                INTO lw_tpn_active
+                WHERE mandt  = sy-mandt
+                  AND name   = lc_ctd_tpn
+                  AND active = abap_true.
+              IF sy-subrc = 0 AND lt_trstlmnt IS NOT INITIAL.
+
+                lt_trstlmnt_cls[] = lt_trstlmnt[].
+                SORT lt_trstlmnt_cls BY lifnr vehl_no tknum.
+                DELETE ADJACENT DUPLICATES FROM lt_trstlmnt_cls
+                  COMPARING lifnr vehl_no.
+
+                REFRESH lt_yttstx0001_cls.
+                SELECT area truck_no shnumber trk_purpos
+                  FROM yttstx0001 CLIENT SPECIFIED
+                  INTO TABLE lt_yttstx0001_cls
+                  FOR ALL ENTRIES IN lt_trstlmnt_cls
+                  WHERE mandt     = sy-mandt
+                    AND truck_no  = lt_trstlmnt_cls-vehl_no(15)
+                    AND shnumber  = lt_trstlmnt_cls-tknum.
+                IF sy-subrc = 0.
+                  SORT lt_yttstx0001_cls BY truck_no shnumber trk_purpos.
+                ENDIF.
+                lw_ytts_cnt = lines( lt_yttstx0001_cls ).
+
+                LOOP AT lt_trstlmnt_cls INTO lw_trstlmnt_cls.
+                  CLEAR lw_closure_area.
+
+                  " Priority 1: YTTS area (same logic as Z_SCE_TRIP_STLMNT)
+                  IF lw_ytts_cnt = 1.
+                    READ TABLE lt_yttstx0001_cls INTO lw_yttstx0001_cls INDEX 1.
+                    IF sy-subrc = 0.
+                      lw_closure_area = lw_yttstx0001_cls-area.
+                    ENDIF.
+                  ELSE.
+                    READ TABLE lt_yttstx0001_cls INTO lw_yttstx0001_cls
+                      WITH KEY truck_no   = lw_trstlmnt_cls-vehl_no(15)
+                               shnumber   = lw_trstlmnt_cls-tknum
+                               trk_purpos = 'R'.
+                    IF sy-subrc = 0.
+                      lw_closure_area = lw_yttstx0001_cls-area.
+                    ENDIF.
+                  ENDIF.
+
+                  " Priority 2: ZTPPAREA via OIGS-TPLST (port / no-YTTS)
+                  IF lw_closure_area IS INITIAL
+                     AND pi_oigs-tplst IS NOT INITIAL.
+                    SELECT SINGLE area
+                      FROM ztpparea CLIENT SPECIFIED
+                      INTO lw_closure_area
+                      WHERE mandt = sy-mandt
+                        AND tplst = pi_oigs-tplst.
+                  ENDIF.
+
+                  IF lw_closure_area IS NOT INITIAL.
+                    CLEAR lt_trip_complete.
+                    CALL FUNCTION 'Z_SCE_TRIP_COMPLETE'
+                      EXPORTING
+                        i_vendor   = lw_trstlmnt_cls-lifnr
+                        i_truck_no = lw_trstlmnt_cls-vehl_no(15)
+                        i_area     = lw_closure_area
+                      IMPORTING
+                        et_return  = lt_trip_complete.
+
+                    " Ignore 'No data found' — no open trip is valid
+                    LOOP AT lt_trip_complete INTO lw_trip_complete
+                      WHERE type = 'E'.
+                      IF lw_trip_complete-message CS 'No data found'.
+                        CONTINUE.
+                      ENDIF.
+                      APPEND lw_trip_complete TO et_return.
+                    ENDLOOP.
+                  ENDIF.
+                ENDLOOP.
+              ENDIF.
+              " EOC CD:xxxx
+
 *  BOC by RIGV_018 on 03.05.2019
               CALL FUNCTION 'Z_SCE_TRIP_STLMNT'
```

> **Note:** Replace the `CS 'No data found'` check with the exact message no. / text element from `Z_SCE_TRIP_COMPLETE` message class during development (text symbol `001`).

---

## 6. Consolidated Patch Map

| Patch | Object | Description |
|---|---|---|
| PATCH-ITM-01 | `Z_SCE_TRIP_STLMNT` | Types/data for OIGS + ZTPPAREA tables |
| PATCH-ITM-02 | `Z_SCE_TRIP_STLMNT` | Bulk SELECT before main loop |
| PATCH-ITM-03 | `Z_SCE_TRIP_STLMNT` | ITM area ZTPPAREA fallback after YTTS block |
| PATCH-CLS-01 | `Z_LOG_TRSTLMNT_UPD_TD` | Types/data for closure check |
| PATCH-CLS-02 | `Z_LOG_TRSTLMNT_UPD_TD` | `Z_SCE_TRIP_COMPLETE` before STLMNT |

**No FM changes:** `Z_SCE_TRIP_COMPLETE`, `Z_SCE_TRIP_UPD`, `Z_SCE_TRIP_COUNTER`, `MYSTXF01`.

---

## 7. Configuration Prerequisites (Functional / Basis)

| Seq | Activity | Owner | Detail |
|---|---|---|---|
| C1 | CTD Master | Functional | Maintain `ZSCE_CTD_MST-AREA = AP` for Adani Port trucks |
| C2 | `ZTPPAREA` | Functional | Add `ABC5 → AP` (port base TPLST); confirm outbound TPLST mappings |
| C3 | `ZLOG_EXEC_VAR` | Basis | Confirm `ZSCE_CTD_TPN_ACTIVATE` is active (already `X` on RD2) |
| C4 | `OIGS-TPLST` | Functional / QA | Validate TPLST on port return loading = `ABC5` (not destination TPLST) |

---

## 8. Transport Object List

| Seq | Object | Type | Patch Ref | Mandatory |
|---|---|---|---|---|
| 1 | `ZTPPAREA` | Table data (SM30 / ZTPPAREA) | Config C2 | Yes |
| 2 | `ZSCE_CTD_MST` | Table data | Config C1 | Yes |
| 3 | `Z_SCE_TRIP_STLMNT` | FM / `ZLOG_CTD` | PATCH-ITM-01/02/03 | Yes |
| 4 | `Z_LOG_TRSTLMNT_UPD_TD` | FM / `ZLOG_TD` | PATCH-CLS-01/02 | Yes |
| 5 | `Z_SCE_TRIP_COMPLETE` | FM / `ZLOG_CTD` | — | No change |
| 6 | `Z_SCE_TRIP_UPD` | FM / `ZLOG_CTD` | — | No change |

---

## 9. Deployment Sequence

```
Step 1 — Config: ZTPPAREA (ABC5 → AP) + CTD Master (AREA = AP) for port trucks
Step 2 — FM:     Z_SCE_TRIP_STLMNT       (PATCH-ITM-01/02/03)
Step 3 — FM:     Z_LOG_TRSTLMNT_UPD_TD   (PATCH-CLS-01/02)
Step 4 — Transport to QA → regression test → PRD
```

> Replace `CD:xxxx` with the actual change document number before transport.  
> If `Code Change Document - CTD Trip Production Bug Fixes` is deployed in parallel, ensure HDR-from-master (PATCH-02) is in place before QA sign-off.

---

## 10. Testing Plan

| Test ID | Scenario | Setup | Expected result |
|---|---|---|---|
| **T01** | Port outbound — first loading | Adani Port truck, `ZSCE_CTD_MST-AREA = AP`, `OIGS-TPLST = JGC5`, no YTTS | HDR-AREA = `AP` · ITM-AREA = `JG` · Trip status `01` · Trip **not** closed |
| **T02** | Port outbound — open trip exists | Same truck, trip status `02` from prior leg, outbound TPLST `JGC5` | ITM-AREA = `JG` · Trip **not** closed (JG ≠ AP) |
| **T03** | Port return — reload after plant run | Open trip `02`, HDR = `AP`, `OIGS-TPLST = ABC5`, no YTTS | `Z_SCE_TRIP_COMPLETE` fires · Trip status `03` · New trip `01` created by STLMNT |
| **T04** | Empty return to port | Truck returns empty (no shipment) | No CTD processing · No change to open trip |
| **T05** | Empty return then reload | T04 followed by T03 | Trip closed on reload (T03) |
| **T06** | Plant truck — TPN already closed | YTTS TPN ran, no open trip, plant loading | COMPLETE no-op · New trip `01` via STLMNT |
| **T07** | Plant truck — TPN missed | Open trip `02`, YTTS area = plant base, matches HDR | Loading confirmation closes trip · New trip created |
| **T08** | Plant truck — mid-trip at different area | Open trip `02`, current area ≠ HDR area | Trip **not** closed · STLMNT adds shipment (`02`) |
| **T09** | `ZSCE_CTD_TPN_ACTIVATE` inactive | Deactivate flag in QA | No COMPLETE call at loading · ITM ZTPPAREA still applies |
| **T10** | No `ZTPPAREA` entry for TPLST | Unknown TPLST | ITM-AREA blank · No closure (area blank) — no error |
| **T11** | `Z_SCE_TRIP_COMPLETE` lock failure | Simulate enqueue conflict | Error returned in `et_return` · STLMNT not blocked if error handling requires review |

---

## 11. Risks

| # | Risk | Impact | Mitigation |
|---|---|---|---|
| R1 | `OIGS-TPLST` on port return ≠ `ABC5` | Trip never closes at port reload | Validate T01/T03 with real Adani Port shipment data before PRD |
| R2 | `Z_SCE_TRIP_COMPLETE` issues `COMMIT WORK` | Split LUW with parent loading confirmation | Accept existing pattern (same as YTTS-TPN path); test rollback behaviour |
| R3 | Outbound loading with port TPLST on `OIGS` | Premature closure if TPLST = `ABC5` on outbound | Confirm outbound shipments carry destination TPLST (`JGC5` etc.) |
| R4 | Duplicate closure (TPN + loading) | TPN closes trip; loading COMPLETE finds no open trip | Harmless — "No data found" ignored per PATCH-CLS-02 |
| R5 | Same-base multi-loading | Premature closure at plant | **Confirmed N/A** by business |
| R6 | HDR / ITM area mismatch (`AP` vs `JG`) | Expected by design — HDR = base, ITM = leg | Document for reporting / downstream consumers |

---

## 12. Reference — existing `Z_SCE_TRIP_COMPLETE` logic (no change)

```abap
SELECT area trip_status trip_no
  FROM zsce_ctd_hdr
  WHERE lifnr = i_vendor AND truck_no = i_truck_no
    AND trip_status IN ('01','02').

IF hdr-area = i_area AND trip_status IN ('01','02').
  UPDATE zsce_ctd_hdr SET trip_status = '03'.
ENDIF.
```

Closure is intentionally gated on **area match** — the new loading-confirmation path reuses this without modification.

---

*End of Code Change Document*
