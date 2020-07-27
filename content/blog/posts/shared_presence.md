---
title: "Global/Shared Presence {:error, :nopresence}"
date: 2020-05-26T13:48:09-08:00
Description: "Using Presence as sharable state"
Tags: ["Elixir", "Elixir-lang", Phoenix", "Phoenix Framework", "Presence", "LiveView", "Channels"]
Categories: ["code"]
draft: false

---
<style>
  .text-center { margin-top: 0;
    text-align: center;
  }
  h1 {
    font-size: 1.8em
  }
</style>


This post talks about how to use Presence as a sharable state object that all connect users can modify and manage. In a sense, a singleton Presence object.

---



#Problem

Only the socket/process tracking a Presence object can update it, otherwise a `{:error, :nopresence}` will occur.  This is by design, but we can workaround it :).

---



# Motivation

Recently I was working on an [agile poker app](https://agile-poka.herokuapp.com/) written with [Elixir](https://elixir-lang.org/)'s  [Phoenix Framework](https://www.phoenixframework.org/) that uses [Presence](https://hexdocs.pm/phoenix/Phoenix.Presence.html#content), [Channels]([Channels docs](https://hexdocs.pm/phoenix/channels.html)), and [LiveView](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html). Users join a room and vote on durations for upcoming tasks in the next sprint (agile stuff).  [The agile poker app GitHub repo is here](https://github.com/nbw/poker/).

<img style="display:block; margin: auto; text-align:center" src="/images/poker.png" />

When a user joins a room they should know the current state of the room (has the game started? what stage is the game at? etc.). The easiest solution for this may be persist the state somewhere in a database, but I wanted a single-session based room where nothing is saved permanently. It occurred to me that I could use Presence to manage the overall state of the game (in addition to each individual user's status).

* With Presence you get the benefits of channels with nice callbacks to update everything
* Presence is just really easy with minimal code



----



# Presence is great but..

[Presence](https://hexdocs.pm/phoenix/Phoenix.Presence.html#content) is really great for tracking processes/channels for individual users, but having a "shared presence object" that everyone can update and modify is a challenge.

To update a Presence instance [the docs](https://hexdocs.pm/phoenix/Phoenix.Presence.html#c:update/3) suggest update/3 or update/4. What that often looks like is

```
Presence.update(
	self(),
	topic,
	key,
	state
)
```

The caveat is that the **only the socket/PID tracking the presence object can update/modify it**. Otherwise you'll receive a `{:error, :nopresence}` response back. Maybe this assumed knowledge, but it's not mentioned in the docs.  This error actually comes from Phoenix's PubSub [Tracker.Shard module](https://github.com/phoenixframework/phoenix_pubsub/blob/ca2b47c8cf31324b0bf96cea862058f783a3e7bd/lib/phoenix/tracker/shard.ex#L508).

The only post that I found mentioning with this `:nopresence` issue was [here](https://elixirforum.com/t/presence-update-loop-works-once-then-returns-error-nopresence/22371/2).

----



# Sharable Presence object workaround

The workaround I used was to <u>register a process with a name</u> so that it can be retrieved by any user.

**Tracking:**

```
Process.register(self(), :insert_name_here)

Presence.track(
	self(),
	topic,
	key,
	state
)
```


**Update:**

```
pid = Process.whereis(:insert_name_here)

Presence.update(
  pid,
  topic),
  key,
  state
)
```


## Caveats

There is a limit to the number of atoms you can have in an Elixir app, thus there is a limit to the number of processes you can register. There are ways around that, ways to unregister processes, etc., but I'm not going to get into that. Mostly because I didn't bother, but that's something I'd monitor if I was making some more production level.

## Code

[Here's the real-world implementation I used for the poker app](https://github.com/nbw/poker/blob/develop/lib/poker_web/live/room_live_view.ex).

----



# Alternatives

1. You could probably use an **Agent** or better a **GenServer** to manage and store state, but you'll end up having to write a bunch of code to issue callbacks and basically do all the nice things that Presence/PubSub/Channels gives you.

2. You could save the "game state" to each user's Presence object. This would work, but you just have to make sure that you update all tracked users.

   To do this:

   1. Channels/PubSub to Broadcast a message which is picked up by each connected socket
   2. Each socket can update its own related Presence object

   For an example, look at what I did with "reset" in [the agile poker code](https://github.com/nbw/poker/blob/develop/lib/poker_web/live/room_live_view.ex).


