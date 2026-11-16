# MathRS — Project Handoff Document

## What you are building

A web application for Serbian high school students preparing for university entrance exams (prijemni ispit) in mathematics. The concept is LeetCode for Serbian math — structured topics, timed past exams, a progress tracker, and a global leaderboard.

The initial release is a web app. A mobile app (React Native) may follow in a future phase using the same Spring Boot backend, so keep the API clean and stateless from day one.

---

## Tech stack decisions

| Layer | Technology | Reason |
|---|---|---|
| Frontend | React (Vite) | Fast, component-based, easy to migrate to React Native later |
| Math rendering | KaTeX | Best-in-class web math rendering, fast, no dependencies |
| Backend | Spring Boot (Java) | JWT auth, REST API, CSV seeding at startup |
| Database | PostgreSQL | Relational, handles all user and progress data |
| ORM | Spring Data JPA + Hibernate | Standard Spring Boot DB access |
| CSV parsing | OpenCSV | Reads problem content at startup into memory |
| Auth | Spring Security + JWT | Stateless, token stored in browser localStorage |

---

## Architecture overview

The app has two distinct types of data:

**Static content (CSV files → in-memory)**
Problem text, equations (stored as LaTeX strings), answer choices, solutions, and explanations. Loaded from CSV files when Spring Boot starts and held in memory for the lifetime of the app. Never written to the database.

**Dynamic user data (PostgreSQL)**
User accounts, progress per problem, exam results, and leaderboard scores. These reference problems by ID only — the actual problem content stays in memory.

The two are linked by a stable problem ID (e.g. `KJ-001`). Once assigned, a problem ID must never change or be reused.

---

## Why LaTeX strings, not Unicode

Unicode characters like ² and √ look fine for simple expressions but break for anything requiring vertical layout — fractions, integrals with limits, square roots over full expressions, matrices. These appear constantly in Serbian entrance exam problems.

LaTeX strings like `\frac{-b}{a}` and `\int_0^2 2x\,dx` are passed directly to KaTeX which renders them as properly typeset math in the browser. Store all equations as LaTeX in the CSV files.

---

## CSV structure

### File layout

One CSV file per topic, stored in `src/main/resources/data/problems/`:

```
KJ.csv        Kvadratne jednačine
TRIG.csv      Trigonometrija
INT.csv       Integrali
NEJ.csv       Nejednačine
SKU.csv       Skupovi i logika
FUNK.csv      Funkcije
KOMP.csv      Kompleksni brojevi
AG.csv        Analitička geometrija
NIZ.csv       Nizovi i redovi
KOMB.csv      Kombinatorika
VER.csv       Verovatnoća
MAT.csv       Matrice i determinante
DER.csv       Derivacije
```

### CSV columns

```
id, topic, title, latex_equation, difficulty,
choice_a, choice_b, choice_c, choice_d, correct_index,
source_faculty, source_year,
theory_title, theory_text, theory_formula,
solution_step_1, solution_step_2, solution_step_3
```

### ID naming convention

Prefix by topic abbreviation, zero-padded number:

```
KJ-001   KJ-002   KJ-003    Kvadratne jednačine
TRIG-001 TRIG-002           Trigonometrija
INT-001                     Integrali
NEJ-001                     Nejednačine
SKU-001                     Skupovi i logika
FUNK-001                    Funkcije
KOMP-001                    Kompleksni brojevi
AG-001                      Analitička geometrija
NIZ-001                     Nizovi i redovi
KOMB-001                    Kombinatorika
VER-001                     Verovatnoća
MAT-001                     Matrice i determinante
DER-001                     Derivacije
```

### Sample CSV row

```
KJ-001,kvadratne_jednacine,Nađi zbir korena,3x^2 - 7x + 2 = 0,MEDIUM,
7/3,-7/3,2/3,3/7,0,
ETF Beograd,2022,
Vijèteove formule,Za ax^2 + bx + c = 0 sa korenima x_1 i x_2:,x_1 + x_2 = \frac{-b}{a} \quad x_1 \cdot x_2 = \frac{c}{a},
Identifikujemo koeficijente: a=3 b=-7 c=2,Vijètova formula: x_1 + x_2 = \frac{-(-7)}{3} = \frac{7}{3},Provjera: koreni su x=2 i x=1/3 zbir = 7/3
```

---

## PostgreSQL schema

Only dynamic user data lives in the database. Problem content is never stored here.

```sql
-- Users
CREATE TABLE users (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email        VARCHAR(255) UNIQUE NOT NULL,
  password     VARCHAR(255) NOT NULL,
  first_name   VARCHAR(100) NOT NULL,
  last_name    VARCHAR(100) NOT NULL,
  faculty      VARCHAR(100),
  dob          DATE,
  created_at   TIMESTAMP DEFAULT NOW()
);

-- Weak topics chosen during onboarding
CREATE TABLE user_weak_topics (
  user_id      UUID REFERENCES users(id) ON DELETE CASCADE,
  topic_id     VARCHAR(20) NOT NULL,
  PRIMARY KEY  (user_id, topic_id)
);

-- One row per problem attempt
CREATE TABLE user_progress (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id) ON DELETE CASCADE,
  problem_id   VARCHAR(20) NOT NULL,
  solved       BOOLEAN NOT NULL,
  attempts     INT DEFAULT 1,
  solved_at    TIMESTAMP
);

-- Full exam sittings
CREATE TABLE exam_attempts (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id      UUID REFERENCES users(id) ON DELETE CASCADE,
  faculty      VARCHAR(100) NOT NULL,
  year         INT NOT NULL,
  score        INT NOT NULL,
  total        INT NOT NULL,
  completed_at TIMESTAMP DEFAULT NOW()
);

-- Per-question answers within an exam sitting
CREATE TABLE exam_attempt_answers (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  exam_attempt_id UUID REFERENCES exam_attempts(id) ON DELETE CASCADE,
  problem_id      VARCHAR(20) NOT NULL,
  selected_index  INT,           -- null if skipped
  correct_index   INT NOT NULL,
  is_correct      BOOLEAN NOT NULL
);
```

Note: `exam_attempt_answers` stores the user's answer and the correct answer for every question in every exam sitting. This is what powers the post-exam review screen.

### Leaderboard query (no separate table needed)

```sql
SELECT
  u.id,
  u.first_name,
  u.last_name,
  u.faculty,
  COUNT(*) FILTER (WHERE p.solved = true)      AS solved,
  COUNT(*) FILTER (WHERE p.solved = true) * 2  AS xp,
  ROUND(
    COUNT(*) FILTER (WHERE p.solved = true) * 100.0
    / NULLIF(COUNT(*), 0)
  )                                             AS accuracy
FROM users u
LEFT JOIN user_progress p ON p.user_id = u.id
GROUP BY u.id
ORDER BY xp DESC;
```

### Activity grid query (365-day heatmap)

```sql
SELECT
  DATE(solved_at) AS day,
  COUNT(*)        AS count
FROM user_progress
WHERE user_id    = :userId
  AND solved     = true
  AND solved_at >= NOW() - INTERVAL '365 days'
GROUP BY DATE(solved_at)
ORDER BY day ASC;
```

---

## Spring Boot dependencies (pom.xml)

```xml
<dependencies>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
  </dependency>
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
  </dependency>
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
  <dependency>
    <groupId>com.opencsv</groupId>
    <artifactId>opencsv</artifactId>
    <version>5.9</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.5</version>
  </dependency>
  <dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.5</version>
  </dependency>
</dependencies>
```

---

## REST API endpoints

### Auth
```
POST   /api/auth/register     Register new user, returns JWT
POST   /api/auth/login        Login, returns JWT
```

### Problems (served from memory — no DB hit)
```
GET    /api/topics                         List all 13 topics with user progress
GET    /api/topics/{topicId}/problems      All problems for a topic
GET    /api/problems/{problemId}           Single problem by ID
GET    /api/problems/random                Random problem, optional ?topicId=KJ
```

### Exams (problem lists served from memory)
```
GET    /api/exams                          All available exams (faculty + year)
GET    /api/exams/{faculty}/{year}         All problems for a specific exam
POST   /api/exams/attempts                 Submit completed exam with all answers
GET    /api/exams/attempts/me              Current user's exam history
GET    /api/exams/attempts/{id}/review     Full review for one exam sitting —
                                           returns every question with the user's
                                           answer, the correct answer, and the
                                           full step-by-step explanation
```

### Progress
```
POST   /api/progress              Submit an answer { problemId, solved, attempts }
GET    /api/progress/me           All progress for current user
GET    /api/progress/me/activity  365-day activity grid data
GET    /api/progress/me/topics    Per-topic completion stats
```

### Leaderboard
```
GET    /api/leaderboard                    Global leaderboard
GET    /api/leaderboard?faculty=ETF        Filtered by faculty
GET    /api/leaderboard?period=month       Filtered by time period
```

### User
```
GET    /api/users/me              Current user profile
PUT    /api/users/me              Update profile
GET    /api/users/me/stats        Solved count, accuracy, XP, streak
```

---

## React (Vite) dependencies

```json
{
  "dependencies": {
    "react":               "latest",
    "react-dom":           "latest",
    "react-router-dom":    "latest",
    "katex":               "latest",
    "react-katex":         "latest",
    "axios":               "latest",
    "@reduxjs/toolkit":    "latest",
    "react-redux":         "latest"
  }
}
```

Store the JWT in `localStorage`. Use axios with a request interceptor that automatically attaches the JWT to every request header. Use Redux Toolkit for global state (current user, cached problem lists). Use React Router for navigation between pages.

---

## App screens

### FTUE (shown once on first visit after registration)

5-step onboarding flow:
1. Email + password (with strength bar and confirmation field)
2. First name + last name
3. Faculty selection (dropdown grouped by city — Belgrade, Novi Sad, Niš — plus "not sure" option)
4. Date of birth (with live age validation)
5. Weak topics (multi-select from all 13 topics, "none" is a valid choice)

After completion the user is taken to the main app and a flag is set in localStorage so the FTUE never shows again.

### Main app — navigation

Top navigation bar with the MathRS logo on the left, three nav links (Teme · Probaj ispit · Rang lista), and the user avatar on the right. Clicking the avatar opens the profile page. The avatar is the only entry point to the profile — it is not a nav link.

### Teme (Topics)

Grid of 13 topic cards. Each shows the topic name, difficulty badge (Easy / Medium / Hard in green / orange / red), a progress bar, and solved/total count. A green left border marks topics where the user has made solid progress. Clicking a card opens the Topic Detail page.

### Topic Detail

Breadcrumb showing teme / topic-name. Header with topic title, difficulty, total problems, and average accuracy. Below that a collapsible theory panel showing the key formula and explanation for the topic. Below that a table of all problems for the topic with columns for title, source (faculty + year), difficulty, and accuracy percentage. A "random problem" button in the header skips the list and opens a random unsolved problem. Clicking a table row opens the Exercise page.

### Exercise (LeetCode-style split layout)

Left panel: a theory callout box with the key formula relevant to this specific problem, the problem statement, and the LaTeX equation rendered by KaTeX. Below the equation, global stats for the problem (how many times solved, overall accuracy percentage).

Right panel: a problem info card (topic, difficulty, source faculty and year, XP reward), four answer choices (A / B / C / D), a submit button (disabled until an answer is selected), and a show solution button. After the user submits, the correct answer turns green, the wrong selection turns red, and a step-by-step solution panel expands showing each solution step with its own LaTeX equation. Navigation buttons at the bottom allow moving to the next or previous problem.

### Probaj ispit (Exam browser)

Grid of exam cards filtered by faculty (ETF Beograd, PMF Novi Sad, Mašinski, RGF, FON) and time period. Each card shows the faculty, year, number of questions, time limit, and the user's completion percentage from their last attempt. Clicking a card starts the exam.

### Exam (full-screen overlay)

The exam runs as a full-screen overlay that covers the entire app. It has a top bar showing the exam title, live score (X / 60 points), a countdown timer, and a submit button. On the left a sidebar shows a grid of numbered question buttons — grey for unanswered, filled for answered. A progress bar shows how many questions have been answered.

Each question shows the question number, topic tag, difficulty badge, the problem statement, a LaTeX equation if applicable, and four answer choices. Navigation buttons at the bottom move between questions. The user can revisit and change any answer before submitting.

### Exam results screen (shown immediately after submitting)

This replaces the full-screen exam overlay. It contains:

**Score summary at the top:**
- Large score number (e.g. 42 / 60)
- Number of correct, incorrect, and skipped questions
- Pass / fail status (ETF Beograd passing mark is 36 / 60)
- Time used

**Per-topic breakdown:**
A bar chart showing how many questions the user got right in each topic area that appeared in the exam. Each bar is colored green if above 60% or red if below. This immediately shows which areas need work.

**Full question review — one row per question:**
Each row shows the question number, the question title and LaTeX equation, the user's selected answer highlighted in red if wrong, the correct answer highlighted in green, and a collapsible explanation section. Clicking the explanation chevron expands the full step-by-step solution with each step rendered in KaTeX. Every single question is shown in this list, not just the wrong ones — users should be able to review correct answers too.

Two action buttons at the bottom: "Pregledaj ponovo" (retake the exam) and "Vežbaj slabe tačke" (which navigates to the topic detail pages for the topics the user scored lowest on).

### Rang lista (Leaderboard)

Table showing rank, avatar initials, name, faculty, XP, accuracy percentage, and problems solved. Filterable by faculty and time period (all time / this month). The current user's row is permanently highlighted in blue regardless of filter. Updates on page load.

### Profil (accessed via avatar)

Header showing the user's name, faculty, join date, current streak, and global rank. Below that a row of four stat cards (problems solved, accuracy, XP, active days). Then a GitHub-style 365-day activity grid where each cell is a small square colored by how many problems were solved that day (five levels: none, 1-2, 3-5, 6-9, 10+) with hover tooltips showing the exact date and count. Below the grid, a per-topic progress bar list showing completion percentage for all 13 topics.

---

## Post-exam review — implementation detail

When the user submits an exam, the frontend sends:

```json
POST /api/exams/attempts
{
  "faculty": "ETF Beograd",
  "year": 2023,
  "answers": [
    { "problemId": "KJ-001", "selectedIndex": 0 },
    { "problemId": "TRIG-001", "selectedIndex": 2 },
    { "problemId": "INT-001", "selectedIndex": null }
  ]
}
```

Spring Boot:
1. Looks up each `problemId` in the in-memory problem store
2. Compares `selectedIndex` to `correctIndex` for each problem
3. Writes one row to `exam_attempts` and one row per question to `exam_attempt_answers`
4. Returns the full review payload immediately in the response

The review response includes every question with the full problem object (title, equation, all choices, correct index, solution steps) merged with the user's answer. The frontend never needs to make a second request — the results screen is built entirely from the submission response.

If the user wants to review a past exam later, `GET /api/exams/attempts/{id}/review` reconstructs the same payload from the database by joining `exam_attempt_answers` with the in-memory problem store.

---

## Build order

Build in this sequence. Each step should be working and manually tested before moving to the next.

### Step 1 — Spring Boot project setup
Create a new Spring Boot project with the dependencies listed above. Configure `application.properties` with the PostgreSQL connection string, JWT secret, and JWT expiry. Verify the app starts and connects to the database with no errors.

### Step 2 — CSV parser and in-memory problem store
Create a `Problem` Java class (POJO) with fields matching every CSV column. Write a `CsvProblemLoader` class annotated with `@Component` that implements `CommandLineRunner`. On startup it reads every CSV file from `resources/data/problems/` using OpenCSV, parses each row into a `Problem` object, and stores the result in a `ProblemRepository` Spring bean as both a `List<Problem>` (for listing) and a `Map<String, Problem>` keyed by ID (for fast lookup). Write a test that loads the CSVs, asserts the correct total problem count, and spot-checks a known problem ID.

### Step 3 — Database schema and JPA entities
Run the full SQL schema above against your PostgreSQL instance. Create JPA entity classes for `User`, `UserWeakTopic`, `UserProgress`, `ExamAttempt`, and `ExamAttemptAnswer`. Create Spring Data JPA repository interfaces for each. Write a basic integration test that saves and retrieves a user.

### Step 4 — JWT authentication
Implement `POST /api/auth/register` — accepts email, password (bcrypt hash it before saving), first name, last name, faculty, date of birth, and weak topics. Saves the user and weak topics to the database. Returns a signed JWT. Implement `POST /api/auth/login` — validates credentials against the database, returns a JWT. Add a Spring Security JWT filter that reads the token from the `Authorization: Bearer` header on every request, validates it, and sets the `SecurityContext`. Test both endpoints manually with curl or Postman before proceeding.

### Step 5 — Problem endpoints
Implement the four problem endpoints. All data comes from the in-memory `ProblemRepository` — no database queries. For authenticated requests, enrich the response with the user's solved status by doing a lightweight database lookup on `user_progress`. Return the correct index only on the single-problem endpoint (not on list endpoints) to avoid leaking answers. Test that topic lists and individual problems return correctly.

### Step 6 — Progress endpoints
Implement `POST /api/progress` — writes a row to `user_progress`. If a row already exists for this user and problem, update `attempts` and `solved` rather than inserting a duplicate. Implement the activity grid endpoint using the SQL query above — it returns an array of `{ day, count }` objects for the last 365 days. Implement the per-topic stats endpoint. Test by submitting several answers and verifying the stats update.

### Step 7 — Exam endpoints
Implement `GET /api/exams/{faculty}/{year}` — returns all problems for that exam from memory, without correct answers. Implement `POST /api/exams/attempts` — processes the submission as described in the post-exam review section above, writes to `exam_attempts` and `exam_attempt_answers`, and returns the full review payload in the response. Implement `GET /api/exams/attempts/{id}/review` — reconstructs the review payload from the database for historical review. Test the full exam submission flow end to end.

### Step 8 — Leaderboard endpoint
Implement `GET /api/leaderboard` using the aggregation query. Add optional query parameters for faculty filtering and time period filtering. Test that ordering is correct and that the current user's row is always included even if they are outside the top results returned.

### Step 9 — React (Vite) project setup
Initialise a new React + Vite project. Install all dependencies listed above. Set up React Router with two route groups — a public group (FTUE, login) and a protected group (main app) that redirects to login if no JWT is in localStorage. Set up Redux Toolkit with slices for auth state and user profile. Set up an axios instance with a base URL pointing to the Spring Boot server and a request interceptor that attaches the JWT from localStorage to every request header.

### Step 10 — FTUE screens
Build the 5-step registration flow matching the HTML prototype (`mathrs-ftue.html`). Use KaTeX only if needed in this flow (it is not needed for FTUE). On completion of step 5, call `POST /api/auth/register`, store the returned JWT in localStorage, dispatch the auth action to Redux, and navigate to the main app. On app load, check localStorage for a JWT — if it exists and is not expired, skip FTUE and load the main app directly.

### Step 11 — Topic and exercise screens
Build the Topics page, Topic Detail page, and Exercise page. Use `react-katex` to render all LaTeX equations. Wire up `POST /api/progress` when the user confirms an answer. Match the layout and color system from the HTML prototype.

### Step 12 — Exam screen and results screen
Build the full-screen exam overlay with countdown timer and question navigator sidebar. On submission, call `POST /api/exams/attempts` and immediately render the results screen from the response — no second request needed. The results screen must show the score summary, per-topic breakdown, and the full question-by-question review with expandable step-by-step solutions rendered in KaTeX. Add the retake and "practice weak topics" buttons.

### Step 13 — Leaderboard and profile screens
Build the Leaderboard page with faculty and time period filtering, highlighting the current user's row. Build the Profile page with the 365-day activity grid (render as a CSS grid of small divs colored according to the five intensity levels) and per-topic progress bars. Wire up all endpoints.

---

## Design system

The UI follows a dark GitHub / LeetCode aesthetic. Reference the two HTML prototype files (`mathrs-ftue.html` and `mathrs-ui.html`) for exact colors, spacing, component structure, and interaction behavior. Recreate these in React components as closely as possible.

**Color tokens:**
```
Background primary:    #0d1117
Background secondary:  #161b22
Background tertiary:   #21262d
Border default:        #30363d
Border hover:          #444c56
Text primary:          #e6edf3
Text secondary:        #8b949e
Text muted:            #6e7681
Accent green:          #3fb950
Accent blue:           #58a6ff
Accent orange:         #d29922
Accent red:            #f85149
```

**Typography:**
```
Display / headings:    Syne (700, 800)         — Google Fonts
Body / UI:             DM Sans (400, 500)       — Google Fonts
Math / code / IDs:     JetBrains Mono (400, 500) — Google Fonts
```

**Difficulty badges:**
```
Easy:    background #0d2217   text #3fb950   border #196c2e
Medium:  background #2a1f00   text #d29922   border #5a3d00
Hard:    background #2d1117   text #f85149   border #5a1d1a
```

**Border radius:**
```
Default components:    6px
Cards:                 10px
Pills / badges:        999px
```

---

## How to use this document with Claude Code

1. Open Claude Code inside your project folder
2. Paste this entire document as your first message
3. Then add the following:

> "Use this document as the full project spec. Start with Step 1 — create the Spring Boot project with the dependencies listed, configure PostgreSQL in application.properties, and verify the app starts and connects to the database successfully."

4. After each step is complete and tested, tell Claude Code to move to the next step
5. When you reach Step 9 (React frontend), add:

> "Reference the two HTML prototype files I am attaching for the exact visual design — colors, layout, component structure, typography, and all interaction behavior. Recreate every screen as React components using the color tokens and font choices defined in the design system section of the spec."

6. Attach `mathrs-ftue.html` and `mathrs-ui.html` at that point

Work through the steps one at a time. Do not move to the next step until the current one is working and tested. Do not jump ahead to the frontend before the backend endpoints are confirmed working.
