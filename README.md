# Hunt St Technical Assessment

**Role:** Senior Full Stack Laravel Developer (011 - 1370)  
**Candidate:** Jhon Paul Quiñal  
**Assessment type:** Laravel Code Review and Framework Upgrade

---

## What this repository contains

This repository documents the unpaid technical assessment completed for the Hunt St / Make Badges senior Laravel role.

| File / folder | Purpose |
|---|---|
| [`Questions.md`](./Questions.md) | Full written answers for Sections 1 and 2 of the assessment |
| Screen recording (submitted separately) | Walkthrough of the assessment, including reasoning and code review |
| `laravel-inertia-react-ziggy/` | Local Laravel + Inertia + React workspace used as a reference environment during preparation / demonstration |

The assessment brief stated that a complete application build or live framework upgrade was **not required**. The expected deliverable was:

1. Written answers
2. A screen recording explaining the approach
3. Shareable links or attachments with view access enabled

---

## Assessment instructions (summary)

- Complete both sections and submit answers with a screen recording
- Talk through technical decisions while working
- Official docs and technical resources may be used
- Target completion window: about 45–60 minutes
- Focus on technical reasoning rather than lengthy explanations
- Scope: evaluate knowledge and problem-solving approach

Full task wording and answers are in [`Questions.md`](./Questions.md).

---

## How the assessment was implemented

### Section 1 — Laravel Code Review (N+1)

1. Identified the N+1 pattern in the provided `Post::latest()->take(50)->get()` loop.
2. Explained query growth: 50 posts ≈ 51 queries; 500 posts ≈ 501 queries.
3. Fixed it with `with('author')` eager loading.
4. Tightened the query with selective columns while keeping keys required by Eloquent relationships.
5. Described verification via Debugbar/Telescope, query timing, and tests.
6. Listed follow-up investigations if the page remained slow after the fix.

### Section 2 — Laravel 11 → 13 Upgrade Plan

1. Inventoried files, dependencies, tests, and infrastructure before changing anything.
2. Proposed a **sequential** upgrade path: Laravel 11 → 12, then 12 → 13.
3. Ordered work as: baseline → PHP/runtime → Composer/framework → frontend → application/config changes.
4. Covered incompatible package decisions (upgrade, replace, workaround, defer, rewrite).
5. Defined validation steps after each major version.
6. Documented downtime/risk controls: staging parity, backups, careful migrations, deployment/monitoring/rollback.

---

## Submission history

1. Completed written reasoning and recorded a full-screen walkthrough.
2. Submitted the screen recording to Hunt St recruitment.
3. Adjusted link sharing settings to **Anyone with the link can view** when requested.
4. This repository packages the written assessment answers for review if a GitHub/shareable code link is requested.

---

## How to review

1. Open [`Questions.md`](./Questions.md) for the full written assessment.
2. Review the submitted screen recording for the spoken walkthrough and live reasoning.
3. Optionally inspect `laravel-inertia-react-ziggy/` as the local Laravel reference environment used around the assessment.

---

## Note

This assessment was a **code review + upgrade planning** exercise. It was not intended as a production application deliverable. The written answers are the primary artifact for evaluating technical approach.
