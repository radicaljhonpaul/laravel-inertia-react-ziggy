# Technical Assessment: Laravel Code Review and Framework Upgrade

**Candidate:** Jhon Paul Quiñal  
**Role:** Senior Full Stack Laravel Developer (011 - 1370)  
**Submission format:** Written answers + screen recording of the assessment walkthrough

---

## Assessment Approach

This assessment was completed as a technical reasoning exercise, not as a full application build or live framework upgrade (per the assessment scope).

During the screen recording I:

1. Reviewed the provided Laravel code and talked through the N+1 problem.
2. Rewrote the query with eager loading and column selection.
3. Explained how I would verify the fix and what else I would check if the page stayed slow.
4. Walked through a staged Laravel 11 → 12 → 13 upgrade plan using the sample `composer.json`.
5. Covered risk controls: staging, backups, migrations, deployment, monitoring, and rollback.

Official Laravel upgrade guides and related documentation were used as reference while answering. Concise technical reasoning was prioritised over long explanations.

---

## 1. Laravel Code Review – N+1 Query Problem

### Original code

```php
$posts = Post::latest()
    ->take(50)
    ->get();

foreach ($posts as $post) {
    echo $post->title;
    echo $post->author->name;
}
```

Assume `Post` has an `author` relationship with `author_id` as the foreign key.

---

### Task 1 — Explain the performance problem

This is a classic **N+1 query** problem.

1. Eloquent runs **1 query** to fetch the 50 posts.
2. Inside the loop, accessing `$post->author` triggers a **separate query per post** because the relationship was not eager loaded.
3. For 50 posts, that becomes **1 + 50 = 51 queries**.
4. As the number of posts grows, the query count grows linearly: **500 posts → 501 queries**.

That repeated author lookup is why the page feels slow under load.

---

### Task 2 — Rewrite the code to resolve the N+1 problem

```php
$posts = Post::with('author')
    ->latest()
    ->take(50)
    ->get();

foreach ($posts as $post) {
    echo $post->title;
    echo $post->author->name;
}
```

`with('author')` eager-loads related authors in a second query using the collected `author_id` values. The result is typically **2 queries total** (posts + authors), instead of one query per post. That keeps query count stable as post volume increases and reduces database round trips.

---

### Task 3 — Retrieve only necessary columns

Yes. Loading full rows is unnecessary when the page only needs titles and author names.

```php
$posts = Post::query()
    ->select(['id', 'title', 'author_id', 'created_at'])
    ->with(['author:id,name'])
    ->latest()
    ->take(50)
    ->get();
```

Columns that must be retained:

| Model  | Required columns | Why |
|--------|------------------|-----|
| Post   | `id` | Primary key for Eloquent model identity |
| Post   | `author_id` | Foreign key required for the `author` relationship |
| Post   | `created_at` | Needed for `latest()` ordering |
| Post   | `title` | Displayed in the response |
| Author | `id` | Must match the foreign key for association |
| Author | `name` | Displayed in the response |

Without `author_id` on posts and `id` on authors, Eloquent cannot correctly associate each post with its author when constrained selects are used.

---

### Task 4 — Confirm the changes work

I would compare query count and timing before and after the fix:

1. **Laravel Debugbar** or **Telescope** to inspect query count and duration on the page request.
2. Temporary `DB::listen` / query logging in local or staging to count SQL statements.
3. Feature test that asserts the expected number of queries (for example with `DB::listen` or available query assertions).
4. Compare wall-clock response time for the same dataset before vs after eager loading.

Success criteria: query count drops from ~51 to ~2 for 50 posts, and page response time improves under the same data volume.

---

### Task 5 — If the page remains slow after the fix

I would investigate beyond this specific N+1:

1. Missing indexes on `posts.created_at` / `posts.author_id`.
2. Other N+1 queries elsewhere on the page (Blade, Inertia props, nested relations).
3. Accidental heavy `$with` / nested eager loads fetching more than needed.
4. Lack of caching for stable read-heavy data.
5. Controller/business logic doing expensive work on every request.
6. Server capacity (CPU, memory, PHP-FPM workers), slow network, or oversized payloads.

---

### Task 6 — Revised code and reasoning

**Final revised approach:**

```php
$posts = Post::query()
    ->select(['id', 'title', 'author_id', 'created_at'])
    ->with(['author:id,name'])
    ->latest()
    ->take(50)
    ->get();

foreach ($posts as $post) {
    echo $post->title;
    echo $post->author->name;
}
```

**Reasoning:**

1. Eager loading removes the N+1 pattern.
2. Column selection reduces data transferred from the database.
3. Required identity/foreign-key columns are kept so Eloquent relationships still resolve correctly.
4. Verification would be done with Debugbar/Telescope and query assertions before considering the fix complete.

---

## 2. Laravel Framework Upgrade – Laravel 11 to Laravel 13

### Sample `composer.json` (illustrative)

```json
{
    "require": {
        "php": "^8.2",
        "laravel/framework": "^11.0",
        "guzzlehttp/guzzle": "^7.0",
        "predis/predis": "^2.0"
    },
    "require-dev": {
        "phpunit/phpunit": "^10.0",
        "laravel/pint": "^1.0"
    }
}
```

---

### Task 1 — What to inspect before making changes

Before upgrading I would inventory:

**Application / dependency files**

- `composer.json` / `composer.lock`
- `package.json` / `package-lock.json` (or Yarn/PNPM lockfile)
- `bootstrap/app.php` and Laravel 11-style application bootstrap
- `config/*`
- Service providers, middleware, routes, jobs, queues, scheduled commands
- Test suite: Unit, Feature, Integration

**Dependency health**

- `composer outdated -D`
- `composer audit`

**Infrastructure**

- PHP runtime version
- Docker / Dockerfiles / CI images
- Database and Redis versions
- Cron / scheduler jobs
- Queue workers and process managers
- Third-party services and SDKs tied to Laravel versions

The goal is a clear baseline of what must move with the framework upgrade and what might block it.

---

### Task 2 — Proposed upgrade approach (sequential)

I would upgrade **sequentially: Laravel 11 → 12, then 12 → 13**, not both majors in one jump.

Doing one major at a time:

1. Isolates breaking changes to a smaller surface.
2. Makes failures easier to diagnose against the official upgrade guide for that version.
3. Lets tests and staging validation catch issues before compounding them with the next major.

**Order of work**

1. **Baseline first**  
   Make sure CI is green, deprecation warnings are cleaned up where practical, and tests pass on Laravel 11.

2. **PHP / runtime compatibility**  
   Confirm the target Laravel version’s PHP requirement and upgrade the runtime in local, CI, staging, and production paths before or as part of that stage.

3. **Laravel framework + Composer dependencies**  
   - Laravel 11 → 12 (including related ecosystem bumps such as PHPUnit / Carbon where required)  
   - Laravel 12 → 13 (including related ecosystem bumps such as PHPUnit / Pest / Tinker where required)  
   Resolve Composer conflicts carefully; avoid force-install shortcuts that hide incompatibilities.

4. **Frontend packages and build tools**  
   Handle separately where possible (`npm outdated` / lockfile updates, Vite/build tooling). Prefer separate PRs so backend upgrade failures are not mixed with frontend churn.

5. **Application code and configuration**  
   Apply upgrade-guide changes, remove deprecated aliases/APIs, update config/bootstrap differences, then re-run tests.

**Verification at each stage before moving on**

- Composer install succeeds without platform requirement bypasses
- Automated tests pass
- App boots locally/staging
- Critical smoke paths work
- No unexpected deprecation/error spikes in logs

Only after Laravel 12 is stable would I proceed to Laravel 13.

---

### Task 3 — Compatibility issues, PHP version, and discovery process

At each stage I would investigate:

1. Official Laravel upgrade guides for 11→12 and 12→13
2. Deprecated features and removed APIs
3. Breaking changes in framework helpers, auth, validation, queues, and config structure
4. Third-party package constraints against the new Laravel version
5. Frontend package compatibility if Inertia/Vite/React tooling is involved

**PHP `^8.2` example**

I would determine Laravel 13’s required PHP version from:

1. Official Laravel docs / release notes
2. `composer.json` constraints for `laravel/framework` on Packagist
3. Local Composer solver feedback when raising the framework version

If Laravel 13 requires a newer PHP than 8.2, I would upgrade PHP in all environments first (local, CI, staging, production images/servers), then continue the framework bump.

**How I identify incompatible code/dependencies**

- Official upgrade guides
- Composer dependency resolution failures and `composer why` / `why-not`
- Automated static analysis where available (Larastan / PHPStan)
- Existing PHPUnit/Pest suite failures
- Optional helpers such as Laravel Shift for guided diffs
- Avoid `--ignore-platform-reqs` and forced installs that hide real blockers

---

### Task 4 — Incompatible third-party package options

If a package is incompatible with Laravel 12 or 13, I would consider:

1. **Upgrade** the package to a supported version
2. **Replace** it with a maintained alternative
3. **Work around** temporarily with a thin adapter / fork only if risk is understood
4. **Defer** the package change if it is non-critical and the upgrade can proceed without it
5. **Remove and rewrite** if the package is abandoned and the feature is still required

Decision factors: how critical the package is, maintenance status, migration effort, security risk, and whether blocking the Laravel upgrade is worse than temporarily reducing that feature. I would not force-install incompatible packages with `--ignore-platform-reqs` for a production path.

---

### Task 5 — Validation after each major upgrade

**Automated**

- Full PHPUnit / Pest suite
- Static analysis (Larastan / PHPStan) if already used
- Pint / style checks where part of CI
- Build pipeline for frontend assets

**Manual / staging**

- Auth flows
- Critical CRUD / business paths
- Queue jobs and scheduled commands
- Cache/config/route optimization behaviour
- Smoke tests against staging that mirrors production

**Before production**

- Staging sign-off
- Migration dry-run review
- Rollback plan confirmed
- Monitoring ready (for example Sentry or equivalent)

---

### Task 6 — Minimise downtime and manage risk

**Staging that reflects production**

Use a staging environment that mirrors production closely: same PHP/runtime, similar services (DB/Redis/queues), and representative config. Validate each major upgrade there before production.

**Backups**

- Database snapshots before deployment
- Critical file/object storage backup where relevant (for example S3 assets)
- Preserve secrets/config safely
- Tag the previous known-good release in Git for rollback

**Migrations**

- Prefer keeping large irreversible schema redesigns out of the framework upgrade release when possible
- Review every migration for reversibility and lock risk
- Avoid first-time testing of destructive migrations only against production-sized data without a prior staging rehearsal

**Deployment, monitoring, rollback**

- Prefer zero/low-downtime release process where available (for example Forge-style releases)
- After release: `php artisan optimize` (as appropriate) and `queue:restart`
- Monitor error tracking (for example Sentry) and application logs immediately after deploy
- Keep a clear rollback path to the previous tagged release / previous release artifact

**Post-deploy validation**

- Smoke-test critical features
- Watch dashboards for error/warning spikes
- Confirm queue workers and scheduler are healthy

**Overall sequence and risk management**

1. Baseline Laravel 11 + green tests  
2. Prepare PHP/runtime  
3. Upgrade to Laravel 12 → validate on staging  
4. Upgrade to Laravel 13 → validate on staging  
5. Production deploy with backups, monitoring, and rollback ready  

Risk is managed by isolating each major, validating early, avoiding forced dependency installs, and keeping production changes reversible where practical.

---

## Notes on AI / tooling disclosure

During the assessment walkthrough, official Laravel documentation and standard Composer/Node dependency tooling were the primary references. If AI assistance was used for phrasing or structuring answers, it was for clarifying technical explanations, not for replacing hands-on reasoning about the N+1 fix or the upgrade plan.
