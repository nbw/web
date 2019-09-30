---
title: "Stripe Mock with Circle CI"
date: 2019-09-29T10:03:19-07:00
Description: "Adding Stripe Mock to your Circle CI config."
Tags: [ "Circle CI", "Stripe", "Stripe Mock", "Testing"]
Categories: [ "code" ]

---

# Using Stripe Mock with CircleCI

I'm assuming **[Circle CI](https://circleci.com/) 2.0 (or higher)**.

In your `.circle/config.yml` file, add the following line to your build:

```
- image: stripemock/stripe-mock:latest
```

That will pull the lastest docker image for your tests to run against.

If for whatever reason you want to test against an older docker tag (I wouldn't ever do this) then you'll find them listed [here](https://hub.docker.com/r/stripemock/stripe-mock).

I'm using Elixir so my final CircleCI config basically looks like:

```
version: 2
jobs:
  build:
    docker:
      - image: circleci/elixir:1.8.1
        environment:
          MIX_ENV: test
      - image: postgres:11
      - image: stripemock/stripe-mock:latest
    working_directory: ~/[APP DIRECTORY]
    steps:
      - checkout
      - run: mix local.hex --force
      - run: mix local.rebar --force
      - run: mix deps.get
      - run: mix ecto.create
      - run: mix ecto.migrate
      - store_test_results:
          path: _build/test/lib/[APP PATH]
```

I love that [Stripe Mock](https://github.com/stripe/stripe-mock) is a thing. I can write tests against a mock API that actually maintained by [Stripe](https://stripe.com).

🍻





