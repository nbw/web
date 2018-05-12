---
title: "Converting from MySQL to Postgres with pgloader, for Heroku"
date: 2018-05-08T21:03:19-07:00
Description: "A guide to using pgloader with sprinkles of snags I encountered."
Tags: [ "pgloader", "MySQL", "Postgres", "Heroku"]
Categories: [ "code" ]

---

This post touches on what I did to convert an existing MySQL database to Postgres, which I eventually loaded onto Heroku. The hopes is that it'll highlight some of the hiccups that I ran into.
As much as I can I'm going to use real examples and not just quote documentation.

## Who is this article for?

For anyone migrating a relatively simple database from MySQL to Postgres. I didn't get too deep into pgloader (and there's seriously a lot you can customize), but I did run into a few snags.

## Complexity?

For beginner programmers like myself. I couldn't find a guide I liked and wish I had.

## Why MySQL to Postgres?

I recently decided to rewrite and move a website I maintain called [Treelib](https://treelib.ca) from a Digital Ocean Droplet to a Heroku Dyno. It was just something I wanted to try and it makes things easier for the scale of traffic I'd be dealing with. Heroku works out of the box with Postgres, but you can use add-ons like ClearDB if you'd prefer to use MySQL. I didn't prefer MySQL, and so the story begins…


## A few other things worth mentioning:

The rewrite I did was from a Sinatra Ruby app to a Phoenix Elixir app
Ecto (database client for Elixir) likes timestamps(), so moving over to Phoenix required some database migrations. I ran those migrations while still in MySQL, then ported over to Postgres.

# Step 1: Get pgloader
To convert from MySQL to Postgres I used pgloader, which is a great tool. There's plenty of documentation to sift through and the founder Dimitri seems to be quite active.

# Step 2: Converting to Postgres
Create a postgres db to move your MySQL db into. Then to convert use pgloader to convert to Postgres (it sounds easy because it is). Here's an example from pgloader's documentation:

```
createdb pagila
pgloader mysql://user@localhost/sakila postgresql:///pagila
```

What I actually ended up using was:

```
createdb treelib
pgloader mysql://root:mysql@localhost/treelib postgresql://localhost/treelib
# my local mysql db has a password of "mysql"
```

You can then test that it worked by running:

```
psql -h localhost -d [your database name]
# once in postgres, list all tables by running:
\dt
```

That's it! You might be in the clear at this point, but I ran into some snags (which is really why I wrote this article). And so the story continues…

# Step 3: Not quite there yet
These are a few of the issues I encountered.

## Schema issues, aka my Heroku woes

Now depending on your setup, when you run the \dt command above (in the postgres console) you'll notice that your schema is set to your database name or something similar. Here's what I saw for my Treelib project after converting to postgres:

```
                 List of relations
 Schema  |      Name    |   Type   | Owner
 - - - - + - - - - - - - - - -+ - - - -+ - - - - - - - -
 treelib | families     |   table  | nathanwillson
 treelib | genera       |   table  | nathanwillson
 treelib | photo_albums |   table  | nathanwillson
 ...
(full table list omitted)
```

It might still work in your dev environment, but Heroku doesn't like that. What you want is a "public" schema.
Dimitri, the maintainer of pgloader, addresses the solution here. But to summarize, the easiest (and most maintainable) way is create a .load file:

**my_load_file.load**

```
LOAD DATABASE
   FROM mysql://user@host/dbname
   INTO pgsql://user@host/dbname
ALTER schema 'dbname' rename to 'public';
```

Replace those mysql and pgsql addresses with your own database urls and don't forget to fill in 'dbname' with the schema you're replacing. Again, here's my load file looked like:

```
LOAD DATABASE
   FROM mysql://root:mysql@localhost/treelib
   INTO postgresql://localhost/treelib
ALTER schema 'treelib' rename to 'public';
```


Then run the pgloader via:

`pgloader my_load_file.load`

If everything checks out then psql into the database, then run `\dt`

```
                List of relations
 Schema |      Name    |   Type   | Owner
 - - - - + - - - - - - - - - -+ - - - -+ - - - - - - - -
 public | families     |   table  | nathanwillson
 public | genera       |   table  | nathanwillson
 public | photo_albums |   table  | nathanwillson
 ...
 (full table list omitted)
```

# Converting MySQL bigints (aka: casting considerations)

When converting from MySQL to Postgres, pgloader has a number of default cast types. The source code for those defaults is here and there's some documentation here. In my case, converting from MySQL bigint resolved as a numeric, but I'd rather it resolve as a Postgres bigint still. Here's what one of my tables looked like in MySQL:

```
MySQL:
+--------------+---------------------+------+-----+---------+
| Field        | Type                | Null | Key | Default |         |
+--------------+---------------------+------+-----+---------+
| id           | bigint(20) unsigned | NO   | PRI | NULL    |
| flickr_id    | bigint(20) unsigned | NO   |     | NULL    |
| photoset_id  | bigint(20) unsigned | NO   |     | NULL    |
...
(I've ommited the "Extra" column to make the text fit)
```

After initial conversion to Postgres:
```
Postgres:
Column        |         Type           |
--------------+------------------------+
 id           | bigint                 |
 flickr_id    | numeric                | <-- not what I wanted!
 photoset_id  | numeric                | <-- also not what I wanted!
...
(I've omitted the "Modifiers" column to make the text fit)
```

## Solution:
Easy fix? You bet. Just a one liner to add to your .load file (the one from Schema issues) and Bob's yer uncle:

```
CAST
    type bigint to bigint drop typemod;
And here's my final .load file:
LOAD DATABASE
   FROM mysql://root:mysql@localhost/treelib
   INTO postgresql://localhost/treelib
ALTER schema 'treelib' rename to 'public'
CAST
    type bigint to bigint drop typemod;
```

# Conclusion
So converting from MySQL to Postgres out of the box was pretty simple, but there were still a few hiccups to sift though. Hopefully this guide can offer some hints as to what to look out for.

To summarize what was covered:

- Converting from Postgres to MySQL using pgloader

- Schema "public" issues

- Overriding default cast types

Also thank you Dimitri for creating pgloader.