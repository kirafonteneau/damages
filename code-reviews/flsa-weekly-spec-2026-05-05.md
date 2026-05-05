# FLSA Weekly Damages — Code-Level Spec (v2)

**File touched:** `/home/user/workspace/damages/index.html` (single-file app)
**Date:** May 5, 2026 — revised after Kira's tweaks
**Status:** Draft for Kira's review — no code changes applied yet

---

## TL;DR

Replace the FLSA tab's blanket "weeks × hours × rate" with a configurable weekly grid
(default Mon–Sun, per-period overridable) that inherits period defaults but lets you
override hours, rate, OT method, or exclude a week entirely. Add a rate-change timeline
so a mid-period raise applies to every following week in one click. Cap damages at the
FLSA statute of limitations — willful (3 yr) is the default, non-willful (2 yr) is a
per-scenario toggle. Cap the back end of the weekly grid at the scenario's trial date.
Drop FLSA prejudgment interest everywhere (liquidated damages account for it). Excel and
PDF exports gain a per-week breakdown. Old cases auto-migrate; totals match today's math
when no overrides are set, modulo the removed FLSA interest.

---

## 1. Decisions locked

| # | Decision |
|---|---|
| 1 | Default workweek is **Mon–Sun**, but each FLSA period has a `workweekStart` selector (Sun–Sat). 29 C.F.R. § 778.105 |
| 2 | Partial weeks at period edges are **prorated honestly** by days actually inside the period |
| 3 | Pay rate changes over time = a **rate timeline** at the period level (`rateChanges[]`), with optional per-week rate override |
| 4 | Excluded weeks are an **explicit flag** with a reason code (vacation / FMLA / sick / suspension / other), greyed in the UI, listed in exhibits with reason, $0 contribution |
| 5 | **No FLSA prejudgment interest.** Liquidated damages cover it. Remove `totFInt`, `fi`, "FLSA Interest" labels |
| 6 | Period exceeding **104 weeks** collapses by month with expand/collapse toggles |
| 7 | Exports include the **full weekly breakdown** in both Excel and PDF |
| 8 | New case-level field **`complaintFiledDate`** anchors the FLSA statute of limitations |
| 9 | New per-scenario toggle **`flsaWillful`** — default `true` (3 yr lookback). When `false`, 2 yr |
| 10 | Weekly grid is bounded **on the front by the SOL cutoff** and **on the back by the scenario's trial date** |
| 11 | Changing `workweekStart` on a period with existing overrides is **blocked behind a confirmation** that clears overrides |

---

## 2. Data model changes

### 2.1 Case-level (`defaultCase()`, line 342)

**Add:**
```js
complaintFiledDate: '',  // YYYY-MM-DD; anchors the FLSA SOL cutoff
```

### 2.2 Per-scenario (line 351)

**Add to each scenario object:**
```js
flsaWillful: true,  // true = 3-yr lookback, false = 2-yr (29 U.S.C. § 255(a))
```

### 2.3 Per-FLSA-period (line 359)

**Before:**
```js
flsaPeriods:[{
  id, startDate:'', endDate:'', regularRate:'', weeklyHours:'',
  weeksWorked:'', description:'', otMethod:'half'
}]
```

**After:**
```js
flsaPeriods:[{
  id, startDate:'', endDate:'', description:'', otMethod:'half',
  workweekStart: 1,            // 0=Sun, 1=Mon, ... 6=Sat. Default Monday.
  defaultRegularRate:'',
  defaultWeeklyHours:'',
  rateChanges:[
    // { id, effectiveDate:'2025-06-02', rate:'22.50', note:'Promotion raise' }
  ],
  weekOverrides:{
    // '2025-07-07': { hours:'62', rate:'', otMethod:'full', excluded:false, exclusionReason:'', note:'' }
  }
}]
```

### 2.4 Migration shim

```js
function migrateFlsaPeriod(fp) {
  if (fp.defaultWeeklyHours !== undefined) return fp;
  const out = {
    id: fp.id || uid(),
    startDate: fp.startDate || '',
    endDate: fp.endDate || '',
    description: fp.description || '',
    otMethod: fp.otMethod || 'half',
    workweekStart: 1,            // legacy cases default to Monday
    defaultRegularRate: fp.regularRate || '',
    defaultWeeklyHours: fp.weeklyHours || '',
    rateChanges: [],
    weekOverrides: {}
  };
  if (!out.endDate && out.startDate && fp.weeksWorked) {
    const wks = parseInt(fp.weeksWorked, 10);
    if (Number.isFinite(wks) && wks > 0) {
      const d = new Date(out.startDate + 'T12:00:00');
      d.setDate(d.getDate() + wks * 7 - 1);
      out.endDate = d.toISOString().slice(0, 10);
    }
  }
  return out;
}

function migrateCase(c) {
  if (!c) return c;
  if (c.complaintFiledDate === undefined) c.complaintFiledDate = '';
  if (Array.isArray(c.scenarios)) {
    c.scenarios = c.scenarios.map(s => ({
      flsaWillful: s.flsaWillful !== undefined ? s.flsaWillful : true,
      ...s
    }));
  }
  if (Array.isArray(c.flsaPeriods)) {
    c.flsaPeriods = c.flsaPeriods.map(migrateFlsaPeriod);
  }
  return c;
}
```

Call `migrateCase` after every parse: `store.load`, `blob.load`, `importJSON` (line 727).

---

## 3. Math change — `computeFlsa()` (lines 894–910)

### 3.1 Helpers

```js
// Start of the workweek containing date `dStr`, given a startDow (0=Sun..6=Sat).
function weekStartOf(dStr, startDow = 1) {
  const d = new Date(dStr + 'T12:00:00');
  const dow = d.getDay();
  const offset = ((dow - startDow) + 7) % 7;
  d.setDate(d.getDate() - offset);
  return d.toISOString().slice(0, 10);
}

// Yields { weekKey, daysInPeriod, isPartial } for every workweek
// overlapping [periodStart, periodEnd], using the period's workweekStart.
function* weeklyIterator(periodStart, periodEnd, startDow = 1) {
  if (!periodStart || !periodEnd) return;
  const end = new Date(periodEnd + 'T12:00:00');
  const ps  = new Date(periodStart + 'T12:00:00');
  let cursor = new Date(weekStartOf(periodStart, startDow) + 'T12:00:00');
  while (cursor <= end) {
    const wkStart = new Date(cursor);
    const wkEnd   = new Date(cursor); wkEnd.setDate(wkEnd.getDate() + 6);
    const overlapStart = wkStart < ps  ? ps  : wkStart;
    const overlapEnd   = wkEnd   > end ? end : wkEnd;
    const days = Math.round((overlapEnd - overlapStart) / 86400000) + 1;
    yield {
      weekKey: wkStart.toISOString().slice(0, 10),
      daysInPeriod: days,
      isPartial: days < 7
    };
    cursor.setDate(cursor.getDate() + 7);
  }
}

function rateForWeek(weekKey, period) {
  const sorted = [...(period.rateChanges || [])]
    .filter(r => r.effectiveDate && r.rate !== '')
    .sort((a,b) => a.effectiveDate.localeCompare(b.effectiveDate));
  let r = parseFloat(period.defaultRegularRate) || 0;
  for (const change of sorted) {
    if (change.effectiveDate <= weekKey) r = parseFloat(change.rate) || r;
    else break;
  }
  return r;
}

// Compute the SOL cutoff for a scenario.
// Returns YYYY-MM-DD or '' if no filing date is set (no cutoff applied — toast warns user).
function flsaCutoff(state, scenario) {
  if (!state.complaintFiledDate) return '';
  const yrs = scenario.flsaWillful === false ? 2 : 3;
  const d = new Date(state.complaintFiledDate + 'T12:00:00');
  d.setFullYear(d.getFullYear() - yrs);
  return d.toISOString().slice(0, 10);
}
```

### 3.2 Replacement `computeFlsa`

```js
function computeFlsa(state, trialDate) {
  const scenario = state.scenarios.find(s => s.id === state.activeScenarioId)
                || state.scenarios[0];
  const cutoff = flsaCutoff(state, scenario);
  const flsaRows = [];
  const weeklyRows = [];
  let totOT = 0, totLiq = 0;

  for (const fp of state.flsaPeriods) {
    if (!fp.startDate || !fp.endDate) continue;
    if (!fp.defaultRegularRate && !(fp.rateChanges||[]).length) continue;
    if (!fp.defaultWeeklyHours && Object.keys(fp.weekOverrides||{}).length === 0) continue;

    let pOT=0, pLiq=0, pOtHrs=0, pWeeks=0, pExcluded=0, pSolDropped=0, pPostTrialDropped=0;
    const startDow = (typeof fp.workweekStart === 'number') ? fp.workweekStart : 1;

    for (const w of weeklyIterator(fp.startDate, fp.endDate, startDow)) {
      // SOL gate (front edge): drop weeks whose start is BEFORE the cutoff
      if (cutoff && w.weekKey < cutoff) {
        weeklyRows.push({
          periodId: fp.id, weekKey: w.weekKey, description: fp.description,
          status: 'sol_excluded', solReason: scenario.flsaWillful === false ? '2-yr SOL' : '3-yr SOL',
          isPartial: w.isPartial, daysInPeriod: w.daysInPeriod
        });
        pSolDropped++; continue;
      }
      // Trial gate (back edge): drop weeks that start AFTER the trial date
      if (trialDate && w.weekKey > trialDate) {
        weeklyRows.push({
          periodId: fp.id, weekKey: w.weekKey, description: fp.description,
          status: 'post_trial', isPartial: w.isPartial, daysInPeriod: w.daysInPeriod
        });
        pPostTrialDropped++; continue;
      }

      const ov = (fp.weekOverrides||{})[w.weekKey] || {};
      if (ov.excluded) {
        weeklyRows.push({
          periodId: fp.id, weekKey: w.weekKey, description: fp.description,
          hours: 0, rate: 0, otMethod: fp.otMethod, otHrs: 0, ot: 0, liq: 0,
          status: 'user_excluded', exclusionReason: ov.exclusionReason || 'other',
          isPartial: w.isPartial, daysInPeriod: w.daysInPeriod, note: ov.note || ''
        });
        pExcluded++; continue;
      }

      const hours = (ov.hours !== undefined && ov.hours !== '')
        ? (parseFloat(ov.hours) || 0)
        : (parseFloat(fp.defaultWeeklyHours) || 0);
      const rate = (ov.rate !== undefined && ov.rate !== '')
        ? (parseFloat(ov.rate) || 0)
        : rateForWeek(w.weekKey, fp);
      const meth = ov.otMethod || fp.otMethod || 'half';
      const mult = meth === 'full' ? 1.5 : 0.5;

      // Honest proration for partial weeks
      const threshold = 40 * (w.daysInPeriod / 7);
      const effHours  = hours * (w.daysInPeriod / 7);
      const otHrs = Math.max(0, effHours - threshold);
      const ot = otHrs * rate * mult;
      const liq = ot;

      weeklyRows.push({
        periodId: fp.id, weekKey: w.weekKey, description: fp.description,
        hours, rate, otMethod: meth, otHrs, ot, liq,
        status: 'counted',
        isPartial: w.isPartial, daysInPeriod: w.daysInPeriod, note: ov.note || ''
      });
      pOT += ot; pLiq += liq; pOtHrs += otHrs; pWeeks++;
    }

    flsaRows.push({
      ...fp,
      otHrs: pOtHrs, ot: pOT, liq: pLiq, sub: pOT + pLiq,
      weeksCounted: pWeeks, weeksExcluded: pExcluded,
      weeksDroppedBySOL: pSolDropped, weeksDroppedPostTrial: pPostTrialDropped
    });
    totOT += pOT; totLiq += pLiq;
  }

  return {
    flsaRows, weeklyRows,
    totOT, totLiq,
    totFInt: 0,                 // back-compat; always 0
    flsaTotal: totOT + totLiq,
    cutoff,                     // for UI banners
    willful: scenario.flsaWillful !== false
  };
}
```

### 3.3 Why proration uses `daysInPeriod / 7`

A worker who normally works 50 hrs/wk and the period covers Wed–Sun of one week is honestly 5/7 of a workweek. Hours scale linearly to ~35.7, threshold scales to ~28.6, OT comes out to ~7.1 hrs. This avoids both overcounting (full week from a partial) and undercounting (snapping to full weeks).

---

## 4. UI changes

### 4.1 Case Setup (`renderSetup`, line 1672)

Add a new field in the existing `g2` grid, after Attorney:

- **Complaint filing date** (`type=date`, `dk:'complaintFiledDate'`)
  - Help text: *"Anchors the FLSA statute-of-limitations cutoff. 29 U.S.C. § 255(a)."*
  - If left blank: a yellow info note appears on the FLSA tab — *"No filing date set; SOL cap not applied. Set complaint filing date in Case Setup."*

### 4.2 Scenario rows (existing scenarios UI in Setup)

Each scenario row gets a new control next to the trial date input:

- **FLSA lookback:** segmented toggle  `[Willful — 3 yr] [Non-willful — 2 yr]`  default Willful.
- Stored as `scenario.flsaWillful` (boolean).
- Toggling re-renders, which updates the SOL cutoff and the FLSA totals on the Results tab.

### 4.3 FLSA period header (`renderFlsa`, line 1931)

Existing fields, with these changes:

1. `regularRate` → renamed label "Default regular rate ($/hr)", bound to `defaultRegularRate`
2. `weeklyHours` → renamed label "Default weekly hours", bound to `defaultWeeklyHours`
3. `weeksWorked` → **removed** (derived from start/end)
4. **New:** `workweekStart` dropdown labeled **"Workweek begins"** with options Sun/Mon/Tue/Wed/Thu/Fri/Sat. Default Mon. Help tooltip: *"29 C.F.R. § 778.105 — the employer designates the workweek as any fixed, recurring 7-day period. Defaults to Monday; change if the employer's policy designates a different start."*

**Workweek change guard:** if the user changes `workweekStart` on a period that has any entries in `weekOverrides` or `rateChanges`, intercept with `confirm_action`-equivalent in-app confirm dialog:

> "Changing the workweek will clear N existing weekly overrides (and may shift M rate-change effective dates). Continue?"
> [Cancel] [Clear and continue]

If confirmed: clear `weekOverrides` and snap each `rateChanges[i].effectiveDate` to the nearest new-workweek start.
If cancelled: revert the dropdown.

### 4.4 Rate timeline subsection

Under the period header, collapsed by default with `+ Add rate change`. Each entry:

| Effective (week start) | New rate ($/hr) | Note (optional) | × |

When the user picks an effective date, snap to the period's workweek start (using `weekStartOf` with the period's `workweekStart`) and show the snapped date. Validate: must fall within `startDate..endDate`, no duplicates. Sort by date on render.

### 4.5 Weekly breakdown table

Appears once a period has start + end dates and at least one of `defaultRegularRate` / `rateChanges`.

**Banner above the table:**

```
Workweek: Mon–Sun  •  SOL cutoff: Mar 14, 2023 (3-yr willful)
N weeks counted  •  M excluded by user  •  K dropped by SOL  •  J dropped past trial
```

**Bulk action bar (sticky):**

```
[ Select all visible ] [ Set hours… ] [ Set rate… ] [ Exclude (reason ▾) ] [ Reset to defaults ]
```

Selection: per-row checkbox + click-and-shift-click on the date column.

**Table columns:**

| ☐ | Week of (X) | Days | Hours | Rate | OT method | OT hrs | Unpaid OT | Status | ⋯ |

Where `(X)` = the period's workweek-start day name.

Row states:
- **Counted, inherit:** values shown muted with a "default" tag
- **Counted, overridden:** values shown normal weight with gold "✎"
- **User excluded:** greyed; shows "Excluded — vacation"
- **SOL excluded:** dark grey, locked, no inputs; shows "Outside 3-yr SOL"
- **Post-trial:** dark grey, locked; shows "After trial date"
- **Partial:** small "(N of 7 days)" tag in the Days column

Collapse-by-month when `weeklyRows.length > 104`.

### 4.6 Period totals (existing metric cards)

Same three cards (Total OT hours / Unpaid OT / + Liquidated). No "interest" card.

### 4.7 `handleChange` extension (line 1596)

Branch on two new data attributes:

```js
// Weekly override:
//   data-list="flsaPeriods" data-idx="0" data-overrides-key="2025-07-07" data-field="hours"
// Rate change entry:
//   data-list="flsaPeriods" data-idx="0" data-rate-change-id="abc" data-field="rate"
```

```js
if (list === 'flsaPeriods' && el.dataset.overridesKey) {
  const wk = el.dataset.overridesKey;
  STATE.flsaPeriods[idx].weekOverrides ||= {};
  STATE.flsaPeriods[idx].weekOverrides[wk] ||= {};
  STATE.flsaPeriods[idx].weekOverrides[wk][field] = val;
  pruneOverride(STATE.flsaPeriods[idx].weekOverrides, wk);
} else if (list === 'flsaPeriods' && el.dataset.rateChangeId) {
  const rc = STATE.flsaPeriods[idx].rateChanges
    .find(r => r.id === el.dataset.rateChangeId);
  if (rc) rc[field] = val;
}
```

Workweek-start change is intercepted before assignment to run the guard dialog.

---

## 5. Export changes

### 5.1 Excel — `buildScenarioExcelSheets` (lines 983–1067)

- **Summary sheet (line 1006):** remove `['FLSA Interest', res.flsa.totFInt]` row.
- **FLSA Overtime sheet** restructured into two sections:
  1. **Period Summary**
     cols: Period / Position | Workweek | OT Hours | OT Wages | Liquidated | Subtotal
  2. **Weekly Detail**
     cols: Period | Week of | Days | Hours | Rate | OT method | OT hrs | Unpaid OT | Liquidated | Status | Note
     - Status values: Counted / Override / Excluded — vacation / Outside 3-yr SOL / After trial date / Partial
- **New "FLSA Settings" mini-block** above the period summary, listing: Filing date, Lookback (Willful 3-yr / Non-willful 2-yr), Cutoff date, Trial date.

Column widths: `[18, 14, 8, 10, 12, 12, 10, 14, 14, 22, 30]`.

### 5.2 PDF — `exportPDF` and `exportAllPDF`

- Drop "Interest" column from FLSA period table; fix the existing totals-row bug.
- Add a second `autoTable` titled "FLSA Overtime — Weekly Detail" with the same columns as Excel weekly detail.
- Add a one-line FLSA settings header above the FLSA section: *"Filing: [date] · Lookback: [Willful 3-yr / Non-willful 2-yr] · Cutoff: [date] · Trial: [date]"*.
- `styles.fontSize:7`, `cellPadding:1.8` for the weekly detail to fit letter portrait; auto-paginate.
- Same edits in `exportAllPDF`.

### 5.3 Footnotes (line 1194 + lines 2196–2219)

Edit footnote 1 to clarify it applies to **discrimination back pay only** (not FLSA).

Add footnote 4:

> "FLSA Damages: 29 U.S.C. § 216(b) provides that liquidated damages in an amount equal to unpaid overtime compensation are awarded in lieu of prejudgment interest. *See Brooklyn Sav. Bank v. O'Neil*, 324 U.S. 697, 715 (1945); *Martin v. Cooper Elec. Supply Co.*, 940 F.2d 896, 910 (3d Cir. 1991). No separate prejudgment interest is calculated on FLSA damages."

Add footnote 5:

> "FLSA Statute of Limitations: Damages are limited to the [3-year willful / 2-year non-willful] period preceding the date of filing, [filing date]. 29 U.S.C. § 255(a). Workweek for each period reflects the employer's designation under 29 C.F.R. § 778.105."

### 5.4 Results tab metric cards (lines 2083–2086)

Drop the "FLSA interest" card. Two FLSA cards remain (Unpaid OT, Liquidated). Switch the row layout from `g4` to two rows of `g3` when both disc and flsa are present.

### 5.5 Results tab FLSA breakdown table (lines 2166–2194)

- Drop the Interest column from header and body.
- Update card-sub at line 2170: *"Unpaid OT wages + equal liquidated damages (29 U.S.C. § 216(b)). Lookback: [Willful 3-yr / Non-willful 2-yr] from filing date [date]."*
- Add a "Show weekly detail" toggle that renders `flsa.weeklyRows` underneath.

---

## 6. Save / load surface

- `exportJSON` (line 719) — no change.
- All parse paths call `migrateCase` → `migrateFlsaPeriod`.
- `scheduleSave` — no change; new fields auto-persist.

---

## 7. Test plan

1. **No-override parity (modulo interest):** open existing case → totals = old totals minus the FLSA interest figure. Confirm in writing.
2. **Single-week override:** add 60-hr override on one week of a 50-hr blanket. Period total grows by exactly `(60-50) * rate * 0.5`.
3. **Excluded week:** mark vacation. Total drops by `(50-40) * rate * 0.5`.
4. **Mid-period rate change:** add change halfway through 52 weeks. Grid shows old rate before, new rate after, boundary on the period's workweek start.
5. **Partial weeks:** Wed-start period, Tue-end period three weeks later. First and last rows tagged partial; OT prorated.
6. **3-yr SOL — willful:** filing 2026-03-14, period start 2022-01-01. Weeks before 2023-03-14 marked "Outside 3-yr SOL" and contribute $0.
7. **2-yr SOL — non-willful:** flip scenario toggle. Cutoff shifts to 2024-03-14; more weeks drop. Weekly grid + banner update.
8. **Trial-date cap:** trial 2026-09-15, period ends 2027-12-31. Weeks after 2026-09-15 marked "After trial date" and contribute $0.
9. **Workweek change:** Sun–Sat period with 3 overrides → change to Mon–Sun → confirmation dialog → on Confirm, overrides cleared. On Cancel, dropdown reverts.
10. **Rate-change snap:** enter effective date 2025-06-04 (Wed) on a Mon–Sun period → snaps to 2025-06-02 (Mon).
11. **Long period (3 yrs):** month-collapse renders, totals match expanded view.
12. **No filing date:** yellow banner appears; SOL not applied; totals not silently capped.
13. **Excel + PDF:** weekly detail present, no "FLSA Interest" anywhere, FLSA settings header present, footnotes 4 and 5 present.
14. **Migration round-trip:** load pre-change blob case → save → reload → shape stable, totals stable (modulo interest).

---

## 8. Sequencing & rough effort

| Step | What | Approx |
|---|---|---|
| 1 | Migration shim + plumbing (load/import/blob paths) | 30 min |
| 2 | New helpers + `computeFlsa` (no UI yet; verify totals via console) | 1 hr |
| 3 | Case Setup additions: `complaintFiledDate` field, scenario willful toggle | 30 min |
| 4 | New `renderFlsa` (period header, rate timeline, weekly grid, bulk bar, banner) | 3 hr |
| 5 | `handleChange` extension + workweek-change guard dialog | 45 min |
| 6 | Excel export — settings block + period summary + weekly detail | 1 hr |
| 7 | PDF export — drop interest col, fix totals bug, add weekly detail, settings header | 1 hr |
| 8 | Footnotes + Results tab metric cards + Notes panel updates | 30 min |
| 9 | Test plan walkthrough on 2–3 real cases | 1 hr |
| **Total** | | **~9 hr** |

---

## 9. Open risks

- **Removing FLSA interest changes existing case totals.** Suggest a one-time toast on first load of a migrated case: *"FLSA prejudgment interest has been removed — liquidated damages already account for the time-value of money under 29 U.S.C. § 216(b). Total may differ from prior exports."*
- **No filing date = no cap.** A user who forgets to set the field could overstate damages. Mitigation: yellow banner on the FLSA tab and on the Results tab. Optional stronger version: hard-warn at export time. Flag for your call.
- **Long-period DOM weight.** 3-yr SOL caps the worst case at ~156 weeks per period; collapse-by-month at >104 keeps the DOM healthy.
- **JSON bloat in saved cases.** `pruneOverride` removes empty override objects so blanket-only periods stay compact.
- **Smokeball / Lawmatics integrations.** `flsa.flsaTotal` schema unchanged (just smaller); `totFInt` remains in payload as `0` for back-compat.

---

## 10. Things I'm explicitly NOT changing

- Discrimination math, back pay calc, prejudgment interest on discrimination — untouched.
- DOL rates tab — untouched. Worth a small label edit on line 1993 to say "applies to discrimination back pay" — flagging here, not making the change.
- Azure blob storage / load logic — untouched except for the migration call.
- Scenario tabs UI shell, mitigation, benefits — untouched.

---

## Approval gate

Reply with one of:

- **A — Build it as specced.** I'll implement steps 1–9 in sequence, run the test plan, and open a PR.
- **B — Build it with these tweaks:** [list tweaks]
- **C — Hold; I want to revise the spec.**
