---
name: django-migrations
description: Writes and reviews Django migrations that are safe to deploy while the old code is still running, including data migrations with RunPython and historical models, batched backfills, adding non-null and unique fields to existing rows, concurrent indexes on PostgreSQL, and removing fields in two steps. Use when creating or editing a migration, backfilling data, changing a column on a large table, or reviewing migrations in a pull request.
license: MIT
---

# Django migrations, including safe data migrations

Two things go wrong with migrations on a live app. Old code meets the new schema (or the reverse) during a deploy. And schema changes lock a busy table. Plan for both.

## Steps

1. Generate with `python manage.py makemigrations <app>`. Read the file. Rename it if the auto name doesn't describe the change (`0012_order_public_id.py`).
2. Print the SQL with `python manage.py sqlmigrate <app> <number>`. Look for table rewrites, `NOT NULL` without a default, and non-concurrent `CREATE INDEX` on big tables.
3. Check the operation against the table below and split it if needed.
4. Keep schema changes and data changes in separate migrations.
5. Test forwards and backwards: `python manage.py migrate <app> <number>`, then migrate back to the previous number, then forwards again.
6. In CI, run `python manage.py makemigrations --check --dry-run` so missing migrations fail the build.

## Risky changes and the safe version

| Change | Risk | Safe version |
|---|---|---|
| Add a non-null field to a table with rows | Old code inserts without it, and the add may rewrite the table | Add as `null=True`, backfill, then make it non-null. Or use `db_default` |
| Add a unique field to existing rows | All rows get the same default and the unique index fails | The three-migration pattern below |
| Add an index on a big PostgreSQL table | `CREATE INDEX` blocks writes | `AddIndexConcurrently` with `atomic = False` |
| Add a constraint on a big PostgreSQL table | Full scan under lock | `AddConstraintNotValid`, then `ValidateConstraint` in a second migration |
| Remove a field | Old code still selects the column | Remove it from the model state first, drop the column in a later deploy |
| Rename a field or model | Old code uses the old name | Add new, copy, switch code, drop old, or keep `db_column` / `db_table` |
| Large data update in one transaction | Long locks, and a huge rollback on failure | `atomic = False` with batches, each in its own `transaction.atomic()` |

## Data migration rules

1. Get models with `apps.get_model("app", "Model")`. Never import them from `app.models`. Imported models reflect today's code and break when the model changes later.
2. Always pass `reverse_code`. Use `migrations.RunPython.noop` when undoing is not needed.
3. Don't call model methods or custom managers. Historical models don't have them (unless a manager sets `use_in_migrations = True`).
4. Process in batches and save only the fields you change (`update_fields=[...]`), or use `bulk_update`.
5. Make it safe to run twice. Filter on rows that still need the change.

## Example: add a unique UUID to existing rows

Migration 1 adds the field as nullable:

```python
migrations.AddField(
    model_name="order",
    name="public_id",
    field=models.UUIDField(null=True),
)
```

Migration 2 fills it in batches (create it with `makemigrations orders --empty`):

```python
import uuid
from django.db import migrations, transaction


def fill_public_ids(apps, schema_editor):
    Order = apps.get_model("orders", "Order")
    while True:
        with transaction.atomic():
            batch = list(
                Order.objects.filter(public_id__isnull=True)
                .select_for_update(skip_locked=True)[:1000]
            )
            if not batch:
                break
            for order in batch:
                order.public_id = uuid.uuid4()
            Order.objects.bulk_update(batch, ["public_id"])


class Migration(migrations.Migration):
    atomic = False

    dependencies = [("orders", "0012_order_public_id")]

    operations = [
        migrations.RunPython(fill_public_ids, reverse_code=migrations.RunPython.noop),
    ]
```

Migration 3 makes it required and unique:

```python
migrations.AlterField(
    model_name="order",
    name="public_id",
    field=models.UUIDField(default=uuid.uuid4, unique=True, editable=False),
)
```

Deploy code that sets `public_id` on new rows (the model `default`) before or with migration 2. Otherwise, rows created during the backfill stay null and migration 3 fails. `select_for_update(skip_locked=True)` keeps two workers from taking the same rows. Remove it if your database doesn't support `SKIP LOCKED`.

## Example: concurrent index (PostgreSQL)

```python
from django.contrib.postgres.operations import AddIndexConcurrently
from django.db import migrations, models


class Migration(migrations.Migration):
    atomic = False

    dependencies = [("orders", "0014_order_public_id_unique")]

    operations = [
        AddIndexConcurrently(
            "order",
            models.Index(fields=["customer", "-created_at"], name="order_customer_created_idx"),
        ),
    ]
```

The model's `Meta.indexes` should list the same index, so later `makemigrations` runs see no change.

## Example: remove a field in two deploys

Deploy 1 removes the field from the model and from Django's state only:

```python
migrations.SeparateDatabaseAndState(
    state_operations=[migrations.RemoveField(model_name="order", name="legacy_code")],
    database_operations=[],
)
```

If the column is `NOT NULL` with no database default, first make it `null=True` in an earlier migration, so inserts from the new code don't fail.

Deploy 2 drops the column:

```python
migrations.RunSQL(
    sql='ALTER TABLE "orders_order" DROP COLUMN "legacy_code";',
    reverse_sql=migrations.RunSQL.noop,
)
```

Check the real table and column names with `sqlmigrate` or `dbshell` first.

## Review checklist

- No model imports in migrations. Only `apps.get_model`.
- Every `RunPython` and `RunSQL` has a reverse, or a comment saying why it can't.
- Data migrations are batched, can run twice safely, and sit in their own migration.
- `atomic = False` is set wherever there's a concurrent operation or a long backfill.
- No migration adds a non-null column to a large table without `db_default` or a nullable first step.
- Field removals and renames are split across deploys.
- Squashing (`squashmigrations`) happens only when every environment has applied the originals.
