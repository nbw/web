---
title: "Using Fly.io's Multi-region databases with Elixir's Phoenix Framework"
date: 2021-09-24T16:44:49-08:00
Description: "A guide to using Fly's multi-region read-replica DB's with Elixir's Phoenix Framework"
Tags: ["Elixir", "Phoenix", "Flyio", "Databases"]
Categories: ["code"]
draft: false
---

<style>
  blockquote {
    border: 1px solid #FDA694;
    border-radius: 5px;
    background-color: #FFE2DC;
    padding: 1rem;
  }
  blockquote p {
    margin-top: 0 !important;
  }
</style>

![fly phoenix](/images/fly_phoenix.jpg)

In this article we'll be using Fly.io to host a Phoenix web app that's connected to multiple Postgres read-replica databases (1 read/write master and a number of read-only replicas). This article assumes that:

- You've installed Fly's CLI, authenticated, etc.
- You're familiar with the Phoenix framework and with Elixir

### Core Documentation

In a sense this article is a combined implementation of:

- [Fly.io's Multi-region Database documentation](https://fly.io/docs/getting-started/multi-region-databases/#library-support)
- Ecto's [Replicas and Dynamic Repositories documentation](https://hexdocs.pm/ecto/replicas-and-dynamic-repositories.html)

But since database configuration with a Fly app happens at runtime there were a few things to look out for. **If you're serious about implementing a multi-region database app then you should use read those two articles first**.

At the end of the article I'll talk about benchmarking/performance.

## Goals

- The implementation should require minimal configuration to add a new region
- The implementation doesn't interfere with development or testing environments

## Final code

The final code is here:
[https://github.com/nbw/phoenix-multi-database-example](https://github.com/nbw/phoenix-multi-database-example)

---

## 0. Table of Contents

- [1. Deploying a Phoenix app to Fly.io](#1-deploying-a-phoenix-app-to-flyio)
  - [IMPORTANT: When creating a database...](#important--when-creating-a-database)
  - [Creating test data](#creating-test-data)
- [2. Creating multi-region databases](#2-creating-multi-region-databases)
  - [Creating Replicas](#creating-replicas)
  - [Connect the cluster to your app](#connect-the-cluster-to-your-app)
- [3. Configure Phoenix/Elixir (but really Ecto)](#3-configure-phoenix-elixir--but-really-ecto-)
  - [Region helper](#region-helper)
  - [Config files](#config-files)
  - [Replicas module](#replicas-module)
  - [Supervise our replica Repos](#supervise-our-replica-repos)
- [4. Using a read-replica in code](#4-using-a-read-replica-in-code)
- [5. Performance/Benchmarking](#5-performance-benchmarking)
- [6. Improvements/Optimizations](#6-improvements-optimizations)
  - [Repos Supervision tree](#repos-supervision-tree)
- [7. Conclusion](#7-conclusion)

---

## 1. Deploying a Phoenix app to Fly.io

Before getting into multi-region databases, we need to setup and deploy a basic Phoenix app to Fly so that we have something to test. I'll defer to the following links:

- Fly.io's [Running an Elixir app](https://fly.io/docs/getting-started/elixir/) documentation
- [HelloElixir](https://github.com/fly-apps/hello_elixir-dockerfile) sample repo maintained by Fly.io

The TLDR is:

- Create a `release.ex` file
- Create a `Dockerfile` file
- Create a `.dockerignore` file
- Create a `runtime.exs` file, then move Endpoint and Repo config into that file (from prod.ex)
- Comment out the import of `prod.secret.exs` in your prod.ex file
- Run `fly launch` which will create your app on [Fly.io](http://fly.io) and also generate a fly.toml file. You'l need to adjust the contents of the fly.toml file (refer to docs)
- Follow the _Prepare to Deploy_ docs to generate and set a secret, etc.
- Finally, deploy to Fly.io. Adjust recipient accordingly. Fingers crossed that it works!

### IMPORTANT: When creating a database...

The docs say to create a database via `fly postgres create`, but in our case we'll use the [command from the Multi-Region Database](https://fly.io/docs/getting-started/multi-region-databases/#create-a-postgresql-cluster) docs instead:

```elixir
fly pg create --name [insert cluster name here] --region [insert region here]

# attach the DB to your app by running the following
# from your app's root folder:
fly pg attach --postgres-app [cluster name here]
```

The name can be anything but will have to be unique.

For a list of regions, refer to [the Region docs](https://fly.io/docs/reference/regions/).

### Creating test data

The fastest way to create scaffolding for test data is to use one of Phoenix's generators. For this example I'll be using an example taken straight from the docs. Run the following from the root of your Phoenix project to generate an Account's context, User schema, HTML templates, etc..

```elixir
mix phx.gen.html Accounts User users name:string age:integer
```

Don't forget to add an endpoint to your `router.ex` file and migrate your local database.

```elixir
# router.ex file

resources "/users", UserController

# migrate from console

mix ecto.migrate
```

We should have everything we need now to generate data from the `/users` endpoint of your app and/or from IEX console.

> 💡 VERIFY:
>
> - You should have a Phoenix web app deployed to Fly.io.
> - You should be able to create and save something. If you're following this example then create some users with names and an age set.

## 2. Creating multi-region databases

If you ran the command mentioned in the last section (`fly pg create --name [insert cluster name here] --region [insert region here]`) then you've already created a leader DB for writes and a replica for reads.

> 💡 VERIFY: run the following command to check for the leader and replica:

```bash
fly status -a [your db cluster name]

# Example Output:

> fly status -a my-postgres

WARN app flag 'my-postgres' does not match app name in config file 'my-app-name'
? Continue using 'my-postgres' Yes
App
  Name     = my-postgres
  Owner    = personal
  Version  = 2
  Status   = running
  Hostname = my-postgres.fly.dev

Instances
ID       TASK VERSION REGION DESIRED STATUS            HEALTH CHECKS      RESTARTS CREATED     23h57m ago
e9e10050 app  2       nrt    run     running (leader)  3 total, 3 passing 0        23h59m ago
7435e9e5 app  2       nrt    run     running (replica) 3 total, 3 passing 0        23h59m ago
```

### Creating Replicas

For now let's create a single replica via the following command

```bash
fly volumes create pg_data -a [your db cluster name] --size 10 --region [insert region name]
```

As I'm based out of Tokyo, I created a replica in Chicago:

```bash
fly volumes create pg_data -a my-postgres --size 10 --region ord
```

**Important:** Add more VM's. If you don't then you won't see the replica you created.

```bash
fly scale count 3 -a [your db cluster name]
```

Why 3? We have one leader (nrt), the local replica of the leader (nrt), and the new region we just created (ord).

> 💡 VERIFY: Check that you see the new replica when listing VM's:

```bash
> fly status -a [your db cluster name]

# Example output:

App
  Name     = my-postgres
  Owner    = personal
  Version  = 2
  Status   = running
  Hostname = my-postgres.fly.dev

Instances
ID       TASK VERSION REGION DESIRED STATUS            HEALTH CHECKS      RESTARTS CREATED
f9ed37a0 app  2       ord    run     running (replica) 3 total, 3 passing 0        23h57m ago
e9e10050 app  2       nrt    run     running (leader)  3 total, 3 passing 0        23h59m ago
7435e9e5 app  2       nrt    run     running (replica) 3 total, 3 passing 0        23h59m ago
```

### Connect the cluster to your app

If you haven't already, attach the cluster to your app by running the following from your app's root folder:

```bash
fly pg attach --postgres-app [your cluster name]
```

> 💡 VERIFY: Even though we haven't connected the read-replicas to our app, we should still be able to read/write from the primary database.

---

## 3. Configure Phoenix/Elixir (but really Ecto)

There are number of ways to configure read-replicas with a Phoenix app, but specifically with [Fly.io](http://fly.io) we have to pay attention to order of events since DB configuration happens at runtime.

### Region helper

First, we need a way of finding the current region so we know which replica-database to use.

Fly automatically sets a `FLY_REGION` environment variable for any app instance. We could grab it using `System.fetch_env("FLY_REGION")` but for the sake of keeping things DRY (and wrapping 3rd-party dependancies) let's create simple helper module.

Create a `/fly` folder and a `region.ex` file:

```elixir
# lib/my_app/fly/region.ex
defmodule MyApp.Fly.Region do
  @moduledoc """
  Helper module for Fly Regions

  [Fly Documenation](https://fly.io/docs/reference/regions/)
  """

  def current do
    System.fetch_env("FLY_REGION")
    |> case do
      {:ok, region} -> region
      _ -> nil
    end
  end
end
```

This module might come in handy later if we want to display the current region somewhere on the site.

### Config files

In the `runtime.exs` file there should already be configuration the project's Repo module. Setting up a read-replica requires:

- Creating module names for each replica (we'll define the modules later)
- Adjusting the `hostname` and `port` of each replica

Add the following to your `runtime.exs` file (ignore the `MyApp.Repo.Replicas` for now):

```elixir
# runtime.exs

  # Define read-replica regions (not including the primary)
  regions = ["ord"]

  # Create Repo modules for each region
  # Ex:
  # %{
  #    "ord" => MyApp.Repo.Ord
  #  }
  replicas =
    Enum.reduce(regions, %{}, fn region, acc ->
      replica = Module.concat(MyApp.Repo, String.capitalize(region))
      Map.put(acc, region, replica)
    end)

  # Replicas Config
  config :my_app, MyApp.Repo.Replicas,
    replicas: replicas

  # Configure each Replica
  for {region, repo} <- replicas do
    # Add region to the hostname
    {_, opts} =
      Ecto.Repo.Supervisor.parse_url(database_url)
      |> Keyword.get_and_update!(:hostname, fn hostname -> {hostname, "#{region}.#{hostname}"} end)

    # Replica's use the port 5433
    opts = Keyword.replace!(opts, :port, 5433)

    opts =
      opts ++
        [
          socket_options: [:inet6],
          pool_size: String.to_integer(System.get_env("POOL_SIZE") || "10")
        ]

    config :my_app, repo, opts
  end
```

The code above configures each read-replica to:

- use a hostname with the region appended:
  - ex. _my-postgres.internal_ —> _ord.my-postgres.internal_
- use port `5433`
- the rest of the DB config (username, password, etc.) is the same as the primary

**Note**: I've used `Ecto.Repo.Supervisor.parse_url/1` to break down the database url into it's components ([code](https://github.com/elixir-ecto/ecto/blob/master/lib/ecto/repo/supervisor.ex)). Using a third-party method this way isn't an _amazing_ way of doing things. We might write our own version of this method to make it's future proof.

### Replicas module

We need some methods to find the right replica database. For the sake of keeping things clean I've created a `/repo` folder and Replicas module (ignore `compile_replicas/0` for now):

```elixir
# /lib/my_app/repo/replicas.ex

defmodule MyApp.Repo.Replicas do
  @moduledoc """
  Handles all replica related tasks
  """

  @type region() :: String.t()
  @type replica() :: module()

  @doc """
  Returns a Map of replica module names with
  """
  @spec replicas() :: %{region() => replica()}
  def replicas, do: Application.get_env(:my_app, __MODULE__)[:replicas] || %{}

  @doc """
  Returns a list of regions configured in runtime.exs
  """
  @spec regions() :: [region()]
  def regions, do: Map.keys(replicas())

  @doc """
  Helper method for returning a list of replica modules
  """
  @spec list_replicas() :: [replica()]
  def list_replicas, do: Map.values(replicas())

  @doc """
  Returns a Repo for the current region
  """
  @spec replica() :: module()
  def replica() do
    MyApp.Fly.Region.current()
    |> replica()
  end

  @doc """
  Returns a Repo for the a given region, returns primary Repo as a default
  """
  @spec replica(region()) :: module()
  def replica(region) do
    if Enum.member?(regions(), region) do
      replicas()[region]
    else
      MyApp.Repo
    end
  end

  @doc """
  Generate Repo modules for each replica
  """
  def compile_replicas do
    for replica <- list_replicas() do
      defmodule replica do
        use Ecto.Repo,
          otp_app: :my_app,
          adapter: Ecto.Adapters.Postgres,
          read_only: true
      end
    end
  end
end
```

Then add the following to your app's Repo module:

```elixir
# /lib/my_app/repo.ex

defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres

  alias MyApp.Repo.Replicas
  defdelegate replica(), to: Replicas
end
```

### Supervise our replica Repos

As explained in Ecto's [Replicas and Dynamic Repositories documentation](https://hexdocs.pm/ecto/replicas-and-dynamic-repositories.html), we need to add the replica Repos to the application supervision tree.

In Ecto’s example, each replica module was generate inside the main Repo module, _but we need something that works with our runtime configuration (believe me, I tried all sorts of ways to get this to work)_. Otherwise when our Application’s supervision tree starts it won’t find any of our generated replica modules (because they don’t exist yet). This is where `MyApp.Repo.Replicas.compile_replicas/0` becomes useful.

In your app's `application.ex` create the modules first, then supervise them. The `start` function should look like the following:

```elixir
defmodule MyApp.Application do

  def start(_type, _args) do
	  # Create our replica modules first
    MyApp.Repo.Replicas.compile_replicas()

    children =
      [
        # Start the Ecto repository
        MyApp.Repo,
        # Start the Telemetry supervisor
        MyAppWeb.Telemetry,
        # Start the PubSub system
        {Phoenix.PubSub, name: MyApp.PubSub},
        # Start the Endpoint (http/https)
        MyAppWeb.Endpoint
        # Start a worker by calling: MyApp.Worker.start_link(arg)
        # {MyApp.Worker, arg}
      ] ++ MyApp.Repo.Replicas.list_replicas()

    # See https://hexdocs.pm/elixir/Supervisor.html
    # for other strategies and supported options
    opts = [strategy: :one_for_one, name: MyApp.Supervisor]
    Supervisor.start_link(children, opts)
  end

end
```

---

## 4. Using a read-replica in code

If you've been following along and created an Accounts context then we can adjust our GET methods to use a read-only replica database. Adjust `list_users` and `get_user` to the following:

```elixir
# /lib/my_app/accounts.ex

def list_users do
  Repo.replica().all(User)
end

def get_user!(id), do: Repo.replica().get!(User, id)
```

That's it! Your app will now read from the read-replica if it's region matches one of your replicas.

---

## 5. Performance/Benchmarking

For benchmarking I used [benchee](https://github.com/bencheeorg/benchee).

**I'm based in Tokyo (local region: nrt).**

You'll want to set up SSH with Fly so that you can remote console into your app. [Here](https://fly.io/docs/getting-started/elixir/#getting-an-iex-shell-into-a-running-node) are some relevant docs.

```bash
fly ssh console

....

app/bin/my_app remote
```

Once you've got an IEx console then you can run the following:

```elixir
%{"nrt" => MyApp.Repo}
|> Map.merge(MyApp.Repo.Replicas.replicas())
|> Enum.map(fn {region, repo} ->
   {region, fn -> repo.all(MyApp.Accounts.User) end}
end)
|> Enum.into(%{})
|> Benchee.run(); nil;

# Note: the ; nil; is just to quell console output of the return value.
```

Here's a comparison of regions when connecting from Japan:

```bash
Benchmark suite executing with the following configuration:
warmup: 2 s
time: 5 s
memory time: 0 ns
parallel: 1
inputs: none specified
Estimated total run time: 35 s

Benchmarking ams...
Benchmarking atl...
Benchmarking nrt...
Benchmarking ord...
Benchmarking syd...

Name           ips        average  deviation         median         99th %
nrt         580.10        1.72 ms    ±32.92%        1.61 ms        3.23 ms
ord           7.23      138.27 ms    ±25.76%      127.82 ms      256.94 ms
syd           6.49      154.09 ms    ±29.89%      137.27 ms      277.29 ms
atl           5.13      194.94 ms    ±32.01%      168.88 ms      339.00 ms
ams           3.56      280.98 ms    ±36.08%      219.86 ms      440.07 ms

Comparison:
nrt         580.10
ord           7.23 - 80.21x slower +136.55 ms
syd           6.49 - 89.39x slower +152.37 ms
atl           5.13 - 113.09x slower +193.22 ms
ams           3.56 - 163.00x slower +279.25 ms

```

We can see that connecting to a Japan replica (nrt) is the fastest based on my current location, whereas Amsterdam is the slowest (ams).

---

## 6. Improvements/Optimizations

### Repos Supervision tree

In this example we’re supervising our main Repo and replica Repos from the top level Application. It would probably make more sense to create a Repos supervision tree that monitors our Repo and replica Repo modules. Then add the Repos supervision tree to our Application supervision tree.

This way we’d then have the opportunity to specify a restart strategy specifically for Repos (probably still one for one). It also just keeps our Application module clean.

### Module Complilation

Calling a function like `compile_replicas` from the Application module is not my favourite solution and I'm sure there is a more appropriate solution.

---

## 7. Conclusion

There's still tons of room for improvement, but this article should serve as a proof-of-concept that can be turned into a library eventually.

Let's check the goals:

- **The implementation should require minimal configuration to add a new region:**

Yes. We only have to change the "region" list in runtime.exs.

- **The implementation that doesn't interfere with development or testing environments**

Yes. Our local and test environments continue to use a single database.

Should you use a multi-region database setup? It really depends on your use case.

Good luck!

