**Environment:** PathReview at commit `f89c06f` (fork `S-A-Adit/pathreview-ai301-fa26-s1`,
same commit the issue and other reproductions reference). Windows 11, Python 3.12.5,
FastAPI 0.109+/Starlette TestClient (httpx-based) and `uvicorn`. PostgreSQL 16 reachable at
`localhost:5432` (native install, not the repo's documented `docker compose` — Docker is
not available on this machine, so Postgres was installed and started directly instead;
`DATABASE_URL` in `.env` was changed from the documented port `5433` to `5432` to match).
Redis was not running in this environment. That does not affect this bug: the probe fails
on an `AttributeError` from a plain attribute access, before it ever attempts a network
connection to Redis, so the failure is identical whether or not a Redis server is reachable
— confirmed directly below.

**Isolating the root cause.** Before hitting the endpoint, confirmed directly that
`Settings` has no `redis_host` attribute:

```
$ .venv/Scripts/python.exe -c "
from core.config import settings
print('redis_url:', settings.redis_url)
try:
    print(settings.redis_host)
except AttributeError as e:
    print('AttributeError:', e)
"
redis_url: redis://localhost:6379/0
AttributeError: 'Settings' object has no attribute 'redis_host'
```

`Settings` (`core/config.py`) defines only `redis_url`; `redis_host`/`redis_port`, which
`api/routes/health.py`'s `health_check()` reads at lines 47-48, do not exist on it.

**Reproducing via the endpoint**, first with FastAPI's `TestClient` directly against the
app object (no live server, no startup/lifespan run):

```
$ .venv/Scripts/python.exe -c "
from fastapi.testclient import TestClient
from api.main import app

client = TestClient(app)
resp = client.get('/health')
print('status:', resp.status_code)
print('body:', resp.json())
"
2026-09-26 19:51:07 [error    ] postgres_health_check_failed   error="Textual SQL expression 'SELECT 1' should be explicitly declared as text('SELECT 1')" request_id=75259b93-...
2026-09-26 19:51:07 [error    ] redis_health_check_failed      error="'Settings' object has no attribute 'redis_host'" request_id=75259b93-...
2026-09-26 19:51:07 [debug    ] vector_db_health_check_passed  request_id=75259b93-...
status: 503
body: {'detail': {'status': 'unhealthy', 'dependencies': {'postgres': 'unhealthy', 'redis': 'unhealthy', 'vector_db': 'healthy'}, 'safety_events_last_hour': 0, 'timestamp': '2026-09-26T23:51:07.039389'}}
```

Then again through a real running server, matching the issue's own steps (`GET /health`
against a live app, not an in-process client):

```
$ .venv/Scripts/python.exe -m uvicorn api.main:app --host 127.0.0.1 --port 8000 &
...
INFO:     Application startup complete.
$ curl -s -w "\nHTTP_STATUS:%{http_code}\n" http://127.0.0.1:8000/health
{"detail":{"status":"unhealthy","dependencies":{"postgres":"unhealthy","redis":"unhealthy","vector_db":"healthy"},"safety_events_last_hour":0,"timestamp":"2026-09-26T23:51:45.245215"}}
HTTP_STATUS:503
```

The server log for that request:

```
2026-09-26 19:51:45 [error    ] postgres_health_check_failed   error="Textual SQL expression 'SELECT 1' should be explicitly declared as text('SELECT 1')" request_id=99637710-...
2026-09-26 19:51:45 [error    ] redis_health_check_failed      error="'Settings' object has no attribute 'redis_host'" request_id=99637710-...
2026-09-26 19:51:45 [debug    ] vector_db_health_check_passed  request_id=99637710-...
INFO:     127.0.0.1:54327 - "GET /health HTTP/1.1" 503 Service Unavailable
```

**Expected:** `/health` reports Redis's true status; if Redis is reachable, the response's
`dependencies.redis` should read `"healthy"`.

**Actual:** both runs return HTTP 503 with `dependencies.redis: "unhealthy"`, and the log
line names the exact cause: `redis_health_check_failed error="'Settings' object has no
attribute 'redis_host'"`. This matches the issue precisely — the probe never reaches
Redis at all; it fails on the attribute lookup itself, so `/health` reports Redis down
regardless of whether Redis is actually up.

**Note on `postgres: "unhealthy"` in the same response:** this is a real but separate,
pre-existing issue, not evidence about #62 — `await db.execute("SELECT 1")` passes a raw
string where SQLAlchemy 2.0 requires `sqlalchemy.text("SELECT 1")`, so the Postgres check
fails on that regardless of whether Postgres is reachable (Postgres was in fact reachable
in this environment — the app's own startup migration check completed against it
successfully moments earlier in the same run, shown by `application_startup_completed` in
the server log). Flagging this rather than folding it into this report, since it isn't
part of what issue #62 describes and its own root cause is unrelated to the Redis probe.
