###################
Database migrations
###################

Modifying database schemata will need database migrations (even for adding and
removing tables). To autogenerate migrations::

    docker compose run web python -m warehouse db revision --autogenerate --message "text"

Verify your migration was generated by looking at the output from the command
above:

.. code-block:: console

    Generating /opt/warehouse/src/warehouse/migrations/versions/390811c1cdbe_.py ... done

Then migrate and test your migration::

    make runmigrations

Migrations are automatically run as part of the deployment process, but prior
to the old version of Warehouse from being shut down. This means that each
migration *must* be compatible with the current ``main`` branch of Warehouse.

This makes it more difficult to make breaking changes, since you must phase
them in over time. See :ref:`destructive-migrations` for tips on doing
migrations that involve column deletions or renames.

.. _migration-timeouts:

Migration Timeouts
==================

To help protect against an accidentally long running migration from taking down
PyPI, by default a migration will timeout if it is waiting more than 4s to
acquire a lock, or if any individual statement takes more than 5s.

The lock timeout helps to protect against the case where a long running app
query is blocking the migration, and then the migration itself ends up
blocking short running app queries that would otherwise have been able to
run concurrently with the long running app query.

The statement timeout helps to protect against locking the database for an
extended period of time (often for data migrations).

It is possible to override these values inside of a migration, to do so you can
add:

.. code-block:: python

    op.execute("SET statement_timeout = 5000")
    op.execute("SET lock_timeout = 4000")

To your migration.

For more information on what kind of operations are safe in a high availability
environment like PyPI, there is related reading available at:

- `PostgreSQL at Scale: Database Schema Changes Without Downtime <https://medium.com/paypal-tech/postgresql-at-scale-database-schema-changes-without-downtime-20d3749ed680>`_
- `Move fast and migrate things: how we automated migrations in Postgres <https://benchling.engineering/move-fast-and-migrate-things-how-we-automated-migrations-in-postgres-d60aba0fc3d4>`_
- `PgHaMigrations <https://github.com/braintree/pg_ha_migrations>`_

.. _destructive-migrations:

Destructive migrations
======================

.. warning::

  Read this section and its respective sub-sections **completely** before
  attempting to follow them! Failure to do so can result in serious
  deployment errors and outages.

Migrations that do column renames or deletions need to be performed
with special care, due to how Warehouse is deployed. Performing a
migration without these steps will cause errors during deployment,
and may require a full revert.

.. _removing-a-column:

Removing a column
-----------------

To remove a column:

1. Perform the Python-level code changes, i.e. remove usages of the
   column/attribute within Warehouse itself. Do **not** generate
   an accompanying migration.
2. Submit the changes as a PR. Tag the PR with ``skip-db-check`` to allow
   it to pass CI without accompanying migrations.
3. Prepare a second PR containing just the generated migrations.
4. Merge the first PR and ensure its deployment before merging the second.

This will ensure that the "old" version of Warehouse (prior to the new migration
has no references to the column being deleted).

Renaming a column
-----------------

Renaming a column is more complex than deleting a column, since it involves
a data migration. To rename a column:

1. Create an initial migration that adds the new column, and add code that
   writes to the new column while reading from both it and the old column.
2. Deploy the initial migration.
3. Prepare a second migration that performs a backfill of the old column to
   the new column.
4. Deploy the second migration.
5. Follow the :ref:`removing-a-column` steps *in entirety* to remove the old
   column.

In total, this requires three separate migrations: one to add the new column,
one to backfill to it, and a third to remove the old column.

Creating Indexes
================

Indexes should be declared as part of SQLAlchemy Table definitions,
often under the ``__table_args__`` attribute.
Once declared, auto-generating a migration will create the index for alembic.

See more in index definition in
`SQLAlchemy documentation <https://docs.sqlalchemy.org/en/20/core/constraints.html#schema-indexes>`_.

Since index creation will often take longer than a few seconds
for tables that are large or active,
it is recommended to create indexes concurrently.

To create an index concurrently, adding ``postgresql_concurrently=True``
to the index definition is incompatible with our deployment migration process
as it runs in a transaction, and concurrent index creation requires a separate transaction.

Instead, manually update the migration to start a separate transaction.

After auto-generating the new migration, update the migration to create the index concurrently.
Here's an example of an migration for a definition of ``Index("tbl1_column1_idx", "column1")``
that was auto-generated, and manually updated to create the index concurrently:

.. code-block:: diff

    def upgrade():
   -    op.create_index(
   -        "tbl1_column1_idx",
   -        "tbl1",
   -        ["column1"],
   -        unique=False,
   -    )
   +    # CREATE INDEX CONCURRENTLY cannot happen inside a transaction. We'll close
   +    # our transaction here and issue the statement.
   +    op.get_bind().commit()
   +    with op.get_context().autocommit_block():
   +        op.create_index(
   +            "tbl1_column1_idx",
   +            "tbl1",
   +            ["column1"],
   +            unique=False,
   +            if_not_exists=True
   +            postgresql_concurrently=True,
   +        )

The original ``op.create_index()`` call is indented under a context manager,
and the keyword args ``if_not_exists=True`` and ``postgresql_concurrently=True``
are added to the call.

Leave the generated ``downgrade()`` function as normal.

If the index creation is likely to continue to take longer than a few seconds,
and most indexes on existing tables in use are likely to take longer than a few seconds,
it is recommended to modify the migration to increase the statement timeout
as described in :ref:`migration-timeouts`.

Another option is to share the SQL statement to create the index concurrently
on the Pull Request, and have a maintainer run the statement manually.
