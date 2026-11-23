# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Task manager built with Django 5.1 + Django REST Framework, PostgreSQL, Redis (cache + Celery broker), and Celery. The frontend is plain HTML/CSS/JS served by Django. The project is also a playground for observability: OpenTelemetry (OTLP/HTTP export) and Prometheus metrics, viewed in Grafana.

## Commands

The README's intended workflow is Docker (`web`, `worker`, `db`, `redis` services):

```powershell
docker compose up --build                                   # app at http://localhost:8000
docker compose exec web python manage.py createsuperuser
docker compose exec web python manage.py makemigrations
docker compose exec web python manage.py migrate
```

A local venv (`python -m venv env`, `pip install -r requirements.txt`) is mainly for editor import resolution. To run outside Docker, Postgres and Redis must be on localhost (the settings defaults), then `python manage.py runserver` and `celery -A config worker -l info`.

There is no test suite and no linter config yet. Django's runner would be `python manage.py test` (or `python manage.py test tasks.tests.SomeTest.test_x` for one test).

## Missing files referenced by code

These are imported but are not in the repo, so Django and Celery will not start until they exist:

- `config/telemetry.py`: must provide `instrument_django()`, `attach_log_handler()`, `instrument_celery()`, `shutdown_telemetry()`
- `config/metrics.py`: must provide `MetricsMiddleware` (first entry in `MIDDLEWARE`; serves the hand-written `/metrics` page, which `OTEL_PYTHON_DJANGO_EXCLUDED_URLS` excludes from tracing)
- `docker-compose.yml` / `Dockerfile` (the README and `.env` comments refer to them; `.env` sets `OTEL_SERVICE_NAME` per service and Grafana admin credentials for port 3000)
- `.gitignore` mentions a `loadtest/results/` directory

## Architecture

- **`config/`**: project package. All settings come from env vars (see `.env`) with localhost defaults. Redis DBs: `/0` cache, `/1` Celery broker, `/2` Celery results. `config/__init__.py` imports the Celery app so it loads with Django.
- **Telemetry init order matters.** In `wsgi.py`/`asgi.py`, `instrument_django()` must run *before* `get_wsgi_application()` (so the OTel middleware is present when the middleware chain is built), and `attach_log_handler()` must run *after* it (because `django.setup()` re-applies `LOGGING` and would drop the handler). `runserver` also imports `wsgi.py` through `WSGI_APPLICATION`.
- **Celery telemetry is per child process.** In the prefork pool, OTel export threads don't survive a fork, so `config/celery.py` calls `instrument_celery()` on `worker_process_init` and flushes on `worker_process_shutdown`. Keep it that way. Don't initialize OTel in the parent worker process.
- **OpenTelemetry pins** in `requirements.txt` are released together (SDK 1.36.0 goes with instrumentation 0.57b0). Upgrade them all at once.
- **`accounts/`**: has no models and uses Django's built-in `User`. DRF `TokenAuthentication` is the only auth class, and `IsAuthenticated` is the global default (signup/login override it with `AllowAny`). `/api/auth/users/` is cached in Redis under `accounts:user_list` for 300s. `accounts/signals.py` clears that key on User save/delete.
- **`tasks/`**: a single `Task` model (`created_by`, nullable `assigned_to`). The access rules are split across files:
  - `TaskViewSet.get_queryset` limits results to tasks the user created or is assigned to.
  - `permissions.IsCreatorOrAssignee`: the creator or assignee can read/update, and only the creator can delete.
  - `TaskSerializer.validate_assigned_to`: only the creator can change `assigned_to`.
- **Background job**: `tasks/tasks.py::send_assignment_notification` is a simulated 1-second "email" that only logs. The viewset enqueues it with `.delay()` on create (if the task is assigned) and on update (when the assignee changes).
- **Frontend**: `config/urls.py` serves `templates/{tasks,login,signup}.html` through `TemplateView`. These pages call the API with `fetch` from `static/js/api.js`, which stores the DRF token in `localStorage`, sends `Authorization: Token <token>`, and redirects to `/login/` on a 401. Session/CSRF auth is not used by the API.
- `AUTH_PASSWORD_VALIDATORS` is intentionally empty, and signup only requires a password of at least 4 characters.
