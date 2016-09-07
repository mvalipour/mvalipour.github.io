---
layout: post
title: "How to safely deploy a decomposed database when using Entity Framework code-first"
description: ""
category: "back-end"
tags: [entity-framework, migrations]
---
{% include JB/setup %}

When moving to a continuous delivery model, we started decomposing some of our larger solutions into smaller sub-systems and systems built with Entity Framework code-first, face a tougher challenge with their database being decomposed.

The challenge is, how to avoid down-time between when the database (migrations) and the main website (or any other sub-system depending on the database) are deployed.

<!--more-->

The solution really depends on how wildly and how often do you change the schema of your database. In our case the product is mature enough to allow us making the safe bet that most of our migrations are either completely *additive* or can be broken down into two or more steps to avoid a complete halt of system functionalities, when in transition period.

With this in-mind, We need to configure our Entity Framework to run *Code-First* on development environment and *Database-First* in production.

This makes it possible to push schema changes to live in three stages:

### Stage 1. Database Migrations

At this stage, we will use EF's `migrate.exe` utility (or simply script them before hand) to run latest migrations against the live database. After migrations are applied our website in production still keeps functioning as if nothing had happened (because it's configured to be database-first and in this mode EF does not perform a schema checking at run-time).

### Stage 2. Update production website

At this stage do your normal website (or any other sub-system) deployment.

### Stage 3. Database Migrations (part 2)

In those cases where we had for example a database table or column renamed, we will need to consider breaking it into two steps of:

- Add a new column (done in part 1)
- Remove old column and migrate data (done in part 2).

---

## Appendices

### EF Database-First only in production

In `Startup.cs` or `Global.asax.cs`:

```language-csharp
#if DEBUG
    Database.SetInitializer(new MigrateDatabaseToLatestVersion<AppDatabase, Migrations.Migrations.Configuration>());
#else
    Database.SetInitializer(new RequireDatabaseToBeUpToDate<AppDatabase, Migrations.Migrations.Configuration>());
#endif
```

This does exactly what it says on the tin:

- **On Local:** Migrates it's database to latest migration.
- **In Production:** Ensures that the database migrations is **NOT AHEAD** of the model assembly it is using. -- this is a safety measure making sure even if we ever accidentally deployed web before database, it stops the site from firing up.

```language-csharp
public class RequireDatabaseToBeUpToDate<TContext, TMigrationsConfiguration> : IDatabaseInitializer<TContext>
    where TContext : DbContext
    where TMigrationsConfiguration : DbMigrationsConfiguration, new()
{
    public void InitializeDatabase(TContext context)
    {
        var migrator = new DbMigrator(new TMigrationsConfiguration());
        var migrations = migrator.GetPendingMigrations().ToList();
        if (migrations.Any())
        {
            var message = "There are pending migrations that must be applied (via a script or using migrate.exe) before the application is started.\r\n" +
                $"Pending migrations:\r\n{string.Join("\r\n", migrations)}";
            throw new MigrationsPendingException(message);
        }
    }
}
```

### Running migrations against live database

```language-bash
$migrate = "<path>\migrate.exe" $migrateConfig = "<path>\migrate.exe.config" $connectionString = <your-live-connection-string> & $migrate <your-project-migration-assembly> /startupConfigurationFile=$migrateConfig <your-migration-configuration-type-name> /connectionString=$connectionString /connectionProviderName=System.Data.SqlClient /verbose
```
