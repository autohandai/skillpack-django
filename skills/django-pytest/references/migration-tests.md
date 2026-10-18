# Testing a data migration with pytest-django


Use `MigrationExecutor` to move the schema to the migration before yours, insert rows with historical models, then migrate forwards:

```python
import pytest
from django.db import connection
from django.db.migrations.executor import MigrationExecutor


@pytest.mark.django_db(transaction=True)
def test_fill_public_ids():
    executor = MigrationExecutor(connection)
    executor.migrate([("orders", "0012_order_public_id")])
    old_apps = executor.loader.project_state([("orders", "0012_order_public_id")]).apps
    Order = old_apps.get_model("orders", "Order")
    order = Order.objects.create(total="10.00", customer_id=create_user_id(old_apps))

    executor = MigrationExecutor(connection)
    executor.migrate([("orders", "0013_fill_public_ids")])
    new_apps = executor.loader.project_state([("orders", "0013_fill_public_ids")]).apps
    assert new_apps.get_model("orders", "Order").objects.get(pk=order.pk).public_id is not None

    executor = MigrationExecutor(connection)
    executor.migrate(executor.loader.graph.leaf_nodes())   # leave the schema current
```

`create_user_id` stands for whatever helper creates the required related rows with `old_apps`. These tests are slow. Keep them for data migrations with real logic, and mark them so they can be skipped locally (`@pytest.mark.slow`, registered under `markers` in the config).

