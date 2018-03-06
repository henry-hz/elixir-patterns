Ecto Multi
==========

Each step of the transaction passes the Multi.new result down through the chain until eventually, after building up our transaction a piece at a time, we tell our Repo to actually make the calls.

If at any point in the steps a changeset has errors or the database call itself fails for one reason or another, none of the database calls will be persisted. The particular call that fails the transaction will be returned in a tuple with the form of {:error, name_of_call, failed_value,changes_that_succeeded}. The failed_value will contain the changeset errors (if using a changeset) or any other error values the call returned. changes_that_succeeded will contain the results of any previous operations that were successful. However, as the docs state,

any successful operations would have been rolled back

because the transaction itself failed.

This is now where we can take a look at the enter_door function. If the transaction fails, we’ve already seen how it will return the {:error, ...} tuple for us to deal with how we’d like. If it succeeds, it will return a tuple with {:ok, map}. In map, we can access the success values of each of the individual operations from the value of what we named that particular operation. So in our example the :entry key in map would correspond with the result of the operation:



```Elixir
defmodule Guard do
  def enter_door(person, door) do
    case entry_transaction(person, door) do
      {:ok, %{entry: entry, log: log, person: person}} ->
        Logger.debug("Success on all three!")
      {:error, :log, _failed_value, _changes_successful} ->
        Logger.debug("Failed to save Log")
      {:error, :person, _failed_value, _changes_successful} ->
        Logger.debug("Failed to save Person")
      {:error, :entry, _failed_value, _changes_successful} ->
        Logger.debug("Failed to save Entry")
    end
  end
  
  def entry_transaction(person, door) do
    Multi.new
    |> Multi.insert(:entry, Entry.changeset(%Entry{}, %{door_id: door.id, person_id: person.id}})
    |> Multi.update(:person, Person.increase_entry_count_changeset(person))
    |> Multi.insert(:log, Log.changeset(%Log{}, %{text, "entry"}))
    |> Repo.transaction()
  end
end
```

Another example:

```Elixir
defmodule Store.Customer do
  alias Ecto.Multi
  alias Store.{Repo, Profile}
  alias Security.User

  def add(params) when is_map(params) do
    multi =
      Multi.new
      |> Multi.run(:user, fn _ -> User.add(params) end)
      |> Multi.run(:profile, fn %{user: user} ->
        params
        |> Map.put_new("user_id", user.id)
        |> Profile.add()
      end)

    case Repo.transaction(multi) do
      {:ok, result} -> {:ok, result}
      {:error, :user, changeset, %{}} -> {:error, :user, changeset}
      {:error, :profile, changeset, %{}} -> {:error, :profile, changeset}
    end
  end
end
```


Using Ecto.Multi.run to execute Arbitrary functions:

```Elixir
defmodule Guard do
  alias Ecto.Multi

  def enter_door(person, door) do
    case entry_transaction(person, door) do
      {:ok, %{entry: entry, log: log, person: person}} ->
        Logger.debug("Success on all four!")
      {:error, :allowed, _failed_value, _changes_successful} ->
        Logger.debug("Failed to pass allowed check")
      # ... other pattern matched failures
    end
  end
  
  def entry_transaction(person, door) do
    Multi.new
    |> Multi.insert(:entry, Entry.changeset(%Entry{}, %{door_id: door.id, person_id: person.id}})
    |> Multi.update(:person, Person.increase_entry_count_changeset(person))
    |> Mutli.run(:allowed, fn(%{person: person}) ->     # NEW CODE
      if person.entry_count > 10 do                     # 
        {:error, "Person entered more than 10 times"}   # 
      else                                              # 
        {:ok, "continue"}                               # 
      end                                               # 
    end)                                                # 
    |> Multi.insert(:log, Log.changeset(%Log{}, %{text, "entry"}))
    |> Repo.transaction()
  end
end
```
Ecto.Multi.run expects to receive a tuple containing either {:ok, message} or {:error,message} in order to determine whether to continue the transaction.

We can see above that we pattern match on the Person which has the result of successfully running the update operation above it. So if before the operation the Person had 10 entries, after the successful update, it would have 11 and trigger the failure. The error message would be passed on into the pattern-matched failure in the case statement.



See how elegant it behaves:

```Elixir
iex(1)> params = %{"username" => "user_1", "name" => ""}
%{"name" => "", "username" => "user_1"}

iex(2)> alias Store.Customer
Store.Customer

iex(3)> Customer.add(params)

19:16:00.727 [debug] QUERY OK db=0.3ms
begin []

19:16:00.765 [debug] QUERY OK db=3.6ms
INSERT INTO "security"."users" ("username","inserted_at","updated_at") VALUES ($1,$2,$3) RETURNING "id" ["user_1", {{2017, 1, 27}, {16, 16, 0, 745833}}, {{2017, 1, 27}, {16, 16, 0, 753016}}]

19:16:00.768 [debug] QUERY OK db=0.4ms
rollback []
{:error, :profile,
 #Ecto.Changeset<action: :insert, changes: %{user_id: 110},
  errors: [name: {"can't be blank", [validation: :required]}],
  data: #Store.ProfileSchema<>, valid?: false>}
```



```Elixir
defmodule Bank.CustomerRegistration do
  use Bank.Model

  def create(username, email, password) do
    Ecto.Multi.new
    |> Ecto.Multi.insert(:customer, Customer.build(%{username: username, email: email}))
    |> Ecto.Multi.run(:account, fn _ ->
      Auth.register(%{email: email, password: password})
    end)
    |> Ecto.Multi.run(:update, fn %{customer: customer, account: account} ->
      Ecto.Changeset.change(customer, auth_account_id: account.id)
      |> Repo.update
    end)
    |> Repo.transaction()
  end
end
```

Before we break this down, let me say this. Ecto.Multi is an all-together-now proposal. In this case, the customer is written, authenticated remotely, and updated, all under a transaction. If I weren’t using this for transactional safety, I could still compose an object over many operations and have that work for me.

But that’s not very useful if we don’t know what’s going on. Let’s break this down. We have a module. Some people (maybe just people I know), call this kind of module a context module. It’s like a Service Object from the OO space and Uncle Bob Martin’s thinking. It produces a context to get something handled simply and directly. This, rather than in models or controllers or some other over-used space in your work.

Inside the create function, we’re creating a new Multi struct with Ecto.Multi.new. The important thing here is we’re creating a data type that stores a local log of every step along the way. This is the all-together-now approach to the code.

Next, we write to the database. We give it a label, :customer in this case. insert takes a changeset or struct. In this example, there’s a really good practice for setting that up, Customer.build:


```Elixir
def build(%{username: username} = params) do
  changeset(%Customer{}, params)
  |> put_assoc(:wallet, Ledger.Account.build_wallet("Wallet: #{username}"))
end
```
This is a nice way to setup a changeset, adding some defaults more simply. A basic Customer changeset with whatever parameters I know about, and setting up the wallet association. A lot of power, and little fuss. Win.

Now we’re on to some custom code with run (twice, actually). We can write any function we’d like. We still label every step, we always label every step. The key is to use the right return value: {:ok, value} or {:error, value}. The more I build with Elixir, the more I love this interface. It makes it easier to manage error handling code with tuples like this.

Finally, we call transaction. This also uses the same tuple return code and the same labeling convention (surprise!). This won’t work for things like MongoDB, but does work for PostgreSQL or similar databases.

So, we have a few functions called in a chain. What’s the big deal? At least two things: it’s all or nothing, and it gives us a log of every step if we need it later. Knowing we’ve got all the steps, even the remote API calls, handled, reduces the complexity of our business logic. And, if it doesn’t go well, we can respond to that by knowing exactly what went wrong and what state we were in.

What does it look like when things go wrong? We get a tuple that looks something like:

{:error, failed_operation, failed_value, changes_so_far}

That is:

- :error, the easiest way to pattern match
- failed_operation: whatever label we gave the step that failed
- failed_value: the failing response
- changes_so_far: all the prior steps, stored and available for reasoning
-
The more-complete instructions for Ecto.Multi are where you’d expect them. You’ll find instructions there for things like update and delete. At work, we use Multi for things like creating a user. In much the same way as the example above, we need an all-or-nothing approach to critical things like this.
