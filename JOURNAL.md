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

**Cohort ledger:** [ ] Issue added to cohort ledger

**"Is this right for me?" checklist notes:**
Before GET /health reports Redis as unhealthy even when Redis is running, because the probe reads settings.redis_host, which doesn't exist.
After GET /health uses settings.redis_url to verify the Redis connection and returns a clean, structured JSON response showing component statuses.
It's tier 1 issue. The bug has a clear root cause and requires a minor fix in a single file (api/routes/health.py) to swap out settings.redis_host / redis_port in favor of settings.redis_url (plus adding or updating a test file).
I opened api/routes/health.py and core/config.py. I found that health.py references nonexistent attributes on the settings object. Looking at the test suite, there are currently no automated unit tests specifically covering the /health endpoint, so a test needs to be written to verify both healthy and degraded states.
I'm fine with the number of claims and the fix and corresponding test should take around 4 hours.