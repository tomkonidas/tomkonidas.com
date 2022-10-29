---
title: "Repo.fetch/3"
date: 2022-10-29T11:00:10-04:00
draft: false
type: "post"
tags: ["elixir", "ecto"]
---

Every project I find myself writing the same bit of code.
I love to use [case](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#case/2)
and [with](https://hexdocs.pm/elixir/Kernel.SpecialForms.html#with/1)
statements for conditional control flow across my code base and much prefer to
use ok/error tuples over pattern matching on `nil`.

I find it weird that Ecto does not have this included in their API already, however luckily for us,
it is easy to key in and extend the Repo module.

### What I add to every Ecto project

We can use what Ecto gives us out of the box and just extend our Repo module
with these two little wrapper functions.

```elixir
defmodule MyApp.Repo do
  use Ecto.Repo,
    otp_app: :my_app,
    adapter: Ecto.Adapters.Postgres

  def fetch(queryable, id, opts \\ []) do
    case get(queryable, id, opts) do
      nil -> {:error, :not_found}
      record -> {:ok, record}
    end
  end

  def fetch_by(queryable, clauses, opts \\ []) do
    case get_by(queryable, clauses, opts) do
      nil -> {:error, :not_found}
      record -> {:ok, record}
    end
  end
end
```

Now calling `Repo.fetch/3` feels like a native Ecto implementation and we do
not ever need to worry about custom logic as we are just re-using what is
already there and passing on all the arguments.
