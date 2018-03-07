

Note that the type name is also a form to document and explain what is your data
structure. number_with_remark is a tuple with the result and a comentary, used
several times by different functions. 


```Elixir
defmodule LousyCalculator do
  @typedoc """
  Just a number followed by a string.
  """
  @type number_with_remark :: {number, String.t}

  @spec add(number, number) :: number_with_remark
  def add(x, y), do: {x + y, "You need a calculator to do that?"}

  @spec multiply(number, number) :: number_with_remark
  def multiply(x, y), do: {x * y, "It is like addition on steroids."}
end
```

In the example below, you can see several strucs as types, and also the |
operator, that will help to describe alternative outputs


```Elixir
defmodule Cloud.Pubsub do

  use Task
  @moduledoc """
  Subscriber for GoogleApi.PubSub. When pulling messages, you will receive a list of 
  GoogleApi.PubSub.V1.Model.ReceivedMessage struct. To get the payload, extract the received message
  from the list, and follow the path, e.g. m.meessage.data

    %GoogleApi.PubSub.V1.Model.ReceivedMessage{
      ackId: "RUFeQBJMNgRESVMrQwsqWBFOBCEhPjA-RVNEUAYWLF1GS",
      message: %GoogleApi.PubSub.V1.Model.PubsubMessage{
        attributes: nil,
        data: "cmVhbCBsb3ZlIGlzIGZvciB0aGUgYmVuZWZpdCBvZiBvdGhlcnM=",
        messageId: "9675723158005",
        publishTime: "2018-03-07T11:02:18.911Z"
      }
    }
  """

  @type project_id            :: binary
  @type subscription_name     :: binary
  @type topic_name            :: binary
  @type payload               :: binary
  @type ack_response          :: atom
  @type publish_response      :: GoogleApi.PubSub.V1.Model.PublishResponse
  @type message               :: GoogleApi.PubSub.V1.Model.ReceivedMessage
  @type messages              :: [GoogleApi.PubSub.V1.Model.ReceivedMessage]
  @type max_messages          :: integer
  #@type on_start :: {:ok, pid} | :ignore | {:error, {:already_started, pid} | term}


  # def init(state), do: {:ok, state}

  @doc "task that will listen the pubsub"
  @spec start_link(project_id, subscription_name) :: {:ok, pid}
  def   start_link(project_id, subscription_name) do
    task = Task.async(GoogleApi.PubSub.Samples.Subscriber, :listen, [project_id, subscription_name])
    {:ok, task.pid}
  end

  @doc """
  Pull a list of mesages from the Pubsub queue. Be sure to ack all of them after pulling during
  the acknoledgement deadline, and if not, you have to pull again. 
  If there are NO messages on the queue, it will lock and wait *20 seconds* and answer nil
  ## Examples
  iex> Cloud.Pubsub.pull("my-project", "my-subscription", 5)

  TODO: add default maxMessages, and optional
        add response and error
  """
  @spec pull(project_id, subscription_name, max_messages) :: messages | nil
  def   pull(project_id, subscription_name, max_messages) do
    conn = Cloud.Connection.get()

    # Make a subscription pull
    {:ok, response} = GoogleApi.PubSub.V1.Api.Projects.pubsub_projects_subscriptions_pull(
      conn,
      project_id,
      subscription_name,
      [body: %GoogleApi.PubSub.V1.Model.PullRequest{
        maxMessages: max_messages
      }]
    )
    response.receivedMessages
  end

```


