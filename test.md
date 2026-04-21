# FYP Mark Calculation — Full Test Plan

End-to-end checklist to verify the updated FYP1 / FYP2 calculation. Follow top-to-bottom. Each step says what to do and what "pass" looks like.

---

## 0. Prerequisites

1. Prisma client regenerated and migration applied:
   ```bash
   npx prisma generate
   npx prisma migrate dev
   ```
2. Dev server running:
   ```bash
   npm run dev
   ```
   Open http://localhost:3000
3. Have admin / coordinator login credentials ready. If no admin exists, create one via your usual seed / signup flow first.

---

## 1. Seed test data

You need (at minimum) to test one FYP1 and one FYP2 project.

### Accounts to create
| Role | Account |
|---|---|
| Student A | e.g. `S1001` — will be FYP1 |
| Student B | e.g. `S1002` — will be FYP2 |
| Supervisor SV1 | lecturer |
| Assessor A1 | lecturer |
| Coordinator | lecturer with `isAdmin = true` |

Create via the existing signup / admin pages.

### Projects
1. Log in as **Coordinator** → `coordinator/projects` → create 2 projects:
   - Project FYP1: title "Test FYP1 Project", student = Student A, supervisor = SV1.
   - Project FYP2: title "Test FYP2 Project", student = Student B, supervisor = SV1.
2. Go to `coordinator/marks` → **Progress Entry & Phase** tab.
   - For Project FYP1 → Phase dropdown → `FYP1`.
   - For Project FYP2 → Phase dropdown → `FYP2`.
3. Go to `coordinator/assign` → assign **Assessor A1** to both projects.

**Pass**: both projects visible in coordinator tables with correct phase.

---

## 2. Supervisor component marks

Log in as **SV1** → open each project.

### 2a. FYP1 project
Enter:
- Progress = **80**
- Presentation = **70**
- Report = **90**
- Paper field should NOT appear (FYP1)
- Notes = anything
- Click **Submit Marks**

**Pass**:
- Green "Saved" box shows: Progress 80.0, Presentation 70.0, Report 90.0.
- No Paper row in Saved list.

### 2b. FYP2 project
Enter:
- Progress = **85**
- Presentation = **75**
- Report = **80**
- Paper = **70**
- Paper field IS visible.
- Click **Submit Marks**

**Pass**: Saved box shows all four values.

### 2c. Validation checks
- Enter `150` in any field → submit → red error "X must be 0–100".
- Clear all component fields → submit → should fail or accept fallback; not critical.

---

## 3. Assessor component marks

Log in as **A1** → open each assigned project.

### 3a. FYP1 project
- Presentation = **75**
- Report field should NOT appear.
- Click **Submit Marks**.

**Pass**: Saved shows only Presentation 75.0.

### 3b. FYP2 project
- Presentation = **80**
- Report = **70**
- Click **Submit Marks**.

**Pass**: Saved shows both values.

---

## 4. Coordinator progress mark

Log in as **Coordinator** → `coordinator/marks` → **Progress Entry & Phase** tab.

For each project, enter Coord Progress:
- Project FYP1: **90**
- Project FYP2: **85**

Click **Save** on each row.

**Pass**: value persists on reload; "SV Components" and "Assessor Components" columns now show the marks you entered in sections 2–3.

---

## 5. Generate final marks

Still on `coordinator/marks` → click **Generate Final Marks** → confirm.

Switch to **Generated Results** tab.

### 5a. Expected FYP1 calculation

Inputs:
- SV Progress 80, Presentation 70, Report 90
- Assessor Presentation 75
- Coord Progress 90

Formula:
- Progress Total = 80/100 × 20 + 90/100 × 10 = **16 + 9 = 25.0**
- Presentation raw = 70 × 0.4 + 75 × 0.6 = 28 + 45 = 73
  - Presentation Total = 73/100 × 35 = **25.55**
- Report Total = 90/100 × 25 = **22.5**
- **Final = 25.0 + 25.55 + 22.5 = 73.05**

**Pass**: Row for FYP1 project shows:
- Phase: FYP1
- Breakdown: `Prog 25.0 · Pres 25.55 (or 25.6) · Rep 22.5`
- Final: **73.05** (or 73.1 rounded in display)
- Grade: **B+**

### 5b. Expected FYP2 calculation

Inputs:
- SV Presentation 75, Report 80 (Progress 85, Paper 70 tracked, not in final)
- Assessor Presentation 80, Report 70
- Coord Progress 85 (tracked, not in final)

Defaults: presentationSvWeight 0.5, reportSvWeight 0.5, includeProgressInFinal false.

Formula:
- Presentation raw = 75 × 0.5 + 80 × 0.5 = **77.5**
  - Presentation Total = 77.5/100 × 50 = **38.75**
- Report raw = 80 × 0.5 + 70 × 0.5 = **75**
  - Report Total = 75/100 × 50 = **37.5**
- **Final = 38.75 + 37.5 = 76.25**

**Pass**: Row for FYP2 project shows:
- Phase: FYP2
- Breakdown includes `Pres 38.75 · Rep 37.5` (Prog shown too but doesn't add to final, Pap shown as stored)
- Final: **76.25** (display may round to 76.3)
- Grade: **A-**

---

## 6. Publish results

On **Generated Results** tab:
- Click **Publish** on one row → that row flips to "Published ✓".
- Click **Publish All Results** banner → all remaining rows flip to published.

**Pass**: all rows show "Published ✓".

---

## 7. Student view

Log in as **Student A** → navigate to `student/fyp/results` (or equivalent page).

**Pass**: final mark and grade match section 5a (73.05 / B+).

Log in as **Student B** → same page.

**Pass**: final mark matches section 5b (76.25 / A-).

---

## 8. Edit & regenerate

1. As SV1, change FYP1 Report mark from 90 → **60**. Submit.
2. As Coordinator, click **Generate Final Marks** again.
3. Expected new FYP1 final:
   - Report Total = 60/100 × 25 = 15.0
   - Final = 25.0 + 25.55 + 15.0 = **65.55**
4. Previously-published status resets to Draft (expected — new generation un-publishes).

**Pass**: breakdown updates, final = 65.55, grade = B, status = Draft.

Re-publish → student sees updated mark.

---

## 9. Legacy fallback (optional)

Create a project where NO component marks are entered — only old aggregate `mark` via API or DB:

- Supervisor `mark` = 80, no `progressMark`/etc.
- Assessor `mark` = 70, no `presentationMark`.

Generate with legacy weights SV 40 / Asr 60.

- Expected final = 80 × 0.4 + 70 × 0.6 = **74**
- Breakdown column shows "legacy" (no component data)

**Pass**: falls back cleanly without crashing; final = 74.

---

## 10. Edge cases

| Case | Expected |
|---|---|
| Switch project phase FYP1 → FYP2 after marks entered | Paper / Report assessor inputs become visible; previously-hidden values are preserved. Regenerate to recompute with new formula. |
| SV submits only Progress (presentation/report empty) | Generates with missing components treated as 0 → lower final. |
| No assessor assigned | Presentation raw uses 0 for assessor component. |
| Multiple assessors | Their presentation/report marks are averaged before weighting. |
| Progress mark = 100 + Coord 100 | Progress Total = 20 + 10 = 30 (cap respected). |

---

## 11. Automated sanity check (optional)

In a Node REPL or test file, exercise the pure calc module directly:

```ts
import { computeFyp1, computeFyp2 } from "@/lib/fypCalc";

console.log(computeFyp1({
  svProgress: 80, svPresentation: 70, svReport: 90,
  assessorPresentations: [75], coordinatorProgress: 90,
}));
// Expect: { progressTotal: 25, presentationTotal: 25.55, reportTotal: 22.5, paperTotal: 0, final: 73.05 }

console.log(computeFyp2({
  svProgress: 85, svPresentation: 75, svReport: 80, svPaper: 70,
  assessorPresentations: [80], assessorReports: [70], coordinatorProgress: 85,
}));
// Expect: { progressTotal: 25, presentationTotal: 38.75, reportTotal: 37.5, paperTotal: 70, final: 76.25 }
```

---

## Sign-off checklist

- [ ] Migration applied without errors
- [ ] Dev server starts clean
- [ ] Supervisor can save FYP1 components (3 fields)
- [ ] Supervisor can save FYP2 components (4 fields)
- [ ] Assessor can save FYP1 presentation
- [ ] Assessor can save FYP2 presentation + report
- [ ] Coordinator can set phase per project
- [ ] Coordinator can save progress mark per project
- [ ] Generate produces correct FYP1 final (section 5a)
- [ ] Generate produces correct FYP2 final (section 5b)
- [ ] Publish / unpublish works
- [ ] Student sees published final mark
- [ ] Regeneration after edit updates mark and resets to Draft
- [ ] Legacy fallback still works for projects with no components
