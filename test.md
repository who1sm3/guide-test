# FYP Mark System — Full Setup & Test Guide

End-to-end walkthrough: from `npm install` on a fresh copy of the project, through seeding test data, to verifying FYP1 and FYP2 mark calculations.

Follow top-to-bottom. Each numbered step has an action and a **Pass** criterion.

---

## Part A — Environment Setup

### A1. Prerequisites

Install once on the machine:

- **Node.js 20+** (22 LTS recommended) — https://nodejs.org
- Verify:
  ```bash
  node -v    # v20.x or higher
  npm -v
  ```
- If using WSL: do all work from WSL, do **not** use `sudo` with `npm` / `npx`.

### A2. Install dependencies

From the project root:

```bash
npm install
```

**Pass**: finishes with no fatal errors. Warnings are OK.

If you see `Permission denied` on `next` or `prisma`, fix ownership:
```bash
sudo chown -R $(whoami):$(whoami) .
```
…then re-run `npm install`.

### A3. Configure `.env`

The project root needs a `.env` with at least:

```
DATABASE_URL="file:./dev.db"
```

Add any other secrets your fork uses (JWT secret, admin password, etc.). If `.env` already exists, skip.

### A4. Generate the Prisma client

```bash
npx prisma generate
```

**Pass**: ends with `✔ Generated Prisma Client`.

### A5. Apply the database migration

```bash
npx prisma migrate dev
```

- Accept the default migration name when prompted (or use `--name init`).
- This creates `dev.db` if it doesn't exist and applies all migrations, including the FYP component-marks migration.

**Pass**: ends with `Your database is now in sync with your schema.`

### A6. Start the dev server

```bash
npm run dev
```

Open http://localhost:3000

**Pass**: welcome / portal selection page loads. Terminal shows `✓ Ready in …`.

Leave this running. Do the rest in your browser + a second terminal if needed.

---

## Part B — Seed Test Accounts & Projects

You'll create accounts through the normal signup/admin pages. If you already have a seed script, run it and skip to B4.

### B1. Create students

Open http://localhost:3000/student/signup and create:

| Student ID | Name       | Password    |
|------------|------------|-------------|
| `S1001`    | Student A  | `student123`|
| `S1002`    | Student B  | `student123`|

**Pass**: both can log in at `/student/login`.

### B2. Create lecturers

From the lecturer signup (or admin panel if that's your flow), create:

| Staff ID  | Name         | Role-like  | Password     |
|-----------|--------------|------------|--------------|
| `SV1`     | Supervisor 1 | supervisor | `lecturer123`|
| `A1`      | Assessor 1   | assessor   | `lecturer123`|
| `COORD`   | Coordinator  | admin      | `admin123`   |

Mark **COORD** as admin (`isAdmin = true`). If there's no admin UI toggle, do it once via Prisma Studio:

```bash
npx prisma studio
```
Open the `Lecturer` table → find `COORD` → set `isAdmin` to `true` → Save.

**Pass**: COORD can log in at the lecturer login and sees the Coordinator menu.

### B3. Create FYP projects

Log in as **COORD** → go to `/coordinator/projects` → **Create Project**:

- Project 1: title `Test FYP1 Project`, student = Student A, supervisor = SV1.
- Project 2: title `Test FYP2 Project`, student = Student B, supervisor = SV1.

**Pass**: both projects listed.

### B4. Set phase

Go to `/coordinator/marks` → **Progress Entry & Phase** tab.

- Project 1 → Phase dropdown → `FYP1`.
- Project 2 → Phase dropdown → `FYP2`.

**Pass**: selection persists after a browser refresh.

### B5. Assign assessor

Go to `/coordinator/assign` → assign **A1** to both projects.

**Pass**: A1 sees both projects when logged in at `/assessor/projects`.

---

## Part C — Enter Component Marks

### C1. Supervisor (log in as SV1)

Navigate to `/supervisor/projects` and open each project.

**C1a. Test FYP1 Project** — enter:
- Progress = **80**
- Presentation = **70**
- Report = **90**
- (Paper field is NOT shown)
- Click **Submit Marks**

**Pass**: Saved box shows Progress 80.0, Presentation 70.0, Report 90.0.

**C1b. Test FYP2 Project** — enter:
- Progress = **85**
- Presentation = **75**
- Report = **80**
- Paper = **70**
- Click **Submit Marks**

**Pass**: Saved box shows all four values.

**C1c. Validation** — try entering `150` in any field → **Pass**: red error "X must be 0–100".

### C2. Assessor (log in as A1)

Navigate to `/assessor/projects` and open each assigned project.

**C2a. Test FYP1 Project**:
- Presentation = **75**
- (Report field is NOT shown)
- Click **Submit Marks** → **Pass**: Saved shows Presentation 75.0.

**C2b. Test FYP2 Project**:
- Presentation = **80**
- Report = **70**
- Click **Submit Marks** → **Pass**: both saved.

### C3. Coordinator progress (log in as COORD)

Go to `/coordinator/marks` → **Progress Entry & Phase** tab.

- Test FYP1 Project → Coord Progress = **90** → Save
- Test FYP2 Project → Coord Progress = **85** → Save

**Pass**: values persist after reload. "SV Components" and "Assessor Components" columns now show the marks from C1/C2.

---

## Part D — Generate & Verify

### D1. Generate final marks

Still as COORD on `/coordinator/marks` → click **Generate Final Marks** → confirm.

Switch to **Generated Results** tab.

### D2. Expected FYP1 result

Inputs recap: SV Prog 80, Pres 70, Rep 90 · Asr Pres 75 · Coord 90.

| Component | Calculation | Points |
|---|---|---|
| Progress Total | 80/100×20 + 90/100×10 | **25.0** |
| Presentation raw | 70×0.4 + 75×0.6 = 73 | → 73/100×35 = **25.55** |
| Report Total | 90/100×25 | **22.5** |
| **Final** | 25.0 + 25.55 + 22.5 | **73.05** |

**Pass**: FYP1 row shows Phase `FYP1`, Breakdown `Prog 25.0 · Pres 25.55 · Rep 22.5`, Final **73.05**, Grade **B+**.

### D3. Expected FYP2 result

Inputs recap: SV Pres 75, Rep 80, Paper 70 · Asr Pres 80, Rep 70. (Progress + Paper tracked, not in final.)

| Component | Calculation | Points |
|---|---|---|
| Presentation raw | 75×0.5 + 80×0.5 = 77.5 | → 77.5/100×50 = **38.75** |
| Report raw | 80×0.5 + 70×0.5 = 75 | → 75/100×50 = **37.5** |
| **Final** | 38.75 + 37.5 | **76.25** |

**Pass**: FYP2 row shows Phase `FYP2`, Breakdown includes `Pres 38.75 · Rep 37.5`, Final **76.25**, Grade **A-**.

### D4. Publish

Click **Publish All Results** in the amber banner (or per-row **Publish**).

**Pass**: all rows flip to "Published ✓".

---

## Part E — Student View

### E1. Student A

Log in as Student A → navigate to `/student/fyp/results` (or the results page).

**Pass**: shows final mark **73.05** and grade **B+**.

### E2. Student B

Log in as Student B → same page.

**Pass**: shows final mark **76.25** and grade **A-**.

---

## Part F — Edit & Regenerate

### F1. Change a mark

Log in as SV1 → open Test FYP1 Project → change Report from 90 → **60** → Submit.

### F2. Regenerate

As COORD → `/coordinator/marks` → **Generate Final Marks** again.

Expected new FYP1:
- Report Total = 60/100 × 25 = **15.0**
- Final = 25.0 + 25.55 + 15.0 = **65.55** → Grade **B**

**Pass**: breakdown updates, final = 65.55, Status resets to **Draft**.

### F3. Re-publish

Click **Publish All** → **Pass**: Student A's results page now shows 65.55 / B.

---

## Part G — Legacy & Edge Cases

### G1. Legacy fallback (optional)

Create a 3rd project with NO component marks — supervisor enters only the old aggregate `mark` (e.g. 80), assessor enters 70. Coordinator generates.

Expected: 80 × 0.4 + 70 × 0.6 = **74** · Breakdown column shows `legacy`.

**Pass**: no crash, final = 74.

### G2. Phase switch

For Test FYP1 Project, switch Phase to `FYP2` → **Pass**: supervisor now sees the Paper field; assessor now sees the Report field. (Regenerate to recompute.) Switch back to restore.

### G3. Validation

| Try | Expect |
|---|---|
| Progress = -5 | Client rejects as invalid |
| Progress = 101 | Client rejects |
| Empty all component fields, submit | Falls back to single aggregate `mark` field |
| Two assessors assigned, both submit presentation | Final uses average of their presentation marks |

---

## Part H — Automated Sanity Check (optional)

Open a Node REPL in the project:

```bash
node
```

```js
const { computeFyp1, computeFyp2 } = require("./src/lib/fypCalc.ts");
// If that fails due to ESM/TS, skip — trust the UI test.

computeFyp1({
  svProgress: 80, svPresentation: 70, svReport: 90,
  assessorPresentations: [75], coordinatorProgress: 90,
});
// { progressTotal: 25, presentationTotal: 25.55, reportTotal: 22.5, paperTotal: 0, final: 73.05 }

computeFyp2({
  svProgress: 85, svPresentation: 75, svReport: 80, svPaper: 70,
  assessorPresentations: [80], assessorReports: [70], coordinatorProgress: 85,
});
// { progressTotal: 25, presentationTotal: 38.75, reportTotal: 37.5, paperTotal: 70, final: 76.25 }
```

---

## Sign-off Checklist

### Setup
- [ ] Node 20+ installed
- [ ] `npm install` clean
- [ ] `.env` present with `DATABASE_URL`
- [ ] `npx prisma generate` succeeds
- [ ] `npx prisma migrate dev` succeeds
- [ ] `npm run dev` serves http://localhost:3000

### Seed
- [ ] 2 students created
- [ ] SV1, A1, COORD lecturers created
- [ ] COORD has `isAdmin = true`
- [ ] 2 projects created, phases set, assessor assigned

### Entry
- [ ] Supervisor FYP1 components saved (3 fields)
- [ ] Supervisor FYP2 components saved (4 fields)
- [ ] Assessor FYP1 presentation saved
- [ ] Assessor FYP2 presentation + report saved
- [ ] Coordinator progress saved for both projects

### Generate & Publish
- [ ] FYP1 final = 73.05 / B+
- [ ] FYP2 final = 76.25 / A-
- [ ] Publish works (single + all)

### Student view
- [ ] Student A sees published result
- [ ] Student B sees published result

### Edit flow
- [ ] Edit + regenerate updates mark
- [ ] Regeneration resets to Draft
- [ ] Re-publish propagates to student view

### Edge cases
- [ ] Legacy fallback works for component-less projects
- [ ] Phase switch toggles visible fields
- [ ] Invalid mark values rejected

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `Cannot find module '.prisma/client/default'` | `npx prisma generate` |
| `sh: prisma: Permission denied` | In WSL: `sudo chown -R $(whoami):$(whoami) .` then `npm install` |
| `ERR_REQUIRE_ESM` from Prisma | Node < 20; upgrade Node |
| Network error on signup | Check dev-server terminal for the real error; usually means Prisma client not generated |
| Component fields don't appear in UI | Phase may be unset; go to coordinator/marks → Progress Entry & Phase tab → set it |
| Final mark doesn't use new formula | No component marks entered → falls back to legacy. Enter components, regenerate. |
| "Forbidden: you are not the supervisor/assessor" | Wrong login, or assessor not assigned to that project |
