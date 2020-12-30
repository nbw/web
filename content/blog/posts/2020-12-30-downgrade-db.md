---
title: "Gigalixir: Upgrading DB Tiers"
date: 2020-12-30T16:44:49-08:00
Description: "Quick instructions on how to backup and restore"
Tags: ["Postgres", "Gigalixir", "Elixir",]
Categories: ["code"]
draft: false

---

I have an app that uses Gigalixir and I'm paying for a Standard DB. Contray to the title, I actually downgrading my app to a free db (not upgraded).

Gigalixr warns that changing db tier types is your own responsibility, which seems intimidating at first but was actually not a big deal.

# 1. Install the Gigalixir CLI

You should already have it already if you're using Gigalixir. I'm not going to give instructions on this one.

# 2. Dump your current DB into a local file

To find your current DB info:

```
gigalixir pg -a [YOUR APP NAME]
```

Which will output something like:

```
[
  {
    "app_name": "...",
    "cloud": "gcp",
    "database": "...",
    "host": "34.70.42.144",
    "id": "...",
    "password": "...",
    "port": 5432,
    "region": "us-central",
    "size": 0.6,
    "state": "AVAILABLE",
    "tier": "STANDARD",
    "url": "postgresql://gigalixir_admin:..............",
    "username": "gigalixir_admin"
  }
]
```

Take the value of `url` and run the following:

```
pg_dump postgresql://gigalixir_admin postgresql://gigalixir_admin:............. -f [OUTPUT FILE NAME .dump]
```

Now we have a dump file of your database.

# 3. Check that it worked (optional)

To be sure that the dump file worked, let's test it locally. I'm on a mac with the Postgresql app. So:

```
createdb dump_tester

psql -h localhost -d dump_tester < [DUMP FILE]
```

That should dump the file into your local db called "dump_tester". I confirmed the contents by connected to it via Postico.

# 4. Delete your DB

You can't provision a second DB with Gigalixir while you have an existing one.

| You can only have one database per app because otherwise managing your DATABASE_URL variable would become trickier.

(source)[https://gigalixir.readthedocs.io/en/latest/database.html?highlight=backup#how-to-provision-a-free-postgresql-database]

You should follow [the instructions from Gigalixir's docs](https://gigalixir.readthedocs.io/en/latest/database.html?highlight=backup#how-to-delete-a-database), but to summarize:

```
gigalixir pg -a [app name]

# Grab the id from the output of ^

gigalixir pg:destroy -d $DATABASE_ID
```

Deletion will take a minute.

Now you don't have a database. Hooray! Just kidding.

# 5. Provision a new database

You can do this via the CLI or from the UI in Gigalixir. Really depends on what kind you want so I'm not going to put code here. What is important is that you can run `gigalixir pg -a [app name]` to check that your new db exists once you've created it.

# 6. Restore using your dump file

```
gigalixir pg -a [app name]

# - Grab the HOST
# - Grab the USERNAME
# - Grab the PASSWORD
# - Grab the ID

psql -h [HOST] -U [USERNAME] -d [ID]  < [path to DUMP file]

# You will be prompted for to enter a password (use the one you grabbed earlier).
```

# 7. You're done!

Check it works either by using the Gigalixr CLI to view the db, or use Postico with the full db url.



