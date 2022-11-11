---
title: "Partitioning Elixir Tests with Github Actions"
date: 2022-11-11T00:44:49-08:00
Description: "Speed up tests with partitioning"
Tags: ["Elixir", "Exunit", "Github Actions", "Test"]
Categories: ["code"]
banner: "/images/reusable_workflows_banner.png"
draft: false
---

![banner](/images/reusable_workflows_banner.png)

[Mark Ericksen](https://twitter.com/brainlid) recently published an article "[Github Actions for Elixir CI](https://fly.io/phoenix-files/github-actions-for-elixir-ci/)", which provides a succinct Github Workflow template for running Elixir CI tests.

The tests for the app I'm working on take approximately 15-16 minutes. **So, how can we get that number lower?** ⌛

[Faster test execution in Elixir](https://bartoszgorka.com/faster-test-execution-in-elixir) by Bartosz Górka is a great place to start if you're looking for tips. This post specifically is about their 4th recommendation: **partitions**.

## Partition tests

The [ExUnit docs](https://hexdocs.pm/mix/main/Mix.Tasks.Test.html) explain the `--partition` flag in detail, but the summary is:

```
MIX_TEST_PARTITION=1 mix test --partitions 4
MIX_TEST_PARTITION=2 mix test --partitions 4
MIX_TEST_PARTITION=3 mix test --partitions 4
MIX_TEST_PARTITION=4 mix test --partitions 4
```

We need two things:

- partition number: `MIX_TEST_PARTITION`
- total number of partitions `--partitions`

---

## Partition tests with Github Actions

We could save copies of Mark's `.github/workflows/elixir.yaml` script (example: `elixir_1.yaml`, `elixir_2.yaml`, ..) and manually set `MIX_TEST_PARTITION` and `--partitions` in each file, but that's not really a sustainable solution. Instead we'll use "resusable workflows".

### Resusable Workflows

Github's [Reusable Workflow docs are here](https://docs.github.com/en/actions/using-workflows/reusing-workflows).

We can modify Mark's script to make it configurable:

```yaml
# resusable_elixir.yaml

name: Reusable Elixir CI

# https://fly.io/phoenix-files/github-actions-for-elixir-ci/

on:
  workflow_call:
    secrets:
      partition:
        required: true
      partitions:
        required: true

env:
  MIX_ENV: test

permissions:
  contents: read

jobs:
  test:
    # CONTENTS OMITTED (read Mark's post!)
    steps:
      # ...
      - name: Run tests (${{ secrets.partition }}/${{ secrets.partitions }})
        run: MIX_TEST_PARTITION=${{ secrets.partition }} mix test --partitions ${{ secrets.partitions }}
```

#### 1. `workflow_call` event:

Instead of on pull request, use the `workflow_call` event.

```
on:
  workflow_call:
```

#### 2. Utilize secrets as arguments:

We want to specify the partion number (`MIX_TEST_PARTITION`) and number of partiions (`--partitions`) and make them required.

```
on:
  workflow_call:
    secrets:
      partition:
        required: true
      partitions:
        required: true
```

#### 3. Run tests using partitions

Lastly, modify the `mix test` call to use the secrets configure partitions. I've also changed the `name` so that we know what partition exactly is running.

```
  - name: Run tests (${{ secrets.partition }}/${{ secrets.partitions }})
    run: MIX_TEST_PARTITION=${{ secrets.partition }} mix test --partitions ${{ secrets.partitions }}
```

### Call the Resusable Workflow

We can call resuable workflow as many times as we'd like with a separate script. This time we'll use the pull request trigger.

```yaml
# elixir.yaml

name: Elixir CI

on: [pull_request]

env:
  MIX_ENV: test

permissions:
  contents: read

jobs:
  test_1-4:
    uses: ./.github/workflows/reusable_elixir.yml
    secrets:
      partition: "1"
      partitions: "4"
  test_2-4:
    uses: ./.github/workflows/reusable_elixir.yml
    secrets:
      partition: "2"
      partitions: "4"
  test_3-4:
    uses: ./.github/workflows/reusable_elixir.yml
    secrets:
      partition: "3"
      partitions: "4"
  test_4-4:
    uses: ./.github/workflows/reusable_elixir.yml
    secrets:
      partition: "4"
      partitions: "4"
```

## Result

Tests execute in partitions and the 15-16 minutes is divided into 3-4 minute chunks:

![tests](/images/reusable_workflows.png)

## Conclusion

Resusable workflows are quite powerful. I'm still new to Github Actions and I'm sure there are improvements that can be made. I'll do my best to update this article with any insights I have.
