---
title: Async Migrations
sidebar: Docs
showTitle: true
---

> Note: Async Migrations are in Beta, if you're interested in trying this out reach out to us on [PostHog Users's Slack](https://posthog.com/slack) first.

## What are async migrations?

Async migrations are _data migrations_ that do not run synchronously on an update to a PostHog instance. Rather, they execute on the background of a running PostHog instance, and should be completed within a range of PostHog versions. Related internal docs can be found [here](/handbook/engineering/async-migrations).

## Why are async migrations necessary?

Migrations are inevitable, and sometimes it may be necessary to execute non-trivial schema changes that can take a long time to complete. 

For example, ClickHouse does not support changing the primary key of a table, which is a change we were [forced to make in anticipation of upgrading ClickHouse beyond version 21.6](https://github.com/PostHog/posthog/issues/5684). As a result, the way to change the schema of the table was to create a new table and insert all the data from the old table into it, which took us an entire week to run on PostHog Cloud.

Now, while we at PostHog can execute such changes to our Cloud instance "manually", we felt compelled to provide a better approach for our users to do so.

As a result, we created a system capable of safely and efficiently managing migrations that need to happen asynchronously.

## Working with async migrations

Managing async migrations is a job for self-hosted PostHog instance admins. These migrations require some level of supervision as they affect how data is stored and may run for long periods of time.

However, worry not! We've built a system to make managing these as easy as possible.

### Prerequisite

To manage async migrations, you must be a staff user. PostHog deployments from version 1.31.0 onwards will automatically give the instance's first user "staff status", but those who have deployed PostHog before 1.31.0 will have to manually update Postgres.

To do so, follow our [guide for connecting to Postgres](/docs/self-host/deploy/troubleshooting#how-do-i-connect-to-postgres) and then run the following query:

```sql
UPDATE posthog_user 
SET is_staff=true
WHERE email=<your_email_here>
```

To confirm that everything worked as expected, visit `/instance/async_migrations` in your instance. If you're able to see the migrations info, you're good to go!

### Async migrations page

We've added a page where you can manage async migrations at `/instance/async_migrations`. 

On this page you can trigger runs, stop running migrations, perform migration rollbacks, check errors, and gather useful debugging information.

Here's a quick summary of the different columns you see on the async migrations table:

| Column | Description |
| : -----: | :--------: |
| Name and Description | The migration's name. This corresponds to the migration file name in [`posthog/async_migrations/migrations`](https://github.com/PostHog/posthog/tree/master/posthog/async_migrations/migrations) followed by an overview of what this migration does |
| Status | The current [status](https://github.com/PostHog/posthog/blob/master/posthog/models/async_migration.py#L5) of this migration. One of: 'Not started','Running','Completed successfully','Errored','Rolled back','Starting'. |
| Progress | How far along this migration is (0-100) |
| Current operation index | The index of the operation currently being executed. Useful for cross-referencing with the migration file |
| Current query ID | The ID of the last query ran (or currently running). Useful for checking and/or killing queries in the database if necessary. |
| Started at | When the migration started. |
| Finished at | When the migration ended. |

The settings tab allows you to change the configuration, e.g. whether async migrations should run automatically.

### How can I stop the migration?

In the async migrations page at `/instance/async_migrations` you can choose to `stop` or `stop and rollback` the migration from the `...` button on the right most column.

![Stopping the migration](../../../../images/async-migrations-stop-rollback.png)

### The migration is in an Error state - what should I do?

Try to rollback the migration to make sure we're in a safe state. You can do so from the async migrations page at `/instance/async_migrations` from `...` button on the right most column. If you're unable to rollback reachout to us in [slack](/slack).

![Rollback errored migration](../../../../images/async-migrations-error-rollback-button.png)


### Celery scaling considerations

To run async migrations, we occupy one Celery worker process to run the task. Celery runs `n` processes (per pod) where `n == number of CPU cores on the posthog-worker pod`. As such, we recommend scaling the `posthog-worker` pod in anticipation of running an async migration.

You can scale in two ways:

1. Horizontally by increasing the desired number of replicas of `posthog-worker`
2. Vertically by increasing the CPU request of a `posthog-worker` pod 

Once the migration has run, you can scale the pod back down. 

