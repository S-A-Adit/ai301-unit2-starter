Hi, I'd like to take this one on. The Redis probe in `health_check()`
(`api/routes/health.py`) builds its client from `settings.redis_host` and
`settings.redis_port`, but `Settings` in `core/config.py` only defines
`redis_url` — so the probe raises `AttributeError`, the surrounding
`except Exception` swallows it, and `/health` reports Redis as down even
when it's reachable.

Next: I'll reproduce this against a fresh clone and post my own repro
report (environment, steps, and the actual response/log), then look at
switching the probe to build its client from `redis_url` instead.
