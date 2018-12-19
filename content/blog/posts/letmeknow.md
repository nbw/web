---
title: "Let Me Know"
date: 2018-11-28T16:44:49-08:00
Description: "A simple CLI tool to get notified when a command is complete - for Mac."
Tags: ["OSAScript", "Mac", "Apple", "bash", "Programming"]
Categories: ["code"]
draft: false

---
**Languages:** Bash, OSAScript

I've always wanted a simple notification CLI tool to poke me when a task is complete. There few tools out that that do this, the most impressive being [Variadico's Noti](https://github.com/variadico/noti). That one is written in Golang using [Cobra](https://github.com/spf13/cobra) and supports basically every notification service you can think of -- it's well done in my opinion. But still, for my use case it was a bit much. I wanted something incredibly simple. It should do one job: _tell me when something is done._

So I looked to bash and <u>I'm on a Mac</u>, which quickly lead me to find out that you can trigger a notification by calling Mac OSAScript directly from the terminal.

**OSAScript** is Apple's scripting language for exectuing small little tasks on your local computer. It's actually quite handy, comes with a GUI if you want, and give you access to a bunch of things such as notifications, volume, opening finder, etc..

Here's the most simple way to execute a notifcation from the command line:

```
osascript -e 'display notification "World." with title "Hello"'
```

<img style="display:block; margin: auto; text-align:center" src="/images/lmk_helloworld.gif" />

Swap out `notification` for `dialog` and an ok/cancel modal will pop up. It's that easy.

# Let Me Know

So triggering a notification is easy. The rest was putting together a script that runs some code first, and then sends a notication. With that in mind [Let Me Know](https://github.com/nbw/letmeknow) was born.

![lmk demo](/images/lmk_demo.gif)

Throw in a bit of error handling and voila this is the whole script:

```
notif(){
  title=$1
  shift
  script='display notification "'$*'" with title "'$title'"'
  osascript -e "$script"
}

$* && (notif "Complete" $*) || (notif "Error" $*)

```

Stupidly simple and does only what I needed. Check it out on [github](https://github.com/nbw/letmeknow) if you're interested.


