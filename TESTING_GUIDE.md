# UniKL FYP & Course Management System — Complete Testing Guide

> **Audience:** Beginners. This guide assumes you know **nothing** about the app. Follow the steps in order. Each section tells you what to click, what to type, and what you should see if it works correctly.

---

## Table of Contents

1. [What This App Is](#1-what-this-app-is)
2. [Before You Start — Setup](#2-before-you-start--setup)
3. [The User Roles (Who Can Do What)](#3-the-user-roles-who-can-do-what)
4. [Testing Plan Overview](#4-testing-plan-overview)
5. [PHASE 1 — Create the First Admin (Coordinator)](#phase-1--create-the-first-admin-coordinator)
6. [PHASE 2 — Admin Tests](#phase-2--admin-tests)
7. [PHASE 3 — Lecturer (Supervisor) Tests](#phase-3--lecturer-supervisor-tests)
8. [PHASE 4 — Lecturer (Assessor) Tests](#phase-4--lecturer-assessor-tests)
9. [PHASE 5 — Student Tests](#phase-5--student-tests)
10. [PHASE 6 — FYP End-to-End Workflow](#phase-6--fyp-end-to-end-workflow)
11. [PHASE 7 — Coordinator Final Marks](#phase-7--coordinator-final-marks)
12. [PHASE 8 — Edge Cases & Security Checks](#phase-8--edge-cases--security-checks)
13. [PHASE 9 — Security Testing with OWASP ZAP](#phase-9--security-testing-with-owasp-zap)
14. [PHASE 10 — Security Testing with Burp Suite](#phase-10--security-testing-with-burp-suite)
15. [PHASE 11 — Deploying to Vercel](#phase-11--deploying-to-vercel)
16. [Bug Reporting Template](#bug-reporting-template)
17. [Quick Reference — URLs & Endpoints](#quick-reference--urls--endpoints)

---

## 1. What This App Is

This is a **University Final Year Project (FYP) and Course Management System** for UniKL. It's built with Next.js, Prisma, and SQLite.

It supports **six user roles**:

| Role | What they do |
|------|--------------|
| **Admin / Coordinator** | Manage everything — users, courses, GPA scales, FYP project assignments, final marks |
| **Lecturer (Supervisor)** | Supervise FYP students, review submissions, grade |
| **Lecturer (Assessor)** | Assess FYP presentations & reports |
| **Lecturer (Teaching)** | Teach courses, post assignments/announcements, mark attendance |
| **Student (FYP)** | Submit FYP reports, view supervisor feedback, see published marks |
| **Student (Course)** | Enroll in courses, submit assignments, read announcements |

> A single lecturer can be both a supervisor/assessor AND teach courses. Admins are lecturers with the `isAdmin` flag turned on.

---

## 2. Before You Start — Setup

### 2.1 Install dependencies

Open a terminal in the project folder and run:

```bash
npm install
```

Wait for it to finish (may take 1–3 minutes).

### 2.2 Set up the database

```bash
npx prisma migrate deploy
npx prisma generate
```

### 2.3 Start the dev server

```bash
npm run dev
```

You should see something like:

```
▲ Next.js 16.1.6
- Local:        http://localhost:3000
- ready in ~2s
```

### 2.4 Open the app

Open your browser and go to **http://localhost:3000/welcome**

You should see a portal selection page with cards for:
- FYP Student
- Supervisor
- Assessor
- FYP Coordinator (gold badge)

✅ **If you see this page, setup is complete.**

❌ **If you see an error**, check:
- Is `dev.db` present in the project root?
- Did `npm install` finish without errors?
- Is port 3000 already in use? (try `lsof -i :3000`)

---

## 3. The User Roles (Who Can Do What)

| URL prefix | Who can access | Notes |
|------------|----------------|-------|
| `/welcome` | Everyone | Public portal selection |
| `/login` | Public | Lecturer login (Staff ID + Password) |
| `/signup` | Public | Lecturer self-registration |
| `/student/login` | Public | Student login (Student ID + Password) |
| `/student/signup` | Public | Student self-registration |
| `/student/*` | Students only | Student dashboard, courses, assignments, FYP |
| `/admin/*` | Admins only | User management, GPA scale |
| `/coordinator/*` | Admins only | FYP coordination, final marks |
| `/supervisor/*` | Lecturers (role=supervisor) | FYP supervisor pages |
| `/assessor/*` | Lecturers (role=assessor) | FYP assessor pages |
| `/my-courses` | Lecturers | Teaching course management |
| `/courses/*` | Lecturers | Course details / AI course create |

---

## 4. Testing Plan Overview

You will test in this order (each phase depends on previous ones):

1. **Create the first admin** — without an admin, nothing works.
2. **Admin creates users** — lecturers + students.
3. **Lecturer (supervisor) tests** — log in, manage courses.
4. **Lecturer (assessor) tests** — log in, view assigned projects.
5. **Student tests** — enroll, submit assignments.
6. **End-to-end FYP workflow** — full lifecycle from project creation to publication.
7. **Coordinator final marks** — generate and publish FYP results.
8. **Edge cases & security** — wrong role access, validation errors.

> 💡 **Tip:** Use two different browsers (or one normal + one in Incognito) so you can be logged in as two roles at the same time. This makes testing faster.

---

## PHASE 1 — Create the First Admin (Coordinator)

The first admin must be created via an API call (there's no UI for the very first admin).

### Step 1.1 — Create admin via terminal

Open a **new terminal** (keep the dev server running) and run:

```bash
curl -X POST http://localhost:3000/api/auth/bootstrap-admin \
  -H "Content-Type: application/json" \
  -d '{"secret":"unikl-bootstrap-2026","staffId":"ADMIN001","name":"Admin User","password":"admin123"}'
```

**Expected response:**
```json
{"ok":true,"adminId":"..."}
```

✅ **What to verify:**
- Response is `{"ok":true, "adminId":"..."}`
- No error in the dev server terminal

❌ **If it fails** with `"admin already exists"`, an admin was created earlier — proceed to Step 1.2 with those credentials.

### Step 1.2 — Log in as admin

1. Open browser → **http://localhost:3000/login**
2. Enter:
   - **Staff ID:** `ADMIN001`
   - **Password:** `admin123`
3. Click **Login**

✅ **Expected:** You're redirected to the main dashboard (`/`). The top navigation should show admin-only links like **Admin** and **Coordinator**.

❌ **If login fails:** Check the dev server terminal for errors. Re-run the bootstrap curl command with a different `staffId`.

---

## PHASE 2 — Admin Tests

### Test 2.1 — Add a Lecturer (Supervisor role)

1. Go to **http://localhost:3000/admin/manage**
2. Find the **"Add Lecturer"** form
3. Fill in:
   - **Staff ID:** `SV1`
   - **Name:** `Supervisor One`
   - **Password:** `lecturer123`
   - **Role:** `Supervisor`
   - **FYP Coordinator:** ☐ (unchecked)
   - **Assign Course(s):** leave blank for now
4. Click **Add Lecturer**

✅ **Expected:** The lecturer appears in the table below with role "Supervisor".

### Test 2.2 — Add a Lecturer (Assessor role)

Repeat Test 2.1 with:
- **Staff ID:** `A1`
- **Name:** `Assessor One`
- **Password:** `lecturer123`
- **Role:** `Assessor`

✅ **Expected:** Appears in lecturer table with role "Assessor".

### Test 2.3 — Add a Lecturer (Teaching only, no FYP role)

Repeat with:
- **Staff ID:** `LEC1`
- **Name:** `Teaching Lecturer`
- **Password:** `lecturer123`
- **Role:** `Supervisor` (default; this lecturer will mainly teach)

### Test 2.4 — Add a Student

1. Scroll to the **"Add Student"** form on the same page
2. Fill in:
   - **Student ID:** `S1001`
   - **Student Name:** `Student Alpha`
   - **Assign Supervisor:** `Supervisor One`
   - **Assign Assessor:** `Assessor One`
3. Click **Add Student**

✅ **Expected:**
- Student appears in the student table
- **Password is automatically set to the Student ID** (`S1001`) — important for testing later

### Test 2.5 — Add a second student (no supervisor assigned)

Add:
- **Student ID:** `S1002`
- **Name:** `Student Beta`
- **Supervisor:** (leave blank)
- **Assessor:** (leave blank)

### Test 2.6 — Edit a Lecturer

1. In the lecturer table, click **Edit** next to `SV1`
2. Change the name to `Supervisor One (edited)`
3. Tick **Admin Access**
4. Click **Save Changes**

✅ **Expected:** Name updates in the table. Role column now shows "Coordinator".

> ❗ Don't forget to untick Admin Access afterward so this lecturer is back to being a supervisor for later tests.

### Test 2.7 — Delete a Lecturer

1. Add a throwaway lecturer (`TEMP1` / `Temp Lecturer`)
2. Click **Delete** → confirm
3. ✅ Lecturer is removed from the table

### Test 2.8 — GPA Scale Configuration

1. Go to **http://localhost:3000/admin/gpa-scale**
2. You should see a table of grade thresholds (e.g., Min 90, Max 100, Grade A+, GPA 4.0)
3. Click **Add row** — verify a blank row appears
4. Click **Remove** on the new row — verify it's gone
5. Try saving with invalid data (e.g., Min=80, Max=70) → should show validation error
6. Save with valid data → should see success message

✅ **Expected:** Validation works; valid data saves correctly.

---

## PHASE 3 — Lecturer (Supervisor) Tests

### Step 3.1 — Log out as admin, log in as supervisor

1. Click logout (or open Incognito window)
2. Go to **http://localhost:3000/login**
3. Login: `SV1` / `lecturer123`

✅ **Expected:** Redirected to dashboard with **Supervisor** menu links.

### Test 3.2 — Assign yourself to teach a course

> First we need a course. As supervisor, you typically don't have courses yet — let's first create one.

#### Option A: Manual creation via API (advanced)
Skip this; let's use the AI course create instead.

#### Option B: AI course create

1. Go to **http://localhost:3000/courses/ai-create**
2. In the text area, paste:
   ```
   Course: ABC12345 — Test Course
   Semester: 1
   Year: 2026
   CLO1 - Understand basics
   CLO2 - Apply knowledge
   Assessments:
   Quiz (CLO1) - 30%
   Project (CLO2) - 70%
   ```
3. Click **Analyze**

✅ **Expected:** Preview shows extracted course code, name, CLOs, and assessments. Weightages sum to 100%.

4. Click **Save Course**

✅ **Expected:** Course is saved.

### Test 3.3 — Pick courses to teach

1. Go to **http://localhost:3000/my-courses**
2. You should see your newly created course in the list
3. Tick the checkbox next to it
4. Click **Save Changes**

✅ **Expected:** Course appears under "Currently Teaching".

❌ **Edge case to test:** Try to select more than 3 courses → should show error.

### Test 3.4 — Post an announcement

1. Click on your course card → navigate to `/my-courses/{courseId}`
2. Click the **Announcements** card
3. Click **+ New Announcement**
4. Fill in:
   - **Target Audience:** All classes (leave dropdown empty)
   - **Title:** `Welcome to the course`
   - **Content:** `Please read the syllabus.`
5. Click **Post Announcement**

✅ **Expected:** Announcement appears in the list below.

### Test 3.5 — Create an assignment

1. From the course page, click the **Assignments** card
2. Click **+ New Assignment**
3. Fill in:
   - **Course Class:** `Class 1`
   - **Title:** `Assignment 1`
   - **Description:** `Submit a 2-page report.`
   - **Due Date:** (pick a future date)
   - **Max Score:** `100`
4. Click **Create Assignment**

✅ **Expected:** Assignment appears in the list with submission count `0`.

---

## PHASE 4 — Lecturer (Assessor) Tests

### Step 4.1 — Log in as assessor

1. Logout / open another browser window
2. Login: `A1` / `lecturer123` at `/login`

### Test 4.2 — View assigned projects

1. Go to **http://localhost:3000/assessor/projects**

✅ **Expected:** Page loads. Initially empty (no projects assigned yet — we'll fix this in Phase 6).

❌ **If you get a permission error:** Check that this lecturer's role is "Assessor" in the admin panel.

---

## PHASE 5 — Student Tests

### Step 5.1 — Log in as student

1. Logout / open another browser window
2. Go to **http://localhost:3000/student/login**
3. Login: `S1001` / `S1001` (password = student ID for admin-created accounts)

✅ **Expected:** Redirected to student dashboard.

### Test 5.2 — Student self-signup

Optional: Test that students can register themselves.

1. Logout
2. Go to **http://localhost:3000/student/signup**
3. Fill in:
   - **Student ID:** `S1003`
   - **Full Name:** `Self Registered`
   - **Password:** `student123` (min 6 characters)
4. Click **Sign Up**

✅ **Expected:** Auto-logged in, redirected to `/student`.

❌ **Try:** Password shorter than 6 chars → should be rejected.

### Test 5.3 — Enroll in a course

1. Log back in as `S1001` / `S1001`
2. Go to **http://localhost:3000/student/enroll**
3. Find your test course (created in Phase 3)
4. Select a class (radio button — e.g., `Class 1`)
5. Click **Enroll Now**

✅ **Expected:** Success message; status changes to "✓ Enrolled".

❌ **Try to enroll again** in a different class of the same course → should be blocked (already enrolled).

### Test 5.4 — View enrolled courses

1. Go to **http://localhost:3000/student/courses**

✅ **Expected:** Your enrolled course is listed with lecturer name.

### Test 5.5 — Submit an assignment

1. Go to **http://localhost:3000/student/assignments**

✅ **Expected:** You see `Assignment 1` (created in Test 3.5) under "Pending".

2. Click on the assignment
3. Fill in:
   - **Comment:** `Here is my submission`
   - **Upload File:** any small PDF (or skip and paste a fake URL)
4. Click **Submit**

✅ **Expected:** Assignment moves to "Submitted" section.

❌ **Try:** Upload a file > 10MB → should be rejected.

### Test 5.6 — View announcements

1. Go to **http://localhost:3000/student/announcements**

✅ **Expected:** The announcement posted in Test 3.4 is visible.

---

## PHASE 6 — FYP End-to-End Workflow

This is the **most important** test sequence. It involves multiple roles, so use multiple browser windows.

> **Roles needed:**
> - Window A: Admin (`ADMIN001`)
> - Window B: Supervisor (`SV1`)
> - Window C: Assessor (`A1`)
> - Window D: Student (`S1001`)

### Step 6.1 — Coordinator creates an FYP project

**(Window A — Admin)**

1. Go to **http://localhost:3000/coordinator/projects**
2. Click **Create Project** (or similar button)
3. Fill in:
   - **Student:** `Student Alpha (S1001)`
   - **Supervisor:** `Supervisor One`
   - **Title:** `Smart Attendance System`
   - **Description:** `An IoT-based attendance system`
   - **Phase:** `FYP1`
4. Click **Save**

✅ **Expected:** Project appears in the project list.

### Step 6.2 — Coordinator creates a presentation schedule

**(Window A — Admin)**

1. Go to **http://localhost:3000/coordinator/schedule**
2. Click **+ Add Schedule**
3. Fill in:
   - **Title:** `FYP1 Presentation Slot 1`
   - **Date/Time:** (pick a future date)
   - **Venue:** `Room A101`
4. Click **Save**

✅ **Expected:** Schedule appears in the list.

### Step 6.3 — Coordinator assigns an assessor

**(Window A — Admin)**

1. Go to **http://localhost:3000/coordinator/assign**
2. Select:
   - **Project:** `Smart Attendance System (S1001)`
   - **Assessor:** `Assessor One`
   - **Schedule:** `FYP1 Presentation Slot 1`
3. Click **Assign**

✅ **Expected:** Assignment row appears.

### Step 6.4 — Supervisor reviews the project

**(Window B — Supervisor SV1)**

1. Go to **http://localhost:3000/supervisor/projects**

✅ **Expected:** You see `Smart Attendance System` assigned to `Student Alpha`.

2. Click into the project → you should see project details
3. Click **Approve** (or change status to "approved")

✅ **Expected:** Status updates to "Approved".

### Step 6.5 — Student submits FYP report/slides

**(Window D — Student S1001)**

1. Go to **http://localhost:3000/student/fyp** (or similar)

✅ **Expected:** You see the assigned FYP project.

2. Upload a report PDF and slides PDF (use any small PDF files)
3. Submit

✅ **Expected:** Submission status = "Pending".

### Step 6.6 — Supervisor approves submission and leaves feedback

**(Window B — Supervisor)**

1. Refresh `/supervisor/projects/{projectId}`
2. You should see the new submission(s)
3. Click **Review** → leave feedback like `Good work, address Section 3.`
4. Click **Approve**

✅ **Expected:** Submission status = "Approved". Student sees feedback in their window.

### Step 6.7 — Supervisor fills detailed assessments

**(Window B — Supervisor)** — Test each form:

#### 6.7a — Progress Defense Form
1. Go to `/supervisor/projects/{id}/progress-defense-form`
2. Fill in 5 rubric criteria (1–10 each)
3. Save

✅ **Expected:** Weighted total ≤ 20 displayed.

#### 6.7b — Progress Week 14 Form
1. Go to `/supervisor/projects/{id}/progress-week14-form`
2. Fill in 5 criteria
3. Save

✅ **Expected:** Weighted total ≤ 21 displayed.

#### 6.7c — Presentation Assessment Form
1. Go to `/supervisor/projects/{id}/presentation-form`
2. Fill all 17 criteria (1–10 each)
3. Save

✅ **Expected:** Weighted total ≤ 35.

#### 6.7d — Report Assessment Form
1. Go to `/supervisor/projects/{id}/report-form`
2. Fill all 13 criteria
3. Save

✅ **Expected:** Weighted total ≤ 35.

### Step 6.8 — Supervisor enters component marks

**(Window B)**

1. Go back to the project detail page
2. Enter component marks (Progress, Presentation, Report, Paper) — each 0–100
3. Save

✅ **Expected:** Saved confirmation. Marks now stored in `SupervisorMark`.

### Step 6.9 — Assessor fills assessments and marks

**(Window C — Assessor A1)**

1. Go to **http://localhost:3000/assessor/projects**

✅ **Expected:** You now see `Smart Attendance System` (because coordinator assigned you).

2. Click into project
3. Fill the **Presentation Form** (17 criteria)
4. Fill the **Report Form** (13 criteria)
5. Enter component marks (Presentation, Report)
6. Leave feedback

✅ **Expected:** All saved.

---

## PHASE 7 — Coordinator Final Marks

**(Window A — Admin/Coordinator)**

### Step 7.1 — Enter coordinator progress mark

1. Go to **http://localhost:3000/coordinator/marks**
2. Find the project for `S1001`
3. In **Tab 1 (Progress Entry)**, enter `Progress Mark: 25`
4. Select **Phase: FYP1**
5. Click **Save** on that row

✅ **Expected:** Mark saved.

### Step 7.2 — Generate final marks

1. Still on `/coordinator/marks`
2. Verify the **Supervisor Weight: 40** and **Assessor Weight: 60** (defaults)
3. Click **Generate Final Marks**
4. Confirm the dialog

✅ **Expected:** Switch to **Tab 2 (Results)** automatically. You should see:
- Final Mark (calculated)
- Grade (e.g., A, B+, C)
- Breakdown (e.g., "Prog 25.0 · Pres 25.55 · Rep 22.5")
- Status: **Draft**

### Step 7.3 — Publish results

1. Click **Publish** (per-row) or **Publish All Results**

✅ **Expected:** Status changes to **Published**.

### Step 7.4 — Student verifies published mark

**(Window D — Student S1001)**

1. Refresh the FYP page

✅ **Expected:** Student now sees the final mark and grade.

---

## PHASE 8 — Edge Cases & Security Checks

These tests confirm the app rejects invalid actions.

### Test 8.1 — Wrong role access

| Try this | While logged in as | Expected |
|----------|--------------------|----------|
| Visit `/admin/manage` | Student | Redirect to login or 403 |
| Visit `/supervisor/projects` | Student | Redirect to `/student/login` |
| Visit `/coordinator/marks` | Non-admin lecturer | Redirect to `/` |
| Visit `/student/courses` | Lecturer | Redirect to `/login` |
| Visit `/admin/manage` | Logged out | Redirect to `/login` |

### Test 8.2 — Form validation

| Action | Expected error |
|--------|----------------|
| Signup with password < 6 chars | "Password too short" or similar |
| Signup with duplicate Staff ID | "Staff ID already exists" |
| Signup with duplicate Student ID | "Student ID already exists" |
| Login with wrong password | "Invalid credentials" |
| Bootstrap admin with wrong secret | 401/403 error |
| Save GPA scale with no rows | Validation error |
| Save GPA scale with min > max | Validation error |
| Lecturer selects 0 courses to teach | Error: must pick 1–3 |
| Lecturer selects 4+ courses | Error: max 3 |
| Student enrolls in already-enrolled course | Error message |
| Student tries to enroll in a full class (50/50) | Error: class full |
| Submit assignment with > 10 MB file | Error: file too large |
| Submit assignment after due date | Should show "Overdue" but still allow submission (or block, depending on logic — verify behavior) |

### Test 8.3 — Cross-supervisor access

1. Create a second supervisor (`SV2`)
2. Log in as `SV2`
3. Try to view a project that belongs to `SV1`
4. ✅ **Expected:** Should not be able to grade or modify it

### Test 8.4 — Session expiry

1. Log in as any user
2. Wait > 12 hours (or manually edit cookie expiry in browser DevTools)
3. Try to perform an action

✅ **Expected:** Redirected to login.

### Test 8.5 — Logout

1. Click logout
2. ✅ Session cookie cleared, redirected to `/welcome` or login
3. Try to visit `/admin/manage` directly via URL
4. ✅ Should redirect to login

### Test 8.6 — File upload security

1. As a student, try to upload a file with a strange extension (e.g., `.exe`)
2. ✅ **Expected:** Rejected or restricted to safe types (`.pdf`, `.doc`, `.docx`, `.txt`, `.zip`, `.rar`, `.jpg`, `.jpeg`, `.png`)

### Test 8.7 — Direct API calls without auth

Open terminal and run:
```bash
curl http://localhost:3000/api/admin/lecturers
```
✅ **Expected:** 401 Unauthorized (no session cookie sent).

---

---

## PHASE 9 — Security Testing with OWASP ZAP

> ⚠️ **IMPORTANT — Legal & Ethical Notice**
> - **Local hosting:** You can scan freely. The app is on your own machine.
> - **Vercel hosting:** Only scan a site **you own** and that **you control**. Vercel's [Acceptable Use Policy](https://vercel.com/legal/abuse) prohibits load testing or scanning sites you don't own. For aggressive scans on Vercel, **email security@vercel.com first** to whitelist your IP, otherwise you may get your account suspended or rate-limited.
> - **Never** scan a public production site that real users depend on without prior notice.

OWASP ZAP (Zed Attack Proxy) is a free security scanner. We'll cover two workflows:
- **9A** — Scan a locally hosted app (`http://localhost:3000`)
- **9B** — Scan a Vercel-hosted app (`https://your-app.vercel.app`)

### Step 9.0 — Install OWASP ZAP

1. Go to **https://www.zaproxy.org/download/**
2. Download the installer for your OS (macOS / Windows / Linux)
3. Install and launch ZAP
4. When prompted "Do you want to persist this session?" → choose **No, I do not want to persist this session** (for testing)

---

### 9A — Testing LOCAL HOSTING (http://localhost:3000)

#### Step 9A.1 — Make sure the app is running

In a terminal:
```bash
cd /Users/user/Downloads/unikl2
npm run dev
```
Confirm it's at **http://localhost:3000**.

#### Step 9A.2 — Quick Automated Scan (good first pass)

1. In ZAP's main window, find the **Quick Start** tab
2. Click **Automated Scan**
3. In **URL to attack**, enter: `http://localhost:3000`
4. ☑ Tick **Use traditional spider**
5. ☑ Tick **Use ajax spider** (this app is Next.js, lots of client-side rendering)
6. Click **Attack**

✅ **Expected:** ZAP will:
1. Spider the site (discover URLs)
2. Run an Active Scan (try common attacks)
3. Show alerts in the **Alerts** tab (left bottom)

⏱ **Time:** 5–20 minutes depending on machine.

#### Step 9A.3 — Set up ZAP as a browser proxy (for authenticated scan)

The Quick Scan only finds public pages. To scan admin/student pages, ZAP needs your session cookie.

1. In ZAP, go to **Tools → Options → Local Proxies**
2. Confirm:
   - **Address:** `localhost`
   - **Port:** `8080`
3. In your browser (use **Firefox** — easiest), go to **Settings → Network Settings → Manual proxy**
   - **HTTP Proxy:** `127.0.0.1` **Port:** `8080`
   - ☑ Use this proxy for all protocols
   - **No proxy for:** (leave blank — we want to capture localhost too)
4. Browse to **http://localhost:3000/login** and log in as admin

✅ **Expected:** Every page you visit appears in ZAP's **Sites** tree on the left.

> 💡 If localhost traffic is NOT being captured, remove `localhost` and `127.0.0.1` from the "No proxy for" field in Firefox.

#### Step 9A.4 — Walk through all role flows while proxied

With ZAP capturing, manually do:
1. Login as `ADMIN001`, visit `/admin/manage`, `/admin/gpa-scale`, `/coordinator/marks`
2. Logout, login as `SV1`, visit `/supervisor/projects`, fill a form
3. Logout, login as student `S1001`, visit `/student/enroll`, submit an assignment

This populates ZAP's **Sites** tree with all real authenticated URLs.

#### Step 9A.5 — Run Active Scan on the captured tree

1. In ZAP **Sites** tab, right-click `http://localhost:3000`
2. Choose **Attack → Active Scan**
3. In the dialog:
   - **Policy:** Default Policy
   - ☑ Recurse
   - ☑ Show advanced options → **Input Vectors** tab → ensure URL Query, Form Data, JSON are ticked
4. Click **Start Scan**

⏱ **Time:** 15–60 minutes.

✅ **Expected:** Findings populate in **Alerts** tab, sorted by severity (Red = High, Orange = Medium, Yellow = Low, Blue = Info).

#### Step 9A.6 — Configure authentication context (so ZAP can stay logged in)

Many endpoints (`/api/admin/*`) require a session cookie. To let ZAP test these:

1. In **Sites** tree, right-click `localhost:3000` → **Include in Context → Default Context**
2. Right-click context → **Properties**
3. Go to **Authentication** → choose **Form-based authentication**
4. **Login URL:** `http://localhost:3000/api/auth/login`
5. **Login Request POST data:** `staffId=ADMIN001&password=admin123`
6. **Logged in indicator regex:** `\"ok\":true` (response from successful login)
7. **Logged out indicator:** `\"error\"`
8. Go to **Users** → **Add User**
   - Name: `admin`
   - Staff ID: `ADMIN001`
   - Password: `admin123`
9. Set the user as **Enabled**
10. Right-click the site → **Attack → Active Scan** → pick **User: admin**

✅ **Expected:** ZAP attacks authenticated endpoints too.

#### Step 9A.7 — Specific tests to focus on (this app)

In ZAP's **Sites** tree, target these requests for **Manual Request Editor** (Tools → Manual Request Editor):

| Target | What to fuzz | Looking for |
|--------|--------------|-------------|
| `POST /api/auth/login` | `password` field with SQL payloads (`' OR 1=1--`) | SQL injection (unlikely with Prisma, but verify) |
| `POST /api/student/upload` | Upload `.php`, `.html`, `.svg` with `<script>` | Unrestricted file upload, stored XSS |
| `POST /api/announcements` | `content` field with `<script>alert(1)</script>` | Stored XSS in announcements |
| `POST /api/assignments` | `title` and `description` with HTML/JS | Stored XSS |
| `GET /api/admin/lecturers` (with student cookie) | Privilege escalation | Should return 403/401 |
| `PUT /api/admin/students/{id}` | Change `id` to another student's ID | IDOR (insecure direct object reference) |
| `POST /api/auth/bootstrap-admin` | Try without `secret` or with wrong secret | Auth bypass |

#### Step 9A.8 — Generate report

1. **Report → Generate Report**
2. Format: **HTML**
3. Select alerts to include (usually all)
4. Save to a folder

✅ **You now have a full security report for the local app.**

---

### 9B — Testing VERCEL HOSTING (https://your-app.vercel.app)

> ⛔ **Before you start:** Confirm you own the deployment. If this is a shared/production deployment with real users:
> - Notify your team
> - Do scans during low-traffic hours
> - Email `security@vercel.com` to whitelist your scanner IP (Vercel may block aggressive scans as DoS)
> - **Never** run Active Scan on a production deployment without backups

#### Step 9B.1 — Identify your Vercel URL

After deploying, Vercel gives you a URL like `https://unikl-fyp.vercel.app`. Use this as `<TARGET>` below.

#### Step 9B.2 — Spider only (safe first step)

1. In ZAP, **Quick Start → Manual Explore**
2. **URL to explore:** `https://<TARGET>`
3. Click **Launch Browser** → ZAP opens a pre-proxied Firefox
4. Browse the app manually (visit `/welcome`, `/login`, `/student/login`)

✅ **Expected:** URLs appear in **Sites** tree. No attacks have run yet — this is safe.

#### Step 9B.3 — Passive Scan (always safe)

ZAP automatically passive-scans every URL you visit. Check the **Alerts** tab.

Common passive findings to look for on Vercel:
- ✅ **Strict-Transport-Security** header → Vercel adds this by default
- ❓ **Content-Security-Policy** header → likely missing unless configured in `next.config.ts`
- ❓ **X-Frame-Options** → needs explicit config
- ❓ **Cookie security flags** → check that `unikl_session` is `Secure; HttpOnly; SameSite=Lax`

#### Step 9B.4 — Authenticated Manual Exploration

1. Click **Launch Browser** in Manual Explore
2. Log in as each role (admin, supervisor, student)
3. Visit every page you tested in Phases 2–7
4. Submit forms (use safe payloads, **not** XSS strings on a live site)

This populates ZAP's tree with real authenticated requests.

#### Step 9B.5 — Active Scan — DO NOT JUST CLICK ATTACK

If you must run an Active Scan on Vercel:

1. **Configure scan policy first**: Analyse → Scan Policy Manager → New
2. Set thresholds for ALL plugins to **Low** or **Medium** (not High)
3. Set strengths to **Low** (sends fewer requests)
4. Right-click target → **Attack → Active Scan**
5. ☑ Show advanced options → **Technology** tab → uncheck OS-level techs (Linux, MacOS) — they don't apply
6. **Speed**: under **Options → Active Scan**, set **Concurrent scanning threads per host** to `1` and **Delay between requests** to `1000ms`
7. Start scan

⏱ **Time:** 1–4 hours on Vercel (slower because of rate limiting, network latency, and edge functions cold-starts).

#### Step 9B.6 — Vercel-specific things to check

| Item | How to test | Why it matters |
|------|-------------|----------------|
| Preview deployment URLs | Try guessing `https://unikl-fyp-git-feature-x.vercel.app` | Preview URLs may leak internal/staging data |
| Environment variables | View page source for accidental `NEXT_PUBLIC_*` leaks | Any `NEXT_PUBLIC_` env var is shipped to the client |
| Source maps | Visit `/_next/static/chunks/*.js.map` directly | Source maps in production can expose source code |
| API rate limiting | Send 100 requests/sec to `/api/auth/login` | Vercel's free tier has no built-in auth rate limit — your app needs to add one |
| Function timeouts | Upload very large file | Vercel hobby = 10s, Pro = 60s timeout |
| `BOOTSTRAP_SECRET` env var | Try the default `unikl-bootstrap-2026` in production | If the env var wasn't overridden in Vercel settings, anyone can create an admin |
| Cookie domain | Inspect `unikl_session` cookie | Must be `Secure` flag set on HTTPS |
| Database access | The app uses SQLite (`dev.db`) | **SQLite doesn't work on Vercel serverless** — you should have migrated to Postgres/Turso/Neon. Verify which DB is live. |

#### Step 9B.7 — Verify the bootstrap-admin endpoint is locked

On a fresh Vercel deployment, this curl should FAIL:
```bash
curl -X POST https://<TARGET>/api/auth/bootstrap-admin \
  -H "Content-Type: application/json" \
  -d '{"secret":"unikl-bootstrap-2026","staffId":"HACKER","name":"Hacker","password":"123456"}'
```

✅ **Expected on a secure deployment:** `401` (because `BOOTSTRAP_SECRET` env var was changed) OR `409` (admin already exists).

❌ **If it returns `200`/`{"ok":true}`:** CRITICAL bug. The default secret is hardcoded. Set `BOOTSTRAP_SECRET` in Vercel **Project Settings → Environment Variables** to a strong random value.

#### Step 9B.8 — Generate report

Same as Step 9A.8.

---

## PHASE 10 — Security Testing with Burp Suite

> Burp Suite (Community Edition is free) is an interactive HTTP proxy. Better than ZAP for **manual** request tampering and authentication testing.

### Step 10.0 — Install Burp Suite Community Edition

1. Go to **https://portswigger.net/burp/communitydownload**
2. Download for your OS
3. Install and launch
4. Choose **Temporary Project → Use Burp defaults → Start Burp**

---

### 10A — Testing LOCAL HOSTING

#### Step 10A.1 — Configure browser proxy

Burp's built-in browser is the easiest path.

1. In Burp, go to the **Proxy** tab → **Open Browser**
2. A pre-configured Chromium opens
3. Visit `http://localhost:3000` in that browser
4. Go back to Burp → **Proxy → HTTP history**

✅ **Expected:** Every request you made is logged here.

> **Alternative — use your own browser:**
> - In Firefox: Proxy = `127.0.0.1:8080`
> - Install Burp's CA cert: visit `http://burp` in the proxied Firefox → download CA → import as trusted in Firefox → Settings → Privacy → Certificates → Authorities

#### Step 10A.2 — Capture login request

1. In Burp, go to **Proxy → Intercept** → click **Intercept is off** to turn it **ON**
2. In the Burp browser, go to `http://localhost:3000/login`
3. Enter `ADMIN001` / `admin123` and click Login
4. Burp will pause the request — you see it in **Intercept** tab
5. Right-click → **Send to Repeater** (Ctrl+R)
6. Right-click → **Send to Intruder** (Ctrl+I)
7. Click **Forward** to continue

#### Step 10A.3 — Test brute-force protection (Intruder)

1. Open **Intruder** tab → the login request is there
2. Click **Positions** → clear all markers
3. Highlight just the password value (e.g., `admin123`) and click **Add §** so it becomes `§admin123§`
4. Click **Payloads** tab
5. **Payload type:** Simple list
6. Paste 20 common passwords:
   ```
   password
   123456
   admin
   admin123
   letmein
   qwerty
   password123
   welcome
   abc123
   ...
   ```
7. Click **Start attack**

✅ **Expected on a secure app:** After ~5 failed attempts, server returns 429 (rate-limited) or starts adding delay.

❌ **If all 20 return 200/400 quickly:** No brute-force protection. **Bug.**

#### Step 10A.4 — Privilege escalation test (Repeater)

1. Log in as **student** `S1001` in the Burp browser
2. Capture any authenticated student request (e.g., `GET /api/student/assignments`)
3. Send to **Repeater**
4. In Repeater, change the URL to `GET /api/admin/lecturers`
5. Click **Send**

✅ **Expected:** `401 Unauthorized` or `403 Forbidden`.

❌ **If it returns 200 with lecturer data:** **CRITICAL bug** — privilege escalation.

Repeat for:
- `GET /api/admin/students`
- `POST /api/admin/students` (with new student data)
- `GET /api/coordinator/marks`
- `POST /api/coordinator/marks`
- `PUT /api/admin/lecturers/{any-id}`

#### Step 10A.5 — IDOR (Insecure Direct Object Reference) test

1. As `S1001`, view your own FYP project → capture `GET /api/student/fyp/project`
2. Send to **Repeater**
3. If the response includes a project ID, change it in the URL to another student's project ID
4. Resend

✅ **Expected:** `404` or `403`.

❌ **If you see another student's project data:** IDOR vulnerability.

Same test for:
- `PUT /api/admin/students/{id}` — change id to another user's id while logged in as that user
- `POST /api/assessor/marks` — try to grade a project not assigned to you
- `POST /api/supervisor/marks` — try to grade a student not under your supervision

#### Step 10A.6 — Stored XSS test

1. Log in as a lecturer
2. Capture `POST /api/announcements`
3. Send to **Repeater**
4. Change body to:
   ```json
   {"title":"XSS Test","content":"<img src=x onerror=alert('XSS')>","courseClassId":null}
   ```
5. Send → success?
6. Log in as a student in another browser → visit `/student/announcements`

✅ **Expected:** The `<img>` tag is rendered as **text**, no alert pops.

❌ **If an alert pops:** Stored XSS vulnerability. React typically escapes this by default — but if announcements use `dangerouslySetInnerHTML`, this fails.

Repeat for:
- Assignment `title` and `description`
- Course name (AI extract)
- User `name` fields

#### Step 10A.7 — Mass assignment test

1. As a regular lecturer, capture `PUT /api/admin/lecturers/{own-id}` if accessible, or signup
2. Add an `isAdmin` field to the JSON body that wasn't in the original form:
   ```json
   {"name":"Me","staffId":"LEC1","password":"x","isAdmin":true}
   ```
3. Resend

✅ **Expected:** `isAdmin` is ignored or request rejected.

❌ **If your account becomes admin:** Mass assignment vulnerability.

#### Step 10A.8 — File upload security

1. Login as student → capture `POST /api/student/upload`
2. Send to Repeater
3. Modify the file body to upload:
   - A `.php` file (e.g., `<?php system($_GET['cmd']); ?>`)
   - A `.html` file with `<script>alert(1)</script>`
   - A 100MB file (test size limit)
   - A file with `../../../etc/passwd` as filename (path traversal)

✅ **Expected:** All rejected with appropriate error.

#### Step 10A.9 — CSRF test

1. As admin, capture a state-changing request (e.g., `POST /api/admin/students`)
2. Inspect headers — is there a CSRF token? An `Origin` check?
3. In Repeater, **remove** any CSRF/Origin/Referer header
4. Resend

✅ **Expected:** Request rejected (because cross-site request shouldn't succeed).

❌ **If it succeeds:** App may be vulnerable to CSRF.

> Next.js with cookie sessions usually needs explicit CSRF protection — verify this with the dev.

#### Step 10A.10 — Session cookie inspection

1. In Burp **Proxy → HTTP history**, click any request after login
2. Look at the `Cookie` header for `unikl_session`
3. Decode it (it should look like `lecturer.{id}.{exp}.{signature}`)

Verify:
- ☑ Cookie has `HttpOnly` flag
- ☑ Cookie has `SameSite=Lax` (or Strict)
- ❌ `Secure` flag NOT set on localhost is OK
- ❌ The signature portion changes when you tamper with the ID portion → if it doesn't validate, that's a bug

Try **tampering with the cookie**:
1. Change the `id` portion to another lecturer's ID
2. Keep the same signature
3. Resend

✅ **Expected:** `401 Unauthorized` (signature mismatch).

---

### 10B — Testing VERCEL HOSTING

> ⚠️ Same legal warning as Phase 9B. Only test sites you own.

#### Step 10B.1 — Point Burp at your Vercel domain

The only real difference vs. local: you're hitting HTTPS, not HTTP.

1. Burp **Proxy → Open Browser**
2. Visit `https://<TARGET>`
3. Burp's CA is auto-trusted in the Burp browser

If using your own Firefox:
- Install Burp CA cert (visit `http://burp` while proxied → CA Certificate → import to Firefox)
- HTTPS will work without warnings

#### Step 10B.2 — Run all tests 10A.2 through 10A.10 against your Vercel URL

The procedures are identical — just substitute `https://<TARGET>` for `http://localhost:3000`.

#### Step 10B.3 — Additional Vercel-specific Burp tests

**Test 10B.3a — Discover preview deployment URLs**

1. In Burp **Target → Site map**, right-click your domain → **Engagement tools → Discover content**
2. Set discovery to look for common Vercel patterns:
   - `https://<project>-git-<branch>-<team>.vercel.app`
   - `https://<project>-<hash>-<team>.vercel.app`

Or use [Burp's Intruder](https://portswigger.net/burp/documentation/desktop/tools/intruder) with a wordlist of common branch names (`main`, `dev`, `staging`, `preview`).

✅ **Expected:** Preview URLs should be password-protected (Vercel offers this) OR not exist OR require authentication.

❌ **If a preview URL serves a dev version of the app publicly:** Information disclosure.

**Test 10B.3b — Check security headers**

1. Capture any GET request to `https://<TARGET>/`
2. In Repeater, send and inspect response headers

Required on Vercel production:
- ☑ `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`
- ☑ `X-Content-Type-Options: nosniff`
- ☑ `X-Frame-Options: DENY` or `SAMEORIGIN`
- ☑ `Content-Security-Policy: ...` (likely missing — flag if so)
- ☑ `Referrer-Policy: strict-origin-when-cross-origin`

If missing, add headers in [next.config.ts](next.config.ts):
```ts
async headers() {
  return [
    {
      source: "/(.*)",
      headers: [
        { key: "X-Frame-Options", value: "DENY" },
        { key: "X-Content-Type-Options", value: "nosniff" },
        { key: "Referrer-Policy", value: "strict-origin-when-cross-origin" },
      ],
    },
  ];
}
```

**Test 10B.3c — Check for sensitive `NEXT_PUBLIC_*` env leaks**

1. In Burp, view the HTML source of any page (`GET /`)
2. Search the response body for keywords:
   - `NEXT_PUBLIC_`
   - `process.env`
   - `API_KEY`, `SECRET`, `TOKEN`
3. Also check `/_next/static/chunks/*.js` files

✅ **Expected:** No production API keys/secrets visible. Only intentional `NEXT_PUBLIC_*` values (which are designed to be public).

**Test 10B.3d — Check session cookie has `Secure` flag**

On HTTPS, `Set-Cookie: unikl_session=...` MUST include `Secure`.

1. Capture login response
2. Look for `Set-Cookie` header

✅ **Expected:** `Secure; HttpOnly; SameSite=Lax`

❌ **If `Secure` is missing on HTTPS:** Bug — cookie could leak over HTTP downgrade.

**Test 10B.3e — Rate limit on auth endpoints**

Vercel free tier does NOT provide automatic auth rate-limiting. Test if the app has its own:

1. Send to **Intruder**: `POST /api/auth/login` with 100 wrong passwords
2. Throttle setting: 5 requests/second

✅ **Expected:** Server returns 429 after some threshold.

❌ **If 100 attempts all return 401 with no delay:** No rate limiting — add it (e.g., via Vercel Edge Middleware + Upstash Redis).

**Test 10B.3f — Verify bootstrap-admin is locked in prod**

```
POST https://<TARGET>/api/auth/bootstrap-admin
Content-Type: application/json

{"secret":"unikl-bootstrap-2026","staffId":"PWN","name":"Pwn","password":"pwned123"}
```

✅ **Expected:** 401 (wrong secret because env var was changed) or 409 (admin exists)

❌ **If 200:** CRITICAL — set `BOOTSTRAP_SECRET` in Vercel env vars immediately.

**Test 10B.3g — Check `/api/*` paths are not enumerable via 404**

Some Vercel deploys leak whether a route exists by returning different status codes (404 for non-existent vs 401 for protected).

1. Send `GET /api/admin/lecturers` → record status
2. Send `GET /api/admin/doesnotexist` → record status

If you get 401 vs 404, you've confirmed `/api/admin/lecturers` exists. Not severe but useful info for an attacker.

#### Step 10B.4 — Save your Burp project

**File → Save copy of project as** → `unikl-vercel-pentest-{date}.burp`

This saves all requests, responses, and findings for your report.

---

### Security Testing Summary Checklist

After running Phases 9 & 10, you should have results for:

- [ ] ZAP automated scan (local)
- [ ] ZAP authenticated active scan (local, all roles)
- [ ] ZAP passive scan (Vercel)
- [ ] ZAP manual exploration (Vercel)
- [ ] Burp login brute-force test
- [ ] Burp privilege escalation test (student → admin endpoints)
- [ ] Burp IDOR test (access other users' data)
- [ ] Burp stored XSS test (announcements, assignments)
- [ ] Burp mass assignment test (`isAdmin` flag)
- [ ] Burp file upload tests (.php, .html, oversized, path traversal)
- [ ] Burp CSRF test
- [ ] Burp session cookie tampering test
- [ ] Vercel preview URL discovery
- [ ] Vercel security headers check
- [ ] Vercel cookie `Secure` flag check
- [ ] Vercel auth rate limit test
- [ ] Vercel bootstrap-admin secret check
- [ ] No `NEXT_PUBLIC_*` secret leakage

### Severity Triage (use in your report)

| Severity | Examples |
|----------|----------|
| **Critical** | Auth bypass, RCE, full DB read/write as anonymous, default bootstrap secret in prod |
| **High** | Privilege escalation, stored XSS, IDOR exposing PII, weak password hashing |
| **Medium** | Missing rate limit, missing CSRF token, missing security headers, mass assignment |
| **Low** | Verbose error messages, missing HSTS preload, weak cookie SameSite |
| **Info** | Server version disclosure, missing X-Powered-By removal |

---

---

## PHASE 11 — Deploying to Vercel

> The project has been prepared for Vercel deployment. The original SQLite + local-filesystem setup **does not work** on Vercel's serverless platform. This phase walks you through the production deployment.

### What changed to make this Vercel-ready

| Area | Before (broken on Vercel) | After (works on Vercel) |
|------|----------------------------|--------------------------|
| Database | SQLite (`dev.db` file on disk) | **Neon Postgres** (cloud-hosted) |
| File uploads | Written to `/public/uploads/` | **Vercel Blob** (with local-filesystem fallback for dev) |
| Bootstrap secret | Hardcoded default `unikl-bootstrap-2026` | Now **required** as an env var — endpoint returns 500 if missing |
| Security headers | None | `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy` |
| Build command | Plain `next build` | `prisma generate && prisma migrate deploy && next build` |
| Prisma migrations | 11 SQLite migrations | Deleted — fresh Postgres baseline will be created on first deploy |
| Dependencies | `better-sqlite3`, `@prisma/adapter-better-sqlite3` | Removed; added `@vercel/blob` |

Files added/modified in this migration:
- [prisma/schema.prisma](prisma/schema.prisma) — `provider = "postgresql"`
- [src/lib/prisma.ts](src/lib/prisma.ts) — standard PG client, no SQLite adapter
- [src/lib/storage.ts](src/lib/storage.ts) — **NEW** unified upload helper (Blob in prod, local in dev)
- [src/app/api/student/upload/route.ts](src/app/api/student/upload/route.ts) — uses `uploadFile()`
- [src/app/api/student/fyp/submit/route.ts](src/app/api/student/fyp/submit/route.ts) — uses `uploadFile()`
- [src/app/api/auth/bootstrap-admin/route.ts](src/app/api/auth/bootstrap-admin/route.ts) — secret no longer defaulted
- [next.config.ts](next.config.ts) — security headers + Blob image patterns
- [package.json](package.json) — new deps, `postinstall: prisma generate`, `vercel-build` script
- [vercel.json](vercel.json) — **NEW** function timeouts (uploads 30s, AI extract 60s)
- [.vercelignore](.vercelignore) — **NEW** excludes `dev.db`, local uploads
- [.env.example](.env.example) — **NEW** template for required env vars
- [.gitignore](.gitignore) — also ignores `*.db`

### Step 11.1 — Verify locally that the migration didn't break anything

Before deploying, do these on your laptop:

```bash
# 1) Install the new dependency set (removes sqlite, adds @vercel/blob)
rm -rf node_modules package-lock.json
npm install

# 2) You DON'T need Postgres locally to verify the build. Skip migrations
#    for now and just confirm next build succeeds:
npx prisma generate
npm run build
```

✅ **Expected:** `Compiled successfully` and a `.next/` folder appears.

❌ **If you get** `Error: Environment variable not found: DATABASE_URL` during build, set a placeholder:
```bash
DATABASE_URL="postgresql://placeholder" npm run build
```
This is fine — Prisma only needs the URL at runtime for the client; the build step just needs the schema.

> ⚠ **Local dev with Postgres:** If you want to keep running `npm run dev` locally against a Postgres DB, the easiest path is to use the **same Neon database** (Neon is fine for dev) or run Postgres in Docker:
> ```bash
> docker run --name unikl-pg -e POSTGRES_PASSWORD=local -p 5432:5432 -d postgres:16
> ```
> Then set `DATABASE_URL="postgresql://postgres:local@localhost:5432/postgres"` in `.env`.

### Step 11.2 — Create a Neon Postgres database

1. Go to **https://console.neon.tech/** and sign up (free tier — no credit card)
2. Click **Create Project**
3. Settings:
   - **Project name:** `unikl-fyp`
   - **Postgres version:** 16 (default)
   - **Region:** closest to your users (e.g., `aws-ap-southeast-1` for Malaysia)
4. After creation, click **Connection Details**
5. Make sure **Pooled connection** is selected
6. Copy the connection string — it looks like:
   ```
   postgresql://neondb_owner:xxx@ep-xxx-pooler.ap-southeast-1.aws.neon.tech/neondb?sslmode=require
   ```
7. **Also copy the non-pooled (direct) URL** (toggle off "Pooled") — needed for migrations. It looks identical but without `-pooler`.

✅ **Save both URLs somewhere safe.** You'll paste them into Vercel.

### Step 11.3 — Create a Vercel Blob store

1. Go to **https://vercel.com/dashboard** (sign up if you haven't)
2. Click **Storage** in the top nav → **Create Database** → choose **Blob**
3. Name it `unikl-uploads` → **Create**
4. Click into the store → **.env.local** tab → copy the `BLOB_READ_WRITE_TOKEN` value (starts with `vercel_blob_rw_…`)

✅ **Save the token.**

### Step 11.4 — Push your project to GitHub

Vercel deploys from a Git repo.

```bash
cd /Users/user/Downloads/unikl2

# Initialize if not already a git repo
git init
git add .
git commit -m "Vercel-ready migration: Neon Postgres + Vercel Blob"

# Create a new GitHub repo (via web UI or gh CLI), then:
git remote add origin https://github.com/YOUR_USERNAME/unikl-fyp.git
git branch -M main
git push -u origin main
```

> ✅ The updated [.gitignore](.gitignore) excludes `dev.db`, `*.db`, `node_modules`, `.env*` (except `.env.example`), and `/public/uploads`. Double-check that `.env` is NOT in the commit (`git status`).

### Step 11.5 — Import to Vercel

1. https://vercel.com/new
2. Click **Import** next to your GitHub repo
3. **Framework Preset:** Next.js (auto-detected)
4. **Root Directory:** `./` (default)
5. **Build Command:** Vercel will read it from `vercel.json` automatically:
   `prisma generate && prisma migrate deploy && next build`
6. **Install Command:** `npm install` (default)
7. **STOP — don't click Deploy yet.** Expand **Environment Variables** first.

### Step 11.6 — Set environment variables in Vercel

Add **all four** of these in the **Environment Variables** section (select all three environments: Production, Preview, Development):

| Variable | Value | How to generate / get |
|----------|-------|-----------------------|
| `DATABASE_URL` | Neon **pooled** connection string | From Step 11.2 |
| `DIRECT_URL` | Neon **non-pooled** (direct) string | From Step 11.2 — **optional but recommended** for migrations |
| `AUTH_SECRET` | A 96-char random hex string | Run: `node -e "console.log(require('crypto').randomBytes(48).toString('hex'))"` |
| `BOOTSTRAP_SECRET` | A different 96-char random hex string | Same command as above — generate a NEW one |
| `BLOB_READ_WRITE_TOKEN` | The `vercel_blob_rw_…` value | From Step 11.3 |

> 🔐 **CRITICAL:** Do NOT reuse `unikl-bootstrap-2026` (the old default) for `BOOTSTRAP_SECRET`. The endpoint will now refuse if the env var is missing, but if you set the default value here you reintroduce the original vulnerability.

> 💡 If you set `DIRECT_URL`, also add this to `prisma/schema.prisma` (optional, only if you want `prisma migrate` to use direct connection):
> ```prisma
> datasource db {
>   provider  = "postgresql"
>   url       = env("DATABASE_URL")
>   directUrl = env("DIRECT_URL")
> }
> ```

### Step 11.7 — Deploy

1. Click **Deploy**
2. Watch the build log. Expected sequence:
   - `npm install`
   - `prisma generate` (postinstall hook)
   - `prisma migrate deploy` (creates all tables in Neon — first time only, creates the baseline)
   - `next build` (compiles Next.js app)

⏱ **Time:** 3–6 minutes for the first deploy.

✅ **Success looks like:** "Deployment Ready" + a URL like `https://unikl-fyp-xxx.vercel.app`

❌ **Common build failures:**

| Error | Fix |
|-------|-----|
| `Environment variable not found: DATABASE_URL` | You forgot to add it in Step 11.6 — go to Project Settings → Environment Variables |
| `P1001: Can't reach database server` | Check the DATABASE_URL is correct and includes `?sslmode=require` |
| `prisma migrate deploy` errors with `P3005` (non-empty DB) | Your Neon DB already has tables. Either drop them or change the command to `prisma db push --accept-data-loss` for first deploy only |
| `Module not found: '@vercel/blob'` | `npm install` didn't run — check Install Command is set to `npm install` |
| `Error: Read-only file system` at runtime | Means a code path still writes to disk outside `/tmp`. Should not happen after this migration — grep for `writeFile` and verify it's only in `src/lib/storage.ts` |

### Step 11.8 — Bootstrap the first admin on production

```bash
curl -X POST https://YOUR-APP.vercel.app/api/auth/bootstrap-admin \
  -H "Content-Type: application/json" \
  -d '{"secret":"YOUR_BOOTSTRAP_SECRET_VALUE","staffId":"ADMIN001","name":"Admin User","password":"a-strong-password"}'
```

✅ **Expected:** `{"ok":true,"adminId":"..."}`

❌ **If you get** `BOOTSTRAP_SECRET is not configured on the server` — go back to Step 11.6 and add the env var, then **Redeploy** (Vercel doesn't pick up env changes without a redeploy).

### Step 11.9 — Smoke test the production deployment

Run a quick subset of the Phase 1–7 tests against the live URL:

1. **Login** as `ADMIN001` at `https://YOUR-APP.vercel.app/login`
2. **Create a lecturer** via `/admin/manage`
3. **Create a student** via `/admin/manage`
4. **Log in as that student** → upload a tiny test file as an FYP submission
5. **Verify the file URL** in the resulting page is `https://*.public.blob.vercel-storage.com/...` (NOT `/uploads/...`)
6. **Click the link** to confirm the file downloads

✅ If all five pass, your deploy is production-ready.

### Step 11.10 — Verify nothing is broken (production checklist)

| Check | How to verify | Pass criteria |
|-------|---------------|---------------|
| Database connected | Visit `/welcome` then `/login`, try logging in | No 500 error; login succeeds |
| Bootstrap admin locked | `curl -X POST .../api/auth/bootstrap-admin -d '{"secret":"wrong",...}'` | Returns `401 Invalid secret` (not 500 about missing config) |
| Blob uploads work | Upload an FYP report as student | Returned `fileUrl` starts with `https://...blob.vercel-storage.com` |
| Blob files publicly accessible | Open the returned URL in a private browser tab | File downloads / displays |
| Security headers | `curl -I https://YOUR-APP.vercel.app/` | Headers include `x-frame-options: DENY`, `x-content-type-options: nosniff`, `referrer-policy: strict-origin-when-cross-origin` |
| Cookie has Secure flag | Inspect `unikl_session` cookie in DevTools after login | `Secure ✓` and `HttpOnly ✓` flags present |
| Old `/uploads/` paths return 404 | Visit `https://YOUR-APP.vercel.app/uploads/anyfile.pdf` | 404 (uploads dir no longer exists in deploy) |
| Function timeouts work | Upload a 9 MB file | Completes within 30s, no timeout |
| Production logs clean | Open Vercel dashboard → your project → **Logs** | No red errors during the smoke test |

### Step 11.11 — Set up a custom domain (optional)

1. Vercel dashboard → your project → **Settings → Domains**
2. Add your domain (e.g., `unikl-fyp.com`)
3. Follow the DNS instructions Vercel shows
4. Vercel auto-issues an HTTPS certificate
5. After DNS propagates (5–60 min), your app is live at the custom domain

### Step 11.12 — Subsequent deployments

After the initial setup, deployments are automatic:

- **Push to `main` branch** → triggers a **Production** deployment
- **Push to any other branch** → triggers a **Preview** deployment (gets its own URL like `unikl-fyp-git-feature-x.vercel.app`)
- **Pull request opened** → comment appears with preview URL

Each push runs `prisma migrate deploy` first, so any new migrations are applied automatically.

### Step 11.13 — Rolling back a bad deploy

1. Vercel dashboard → **Deployments** tab
2. Find the previous good deployment
3. Click the `⋯` menu → **Promote to Production**

⚠ **Note:** Rolling back code does NOT roll back database migrations. If a deploy ran a destructive migration, you must restore the DB from a Neon point-in-time backup (Neon dashboard → Backups).

### Vercel deployment troubleshooting cheat sheet

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| Build fails on `prisma migrate deploy` with `Can't reach database` | Wrong DATABASE_URL or Neon project paused | Verify URL in Vercel env vars; Neon free tier pauses after inactivity — visit Neon console to wake |
| Runtime error `BLOB_READ_WRITE_TOKEN is not defined` when uploading | Token env var not set | Add token in Vercel env vars, redeploy |
| Uploads succeed but file URL 404s | Blob token is for a different store | Regenerate token from the correct Blob store |
| Cookie not setting on first login | `AUTH_SECRET` not set | Add env var, redeploy, re-login |
| `503 Database connection error` after period of inactivity | Neon free tier autosleeps | Either upgrade Neon plan or accept ~3s wake delay on first request |
| Function times out on large file | Default timeout too short | Already configured in `vercel.json` (30s for uploads, 60s for AI extract). Hobby plan max is 60s; need Pro for 300s |
| `SELECT pg_advisory_lock` errors during migration | Multiple builds running concurrently | Cancel other builds; only one migration can run at a time |

### What you should know about ongoing costs

| Service | Free tier | Likely to need paid? |
|---------|-----------|----------------------|
| Vercel | Hobby: 100 GB bandwidth/month, 100 GB-hrs compute, 1 concurrent build | No, for typical university course usage |
| Neon Postgres | 0.5 GB storage, 100 hours compute/month | Possibly — if many students concurrently. Upgrade to Launch ($19/mo) for more |
| Vercel Blob | 1 GB storage, 1 GB outbound bandwidth/month | Likely yes once students start submitting FYP reports. $0.15/GB beyond |

> 💡 **Estimate:** 100 students × 20 MB FYP report = 2 GB. Plan for ~$1–5/month in Blob costs once active.

---

## Bug Reporting Template

When you find a bug, record it in this format:

```
TITLE: [Short summary]
ROLE: [Admin / Lecturer / Student / etc.]
URL: [Where it happened]
STEPS TO REPRODUCE:
1. ...
2. ...
3. ...
EXPECTED: [What should happen]
ACTUAL: [What actually happened]
SEVERITY: [Critical / High / Medium / Low]
SCREENSHOT/CONSOLE LOG: [If applicable]
BROWSER: [Chrome 120, Firefox 122, etc.]
```

---

## Quick Reference — URLs & Endpoints

### Public pages
- `/welcome` — Portal selection
- `/login` — Lecturer login
- `/signup` — Lecturer registration
- `/student/login` — Student login
- `/student/signup` — Student registration

### Admin pages
- `/admin/manage` — Manage lecturers & students
- `/admin/gpa-scale` — Configure grade thresholds

### Coordinator pages (admin-only)
- `/coordinator/projects` — Create/view FYP projects
- `/coordinator/assign` — Assign assessors
- `/coordinator/schedule` — Presentation schedules
- `/coordinator/marks` — Enter progress, generate & publish final marks

### Supervisor pages
- `/supervisor/projects` — List supervised students
- `/supervisor/projects/{id}` — Project detail
- `/supervisor/projects/{id}/presentation-form`
- `/supervisor/projects/{id}/report-form`
- `/supervisor/projects/{id}/progress-defense-form`
- `/supervisor/projects/{id}/progress-week14-form`

### Assessor pages
- `/assessor/projects` — Assigned projects
- `/assessor/projects/{id}` — Project detail
- `/assessor/projects/{id}/presentation-form`
- `/assessor/projects/{id}/report-form`

### Lecturer (teaching) pages
- `/my-courses` — Pick courses to teach (max 3)
- `/my-courses/{courseId}` — Course dashboard
- `/my-courses/{courseId}/announcements`
- `/my-courses/{courseId}/assignments`
- `/courses/ai-create` — AI-assisted course creation
- `/courses/{courseId}` — Course overview, CLO analysis

### Student pages
- `/student` — Dashboard (redirects to FYP)
- `/student/courses` — Enrolled courses
- `/student/enroll` — Browse & enroll in courses
- `/student/assignments` — View and submit assignments
- `/student/announcements` — Read announcements
- `/student/fyp` — FYP project & submissions

### Key API endpoints (for advanced testing with curl/Postman)

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/auth/bootstrap-admin` | POST | Create first admin |
| `/api/auth/login` | POST | Lecturer login |
| `/api/auth/signup` | POST | Lecturer signup |
| `/api/auth/logout` | POST | Logout |
| `/api/auth/me` | GET | Current user |
| `/api/student/auth/login` | POST | Student login |
| `/api/student/auth/signup` | POST | Student signup |
| `/api/admin/lecturers` | GET/POST | List/create lecturers |
| `/api/admin/students` | GET/POST | List/create students |
| `/api/admin/gpa-scale` | GET/PUT | GPA scale config |
| `/api/student/enroll` | POST | Self-enroll in class |
| `/api/student/assignments/submit` | POST | Submit assignment |
| `/api/student/upload` | POST | Upload file |
| `/api/assignments` | POST | Lecturer creates assignment |
| `/api/announcements` | POST | Lecturer posts announcement |
| `/api/coordinator/projects` | GET/POST/PATCH | FYP project CRUD |
| `/api/coordinator/assign` | POST/DELETE | Assessor assignment |
| `/api/coordinator/marks` | POST/PATCH | Generate/publish final marks |
| `/api/supervisor/marks` | POST | Supervisor enters marks |
| `/api/supervisor/presentation-assessment` | POST | Presentation rubric |
| `/api/assessor/marks` | POST | Assessor enters marks |

---

## Grading Scale Reference (built-in)

| Mark Range | Grade |
|-----------|-------|
| ≥ 90 | A+ |
| ≥ 80 | A  |
| ≥ 75 | A- |
| ≥ 70 | B+ |
| ≥ 65 | B  |
| ≥ 60 | B- |
| ≥ 55 | C+ |
| ≥ 50 | C  |
| ≥ 45 | C- |
| ≥ 40 | D  |
| < 40 | F  |

---

## FYP Mark Calculation Reference

### FYP1
Final mark = Progress (30) + Presentation (35) + Report (25)
- Progress: Supervisor 20% + Coordinator 10%
- Presentation: Supervisor 40% + Assessor 60%
- Report: Supervisor 100%

### FYP2
Final mark = Presentation (50) + Report (50)
- Default: Supervisor 50% / Assessor 50% for each
- Adjustable via weight inputs in coordinator page

---

## Sample Test Account Summary

After completing PHASE 2, you should have:

| Role | ID | Password |
|------|----|----|
| Coordinator/Admin | `ADMIN001` | `admin123` |
| Supervisor | `SV1` | `lecturer123` |
| Assessor | `A1` | `lecturer123` |
| Teaching Lecturer | `LEC1` | `lecturer123` |
| Student (assigned) | `S1001` | `S1001` |
| Student (unassigned) | `S1002` | `S1002` |
| Self-registered student | `S1003` | `student123` |

---

## Final Checklist

Before you call testing "complete", make sure you have:

- [ ] Created the first admin via bootstrap
- [ ] Created at least 1 supervisor, 1 assessor, 1 teaching lecturer
- [ ] Created at least 2 students (one with supervisor, one without)
- [ ] Tested student self-signup
- [ ] Created a course (via AI or manual)
- [ ] Tested course enrollment by a student
- [ ] Posted an announcement and verified the student sees it
- [ ] Created and submitted an assignment
- [ ] Run the full FYP workflow from project creation to mark publication
- [ ] Filled all 4 detailed assessment forms (presentation, report, progress defense, week 14)
- [ ] Generated and published final FYP marks
- [ ] Tested at least 5 edge cases from Phase 8
- [ ] Logged out and verified protected pages redirect to login

🎉 **If every box is ticked and no bugs blocked you, the app is ready for use.**

---

*Generated for the UniKL FYP & Course Management System — Next.js + Prisma + SQLite stack.*
