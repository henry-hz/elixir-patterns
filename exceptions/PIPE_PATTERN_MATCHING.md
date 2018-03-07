Pipes + Pattern Matching
========================

Two ways for achieving the same result. The first is to use the pipe |> and for each function catch the error and propagate it down. The error has to be always returned with the same pattern, like {:error, reason} 


```elixir
connect
  |> receive_image
  |> resize
  |> rotate
  |> save

```

If *receive_image* returns an error, resize need to handle it, so we use pattern matching, and to propagate it down through the pipe. So rotate needs to do the same, etc..



```elixir
def resize({:error, _}=error), do:error
```

So if you need to write a sequence of function calls where some or all are expected to return an error, `with` would be the most appropriate tool to use.

If some or all functions are expected to return an error, see the
[with](WITH.md) pattern.






```Elixir
defmodule Test do

  @doc """
  Inserts the given JSON into the `data` field of `file`.
  """
  def insert(file, data) do
    # Read the contents of the file
    file
    |> File.read()
    |> rewrite(file, data)
    |> to_user_message()
    |> IO.puts
  end

  defp rewrite({:ok, content}, file, data) do
    content
    |> Poison.decode()
    |> write_decoded_content(file, data)
  end
  defp rewrite({:error, reason}, _, _) do
    {:error, {:on_read, reason}}
  end

  defp write_decoded_content({:ok, decoded_content}, file, data) do
    data
    |> Poison.encode()
    |> write_encoded_data(file, decoded_content)
  end
  defp write_decoded_content({:error, :invalid}, _, _) do
    {:error, {:on_decode_invalid, ""}}
  end
  defp write_decoded_content({:error, :invalid, reason}, _, _) do
    {:error, {:on_decode, reason}}
  end

  defp write_encoded_data({:ok, encoded_data}, file, decoded_content) do
    # Prepare the updated data for insertion
    # and encode the updated data
    decoded_content
    |> Map.update("data", encoded_data, fn _ -> encoded_data end)
    |> Poison.encode()
    |> write_encoded_final_data(file)
  end
  defp write_encoded_data({:error, {:invalid, reason}}, _, _) do
    {:error, {:on_encode_data, reason}}
  end

  defp write_encoded_final_data({:ok, encoded_final_data}, file) do
    file
    |> File.write(encoded_final_data)
    |> report_on_file_write()
  end
  defp write_encoded_final_data({:error, {:invalid, reason}}, _) do
    {:error, {:on_encode_final_data, reason}}
  end

  defp report_on_file_write(:ok),
    do: :ok
  defp report_on_file_write({:error, reason}),
    do: {:error, {:on_file_write, reason}}

  defp to_user_message(:ok),
    do: "Successfully updated file!"
  defp to_user_message({:error, {source, reason}}),
    do: to_error_msg source, reason

  defp to_error_message(source, reason) do
    case source do
      :on_read ->
        "Could not read file because #{reason}!"
      :on_decode_invalid ->
        "Could not parse file to JSON!"
      :on_decode ->
        "Could not parse file to JSON because #{reason}!"
      :on_encode_data ->
        "Could not encode given JSON because #{reason}!"
      :on_encode_final_data ->
        "Could not encode updated JSON because #{reason}!"
      :on_file_write ->
        "Could not write to file because #{reason}!"
    end
  end
end
```

In implementations with imperative languages I often encounter a certain unwillingness to acknowledge the complexity burden that detailed error handling imposes which typically devolves in rampaging [Arrow head](http://wiki.c2.com/?ArrowAntiPattern)  (especially in JavaScript) and insufficient separation/segregation of “happy path” from “unhappy path” code. “Unhappy path” code deserves the same “management” consideration as the “happy path” code that the domain logic is primarily concerned with.


Nothing worst than:

```JavaScript
if get_resource
   if get_resource
     if get_resource
       if get_resource
         do something
         free_resource
       else
         error_path
       endif
       free_resource
     else
       error_path
     endif
     free_resource
   else
     error_path
   endif
   free_resource 
 else
   error_path
 endif
```


An example of a pipe that send emails and retry:

```Elixir
defp send_email(request, email, times) do
    request
    |> ExAws.request()
    |> maybe_retry(request, email, times)
  end

  defp maybe_retry({:error, {:http_error, 454, _body}} = error, request, email, times) do
    if times > @backoff_times do
      Logger.warn("AWS SES throttled ##{times}")
      raise "failed to send email\n\n#{inspect(email)}\n\n#{inspect(error)}"
    else
      Process.sleep(@backoff * trunc(:math.pow(2, times)))
      send_email(request, email, times + 1)
    end
  end

  defp maybe_retry({:error, _} = error, _request, email, _times) do
    raise "failed to send email\n\n#{inspect(email)}\n\n#{inspect(error)}"
  end

  defp maybe_retry({:ok, result}, _request, _email, _times) do
    result
  end

```

Note that the first argument of maybe_retry is pattern matching the possible
outputs of ExAws.request()


See the example below, as a good candidate to be written using the "with" pipe:


```Elixir
defmodule User do
  defstruct name: nil, dob: nil

  def create(params) do
  end

end
```

The create function should either create a User struct or return an error if the params are invalid. How would you go about doing this? Here's an example using pipes:




```Elixir
def create(params) do
    %User{}
      |> parse_dob(params["dob"])
      |> parse_name(params["name"])
  end

  defp parse_dob(user, nil), do: {:error, "dob is required"}
  defp parse_dob(user, dob) when is_integer(dob), do: %{user | dob: dob}
  defp parse_dob(_user, _invalid), do: {:error "dob must be an integer"}

  defp parse_name(_user, {:error, _} = err), do: err
  defp parse_name(user, nil), do: {:error, "name is required"}
  defp parse_name(user, ""), do: parse_name(user, nil)
  defp parse_name(user, name), do: %{user | name: name}
```

The problem with this approach is that every function in the chain needs to handle the case where any function before it returned an error. It's clumsy, both because it isn't pretty and because it isn't flexible. Any new return type that we introduced has to be handled by all functions in the chain.

The pipe operator is great when all functions are acting on a consistent piece of data. It falls apart when we introduce variability. That's where with comes in. with is a lot like a |> except that it allows you to match each intermediary result.



I am copying here the "with" version of the pipe above, even that you can also
find in the [with](WITH.md) section with detailed explanations


```Elixir
def create(params) do
    with {:ok, dob} <- parse_dob(params["dob"]),
         {:ok, name} <- parse_name(params["name"])
    do
      %User{dob: dob, name: name}
    else
      # nil -> {:error, ...} an example that we can match here too
      err -> err
    end
  end

  defp parse_dob(nil), do: {:error, "dob is required"}
  defp parse_dob(dob) when is_integer(dob), do: {:ok, dob}
  defp parse_dob(_invalid), do: {:error "dob must be an integer"}

  defp parse_name(nil), do: {:error, "name is required"}
  defp parse_name(""), do: parse_name(nil)
  defp parse_name(name), do: {:ok, name}
```

