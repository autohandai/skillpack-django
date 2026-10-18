---
name: django-models
description: Designs and edits Django models (Django 5.2 LTS and 6.x) with correct field options, choices, database constraints, indexes, on_delete behaviour, managers and query patterns that avoid N+1 queries. Use when adding a model or field, enforcing a business rule in the database, choosing on_delete, or reviewing models.py in a pull request.
license: MIT
---

# Django models

Check the version in `requirements*.txt`, `pyproject.toml` or `uv.lock` first. This skill targets Django 5.2 LTS and 6.x. Notes call out where they differ.

## Steps

1. Read the existing models in the app, and match their base class, naming, `Meta` style and timestamp fields.
2. Choose field types and options from the table below.
3. Put rules that must always hold into `Meta.constraints` as well as into `clean()` or a form.
4. Add an index for each new filter or ordering the code will run on a large table.
5. Pick `on_delete` deliberately for every `ForeignKey`.
6. Run `python manage.py makemigrations`, read the generated file, and run `python manage.py sqlmigrate <app> <number>` for anything touching a big table. The `django-migrations` skill covers safe rollout.

## Field choices

| Need | Use | Avoid |
|---|---|---|
| Optional text | `CharField(blank=True, default="")` | `null=True` on text, which gives two kinds of empty |
| Optional number, date or FK | `null=True, blank=True` | |
| Money | `DecimalField(max_digits=12, decimal_places=2)` or integer cents | `FloatField` |
| Fixed set of values | `models.TextChoices` / `IntegerChoices` | Free text plus validation in a form |
| Default set by the database | `db_default=Now()` | A Python default when rows are inserted outside Django |
| Created and updated times | `auto_now_add=True` / `auto_now=True` | Setting them by hand in `save()` |
| Public identifier | `UUIDField(default=uuid.uuid4, unique=True, editable=False)` | Exposing the integer primary key |

`blank` is for validation (forms, `full_clean`). `null` is for the database column. They are separate settings.

## Example

```python
import uuid
from django.conf import settings
from django.db import models
from django.db.models import Q
from django.db.models.functions import Now


class Order(models.Model):
    class Status(models.TextChoices):
        DRAFT = "draft", "Draft"
        PAID = "paid", "Paid"
        CANCELLED = "cancelled", "Cancelled"

    public_id = models.UUIDField(default=uuid.uuid4, unique=True, editable=False)
    customer = models.ForeignKey(
        settings.AUTH_USER_MODEL, on_delete=models.PROTECT, related_name="orders"
    )
    status = models.CharField(max_length=16, choices=Status, default=Status.DRAFT)
    total = models.DecimalField(max_digits=12, decimal_places=2)
    note = models.TextField(blank=True, default="")
    created_at = models.DateTimeField(db_default=Now())
    paid_at = models.DateTimeField(null=True, blank=True)

    class Meta:
        ordering = ["-created_at"]
        indexes = [
            models.Index(fields=["customer", "-created_at"], name="order_customer_created_idx"),
        ]
        constraints = [
            models.CheckConstraint(condition=Q(total__gte=0), name="order_total_non_negative"),
            models.CheckConstraint(
                condition=~Q(status="paid") | Q(paid_at__isnull=False),
                name="order_paid_has_paid_at",
            ),
            models.UniqueConstraint(
                fields=["customer"], condition=Q(status="draft"), name="order_one_draft_per_customer"
            ),
        ]

    def __str__(self):
        return f"Order {self.public_id}"
```

Version notes:
- `CheckConstraint` takes `condition=` on Django 5.1 and later. The older `check=` argument was deprecated in 5.1 and removed in 6.0.
- `choices=Status` (passing the enum class directly) works on 5.0 and later. Older code uses `Status.choices`.
- `db_default` needs 5.0 or later.
- Use `UniqueConstraint` instead of `unique_together`, which the docs mark as likely to be deprecated.

## on_delete

| Choice | Use when |
|---|---|
| `PROTECT` | The child is a record you must keep (orders, invoices) |
| `RESTRICT` | Like `PROTECT`, but allows the delete when the parent is also being removed by another cascade |
| `CASCADE` | The child has no meaning without the parent (line items of an order) |
| `SET_NULL` | The link is optional history (`null=True` required) |
| `SET_DEFAULT` / `SET(...)` | You reassign to a placeholder |
| `DO_NOTHING` | Only with a database-level rule you manage yourself |

`CASCADE` on a user foreign key deletes the user's data along with the account. Pick it on purpose. Django 6.1 adds database-level variants (`DB_CASCADE`, `DB_SET_NULL`, `DB_SET_DEFAULT`). Check the 6.1 release notes before you use them, since they skip Python signals.

## Queries that scale

- Use `select_related("customer")` for foreign keys and one-to-one fields you read in a loop. Use `prefetch_related("items")` for reverse and many-to-many relations.
- Use `exists()` to test for rows, `count()` to count them, and `values_list("id", flat=True)` when you only need ids.
- Use `update()` and `bulk_create()` for batch writes. They skip `save()` and signals, so check that nothing relies on those.
- Put reusable filters on a custom `QuerySet` and expose it with `objects = OrderQuerySet.as_manager()`.
- Wrap multi-row writes in `transaction.atomic()`.

## Checks before you finish

- Every rule that must always hold has a database constraint, and every constraint has a stable `name`.
- No `null=True` on `CharField` or `TextField` without a reason.
- Every `ForeignKey` has a deliberate `on_delete` and a `related_name`.
- New filters on large tables have an index.
- `__str__` is defined and doesn't trigger extra queries.
- `makemigrations --check` reports no missing migrations.
