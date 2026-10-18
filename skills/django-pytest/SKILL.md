---
name: django-pytest
description: Sets up and writes Django tests with pytest and pytest-django, covering configuration, database access markers, the client, rf, settings and mailoutbox fixtures, query count assertions, on_commit callbacks and testing data migrations. Use when adding pytest to a Django project, writing tests for models, views or APIs, fixing tests that fail with database access errors, or speeding up a slow test suite.
license: MIT
---

# Django testing with pytest-django

## Setup

```bash
pip install pytest pytest-django
```

Configure the settings module. In `pyproject.toml`:

```toml
# pytest 7 and 8
[tool.pytest.ini_options]
DJANGO_SETTINGS_MODULE = "config.settings.test"
python_files = ["test_*.py", "*_test.py", "tests.py"]
addopts = "--reuse-db"

# pytest 9 also accepts a native table:
# [tool.pytest]
# DJANGO_SETTINGS_MODULE = "config.settings.test"
```

Use the project's real settings path. Check `manage.py` for the value it sets. Keep the test settings fast:

```python
# config/settings/test.py
from .base import *  # noqa

PASSWORD_HASHERS = ["django.contrib.auth.hashers.MD5PasswordHasher"]
EMAIL_BACKEND = "django.core.mail.backends.locmem.EmailBackend"
```

`--reuse-db` keeps the test database between runs. After a schema change, run once with `--create-db`.

## Database access

Tests can't touch the database by default. Grant it explicitly:

| Need | Use |
|---|---|
| A test that reads or writes | `@pytest.mark.django_db` |
| A fixture that writes | Request the `db` fixture in the fixture |
| Real commits, `on_commit` with a real transaction, or several threads | `@pytest.mark.django_db(transaction=True)` (slower) |
| A second database | `@pytest.mark.django_db(databases=["default", "replica"])` |
| Mark a whole module | `pytestmark = pytest.mark.django_db` at the top |

The error "Database access not allowed, use the django_db mark" means one of these is missing.

## Fixtures to know

```python
import pytest
from django.urls import reverse


@pytest.fixture
def user(db, django_user_model):
    return django_user_model.objects.create_user(username="ana", password="pw-12345")


def test_dashboard_requires_login(client, db):
    response = client.get(reverse("dashboard"))
    assert response.status_code == 302
    assert "/login" in response["Location"]


@pytest.mark.django_db
def test_dashboard_for_user(client, user):
    client.force_login(user)
    response = client.get(reverse("dashboard"))
    assert response.status_code == 200
    assert b"Welcome" in response.content


def test_admin_list(admin_client):
    assert admin_client.get("/admin/").status_code == 200


def test_feature_flag(settings, client, db):
    settings.FEATURE_NEW_CHECKOUT = True
    assert client.get(reverse("checkout")).status_code == 200


def test_view_directly(rf, user):
    request = rf.get("/reports/")
    request.user = user
    from reports.views import report_view
    assert report_view(request).status_code == 200
```

`settings` changes are undone after each test. `mailoutbox` holds the emails a test sent.

## Query counts

Pin the number of queries on pages that render lists, so N+1 regressions fail the test:

```python
@pytest.mark.django_db
def test_order_list_queries(client, user, django_assert_max_num_queries):
    OrderFactory.create_batch(20, customer=user)
    client.force_login(user)
    with django_assert_max_num_queries(6):
        client.get(reverse("orders:list"))
```

`django_assert_num_queries(n)` asserts an exact count. Use the max version for pages whose fixed overhead (session, auth) may change.

## on_commit callbacks

Code that sends email or enqueues jobs in `transaction.on_commit` doesn't run inside a normal test transaction. Capture the callbacks and run them:

```python
@pytest.mark.django_db
def test_signup_sends_welcome(client, mailoutbox, django_capture_on_commit_callbacks):
    with django_capture_on_commit_callbacks(execute=True) as callbacks:
        client.post(reverse("signup"), {"email": "a@example.com", "password": "pw-12345"})
    assert len(callbacks) == 1
    assert mailoutbox[0].to == ["a@example.com"]
```

Don't use this fixture in `transaction=True` tests, where callbacks run on real commit.

## Testing a data migration

For a data migration with real logic, test it by migrating to the previous migration, inserting rows with the historical models, then migrating forwards. The full example is in `references/migration-tests.md` in this skill's folder.

## Speed

- `--reuse-db` for local runs. `pytest -n auto` with `pytest-xdist` for parallel runs.
- `--no-migrations` builds the schema from models, which is faster, but it hides broken migrations. Keep one CI job that runs with migrations.
- Use factories (factory_boy) or small fixtures. Avoid large JSON fixtures.

## Checks before you finish

- `pytest` passes from a clean database (`--create-db`).
- Each test that touches the database has a marker or a `db` fixture.
- Views are tested through `client` with real URLs from `reverse`.
- List pages have a query count assertion.
- No test depends on the order it runs in. If `pytest-randomly` is installed, it shuffles the order on every run and shows the seed to rerun a failure.
