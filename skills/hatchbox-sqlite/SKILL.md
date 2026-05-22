---
name: hatchbox-sqlite
description: |
  Configure SQLite for a Rails app deployed to Hatchbox. Use whenever the user is
  deploying (or has deployed) a Rails app to Hatchbox and the database is SQLite —
  the database file MUST live in the Capistrano-style `shared/` directory, not the
  app root, or it gets wiped on every deploy. Also use when troubleshooting "my
  data disappeared after deploy", missing tables, or fresh-looking production
  data on a Hatchbox SQLite app. Single-server only — flag this if the user
  mentions horizontal scaling, multiple app servers, or load balancing.
---

# Hatchbox + SQLite

Hatchbox deploys with a Capistrano-style layout: each release goes into a timestamped directory and the live app is a symlink to the current one. **Anything written to the app directory is lost on the next deploy.** SQLite stores its database as a file in the app directory by default, so without intervention every deploy effectively resets the database.

The fix is to point SQLite at the `shared/` directory, which persists across deploys.

Reference: [hatchbox.relationkit.io/articles/75](https://hatchbox.relationkit.io/articles/75-can-i-use-sqlite-databases)

## Hard constraint: single server only

SQLite is a file, not a server. It only works on **one** machine. Before configuring this:

- If the user mentions multiple app servers, autoscaling, or horizontal scaling — stop and flag that SQLite is the wrong choice. Recommend Postgres (Hatchbox provisions it).
- If the user wants high availability across servers — same story.
- If the app is single-server (the common Hatchbox setup) — proceed.

## The fix

Put the SQLite file under `/home/deploy/<app>/shared/`. Replace `<app>` with the actual app name as it appears in Hatchbox.

Pick **one** of these (don't do both):

### Option A: `DATABASE_URL` (recommended)

Set the env var in Hatchbox's app environment:

```
DATABASE_URL=sqlite3:///home/deploy/<app>/shared/production.sqlite3
```

Three slashes after `sqlite3:` — two for the URL scheme, one for the absolute path.

This overrides whatever's in `config/database.yml`, so it's the cleanest way: no code changes, just an environment setting.

### Option B: `config/database.yml`

```yaml
production:
  adapter: sqlite3
  database: /home/deploy/<app>/shared/production.sqlite3
  pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  timeout: 5000
```

Commit this only if every production environment really does deploy with the same on-disk path. Otherwise prefer Option A.

## Multi-database (Rails 7.1+ solid_queue / solid_cache / solid_cable)

Rails 8's default `solid_*` stack uses additional SQLite databases. Each needs its own file under `shared/`:

```yaml
production:
  primary:
    adapter: sqlite3
    database: /home/deploy/<app>/shared/production.sqlite3
  queue:
    adapter: sqlite3
    database: /home/deploy/<app>/shared/production_queue.sqlite3
    migrations_paths: db/queue_migrate
  cache:
    adapter: sqlite3
    database: /home/deploy/<app>/shared/production_cache.sqlite3
    migrations_paths: db/cache_migrate
  cable:
    adapter: sqlite3
    database: /home/deploy/<app>/shared/production_cable.sqlite3
    migrations_paths: db/cable_migrate
```

If only `primary` is moved to `shared/`, jobs and cache will be wiped every deploy — usually fine for cache, definitely not for queue.

## Backups

Hatchbox doesn't back up SQLite databases automatically (its built-in backup tooling is Postgres/MySQL-focused). At minimum:

```bash
# On the server, run as a cron job:
sqlite3 /home/deploy/<app>/shared/production.sqlite3 ".backup /home/deploy/<app>/shared/backups/production-$(date +%F).sqlite3"
```

Then ship the resulting file off-box (S3, rsync, etc.). [Litestream](https://litestream.io/) is the more robust option for continuous replication.

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| Data disappears after every deploy | DB file is in the app dir, not `shared/` | Set `DATABASE_URL` per Option A and re-deploy. The first deploy after the fix starts with an empty DB — restore from backup if you have one. |
| `SQLite3::CantOpenException` / `unable to open database file` | `shared/` path doesn't exist yet, or permissions wrong | SSH in and `mkdir -p /home/deploy/<app>/shared && chown deploy:deploy /home/deploy/<app>/shared` |
| `database is locked` errors under load | Concurrent writers contending for the file | Raise `timeout` in `database.yml`, enable WAL mode (`PRAGMA journal_mode=WAL`), or migrate to Postgres |
| Two app servers, only one sees writes | Trying to scale SQLite across machines | Not solvable — switch to Postgres |
