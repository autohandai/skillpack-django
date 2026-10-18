# Django skills

Three skills for Django work: models with real database constraints, migrations (including data migrations) that are safe to deploy, and pytest-django tests.
Each one is a plain SKILL.md file. Your coding agent reads it when the task matches.

## Install

From the root of your Django project:

```bash
git clone https://github.com/autohandai/skillpack-django
mkdir -p .claude/skills
cp -R skillpack-django/skills/* .claude/skills/
```

Copy the skill folders themselves, so you get `.claude/skills/django-migrations/SKILL.md` and not a nested pack folder. Some agents only look one level deep. For every project on your machine, copy them into `~/.claude/skills/` instead.

Most agents pick a skill up when your request matches its description. To call one by name, mention it: `$django-migrations` in Autohand Code, or `/django-migrations` in Claude Code.

## Skills

| Skill | What it does |
|---|---|
| `django-models` | Field options, `TextChoices`, `CheckConstraint` and `UniqueConstraint`, indexes, `on_delete`, and N+1 avoidance |
| `django-migrations` | Batched `RunPython` backfills, unique fields on existing rows, `AddIndexConcurrently`, two-step field removal |
| `django-pytest` | pytest-django setup, database markers, fixtures, query count assertions, `on_commit`. Migration tests are in `references/` |

## Adapting it

- Not on PostgreSQL? Delete the concurrent index and `AddConstraintNotValid` rows in `django-migrations`.
- Set your real settings module path in `django-pytest`.
- If you use DRF, add an API client example (`rest_framework.test.APIClient`) to `django-pytest`.

Keep the `name` in the frontmatter the same as the folder name.

Written against Django 6.1 (with notes for 5.2 LTS) and pytest-django 4.14 as of October 2026. If something's out of date, please open an issue.

Runs in Autohand Code (free CLI, any model): https://autohand.ai/cli/?utm_source=skillpack-django&utm_medium=github

Until 31 October 2026: first month of Pro for $10 with code FOUNDING10 (first-time customers).

MIT licensed. See LICENSE.
