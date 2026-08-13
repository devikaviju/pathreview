## Week 7 — Issue selection

**Issue link:** https://github.com/ascherj/pathreview/issues/155

**Issue title:** Health check references `settings.redis_host`, which does not exist on Settings

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
When a request is made to GET /health, it returns a normal-looking response that reports Redis as unhealthy, even when Redis is running fine — so the endpoint gives you wrong information rather than an error.
In api/routes/health.py, the Redis probe attempts to read settings.redis_host (and settings.redis_port), but in core/config.py, the Settings model defines the configuration using redis_url instead of separate redis_host / redis_port fields.
The application starts up normally because core/config.py loads valid settings (including redis_url). The error is dormant until a user or system actually invokes the GET /health route, at which point Python attempts to evaluate the missing attribute dynamically. The probe is wrapped in a broad except Exception (line 53) that catches it, logs it, and sets the status to "unhealthy" — so a code bug gets disguised as Redis being down.
Once fixed, GET /health will successfully read settings.redis_url to connect to Redis, perform the health ping, and return a 200 OK JSON response displaying the health status of both the database and Redis ({"status": "healthy", ...}).

**Branch name:** fix/155-health-check-redis-host

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [x] Issue added to cohort ledger

**"Is this right for me?" checklist notes:**
Before GET /health reports Redis as unhealthy even when Redis is running, because the probe reads settings.redis_host, which doesn't exist.
After GET /health uses settings.redis_url to verify the Redis connection and returns a clean, structured JSON response showing component statuses.
It's tier 1 issue. The bug has a clear root cause and requires a minor fix in a single file (api/routes/health.py) to swap out settings.redis_host / redis_port in favor of settings.redis_url (plus adding or updating a test file).
I opened api/routes/health.py and core/config.py. I found that health.py references nonexistent attributes on the settings object. Looking at the test suite, there are currently no automated unit tests specifically covering the /health endpoint, so a test needs to be written to verify both healthy and degraded states.
I'm fine with the number of claims and the fix and corresponding test should take around 4 hours.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/devikaviju/pathreview/commit/30e4007

**Reproduction summary:**
I ran `curl http://localhost:8000/health` against a
running stack and saw `"redis":"unhealthy"` while the Redis container was up;
the log showed `'Settings' object has no attribute 'redis_host'`. The test file
in the linked commit fails 2/4 against the pre-fix code.

**PLAN.md link:** https://github.com/devikaviju/pathreview/blob/fix/155-health-check-redis-host/PLAN.md

**Blockers or open questions:**
None

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Fixed the dormant attribute error in api/routes/health.py by updating the Redis probe logic to use settings.redis_url instead of the non-existent settings.redis_host and settings.redis_port attributes.

**Next steps:**
Write unit tests for the updated Redis health probe to verify healthy and degraded states, run local quality checks (make check, make test-unit), and submit the pull request.

**Blockers:**
None

---

### Check-in 2 (end of week)

**PR link:** https://github.com/ascherj/pathreview/pull/384

**Branch:** `fix/155-health-check-redis-host`

**What you built:**
Updated the health check endpoint in api/routes/health.py to build the Redis client from settings.redis_url via redis.Redis.from_url() instead of referencing nonexistent host/port attributes. This resolves false "unhealthy" Redis status responses while preserving existing exception logging.

**Tests added or updated:**
Added tests/unit/test_health.py with 4 unit tests using pytest and unittest.mock. The tests verify that Settings defines redis_url rather than redis_host/redis_port, that the probe builds its client from redis_url, and that /health reports Redis as healthy on a successful ping and unhealthy when the ping fails.

**Self-review confirmation:** [x] make test-unit passes (53 failed/375 passed before, 53 failed/379 passed after — same 53 pre-existing failures)
[ ] make check — fails on ~179 pre-existing ruff errors in tests/unit/ and missing mypy stubs; no errors introduced by this change (verified: `make lint` and `make typecheck` report nothing in tests/unit/test_health.py)

**Draft PR feedback received from:** I didn't recieve any feedback