---
title: "Repo.transact/2 (The Case Against Ecto.Multi)"
date: 2023-04-29T10:59:51-04:00
draft: false
type: "post"
tags: ["elixir", "ecto", "transaction", "multi"]
---

After reading [Towards Maintainable
Elixir](https://medium.com/very-big-things/towards-maintainable-elixir-the-core-and-the-interface-c267f0da43)
by Saša Jurić and hearing about his famous `Repo.transact` in some of his
talks, I decided it was time to explore this for myself.

This post takes into account that you (the reader) are aware and know why and
when to use [Ecto.Multi](https://hexdocs.pm/ecto/Ecto.Multi.html). But for
those unfamiliar, the TL;DR is you would use an `Ecto.Multi` when you want to
perform multiple transaction that you want to be committed to the database in
one shot. Meaning, if one of the transactions fails, you would want to revert
all other transactions in the run.


## The Problem with Ecto.Multi

Lets get something straight, **there is nothing wrong with using Ecto.Multi**
in your codebase. If it works for you, then it works. However after working
with it in multiple codebases I have started to see a common theme: it is very
noisy and can be sometimes hard to follow and DRY up. You can get around it by
using a lot of private functions to support the Ecto.Multi, but then your
module just has a tons of wrapper functions.


## What is Repo.transact/2?

> The function Repo.transact is our small wrapper around
> [Repo.transaction/2](https://hexdocs.pm/ecto/Ecto.Repo.html#c:transaction/2).
> This function commits the transaction if the lambda returns `{:ok, result}`,
> rolling it back if the lambda returns `{:error, reason}`. In both cases, the
> function returns the result of the lambda. We chose this approach over
> Ecto.Multi, because we’ve experimentally established that multi adds a lot of
> noise with no real benefits for our needs.
>
> -- <cite>Saša Jurić</cite>


## Function definition

Saša never gives out the implementation of the function but I came up with
this, and it works great; Exactly as you would expect.

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres

  @doc """
  A small wrapper around `Repo.transaction/2'.

  Commits the transaction if the lambda returns `:ok` or `{:ok, result}`,
  rolling it back if the lambda returns `:error` or `{:error, reason}`. In both
  cases, the function returns the result of the lambda.
  """
  @spec transact((-> any()), keyword()) :: {:ok, any()} | {:error, any()}
  def transact(fun, opts \\ []) do
    transaction(
      fn ->
        case fun.() do
          {:ok, value} -> value
          :ok -> :transaction_commited
          {:error, reason} -> rollback(reason)
          :error -> rollback(:transaction_rollback_error)
        end
      end,
      opts
    )
  end
end
```

## Ecto.Multi vs Repo.transact

Lets say we want to take the typical user registration flow as an example. If
we want to insert a user, log the action to an audit table and also enqueue a job
to send a confirmation email.

We would have something like this using an Ecto.Multi:

#### Ecto.Multi implementation

```elixir
def register_user(params) do
  Multi.new()
  |> Multi.insert(:user, Accounts.new_user_changeset(params))
  |> Multi.insert(:log, fn %{user: user} ->
    Logs.log_action(:user_registered, %{user: user})
  end)
  |> Multi.insert(:email_job, fn %{user: user} ->
    Mailer.enqueue_email_confirmation(user)
  end)
  |> Repo.transaction()
  |> case do
    {:ok, %{user: user}} ->
      {:ok, user}

    {:error, _failed_operation, failed_value, _changes_so_far} ->
      {:error, failed_value}
  end
end
```

As we can see, it is not the worst, but once we see the `Repo.transact/2` way,
it will be clear which is better.

#### Repo.transact implementation

```elixir
def register_user(params) do
  Repo.transact(fn ->
    with {:ok, user} <- Accounts.create_user(params),
         {:ok, _log} <- Logs.log_action(:user_registered, user),
         {:ok, _job} <- Mailer.enqueue_email_confirmation(user) do
      {:ok, user}
    end
  end)
end
```

As you can see, it is much shorter and easier to read. Another big benefit is
that we do not need to go down to the changeset level for inserting, we could
use our functions that perform `Repo.insert`s in them
(`Accounts.new_user_changeset/1` vs `Accounts.create_user/1`). This lets us
compose many functions together from outside the context modules without having
the need to expose your changeset functions.


The end result is the same, but it is a lot easier to read what is going on IMHO.
