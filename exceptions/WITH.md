With
====



```Elixir
opts = %{width: 10}
with {:ok, width} <- Map.fetch(opts, :width),
     {:ok, height} <- Map.fetch(opts, :height) do
  {:ok, width * height}
else
  :error ->
    {:error, :wrong_data}
end
```


We are pattern matching {:ok, width} with Map.fetch(opts, :width). This uses a backwards arrow operator which is a little strange. There’s probably a good explanation about why not pattern match on an =. Whatever the reason, it helps me think about each step of my work because it’s different. 

Put another way, that first line fetches :width from the opts map. It expects a tuple starting with :ok, and whatever width is in the map (10 in our case). If that doesn’t match, the else code is called. More on that in a moment. 

Notice a comma at the end of the first pattern match/operation. We then go to the next operation, and keep going as long as we’d like, pattern matching and collecting values as we go. These values are available to me in subsequent calls or my do block. Did you notice that block? In this case, it returns {:ok, width * height}. If all went well, we get the area of a square. If not, the else code is written. 

Now, the else statement works like a case. Something didn’t work, let’s pattern match what that was. In this case, we expect :error only, so we use :error -> {:error, :wrong_data}. If things could go wrong inconsistently, you could write many pattern matching entries here. The syntax for the else is like a case, meaning no commas and forward arrows. 


with recapped:

- start the with operations on the same line as with
- use backward arrows for the operations
- use commas to separate operations
- pattern match the happy path from each operation
- create a do block to handle what happens when everything goes write
- create an else block to handle what happens when something goes wrong
- leave the else block out if you can handle the error response directly
- use case syntax for the else block (forward arrows (->) and no commas)

    This is a tricky little special form, but I tend to use it quite a bit. Why? I can have the all-or-nothing benefits I got from Ecto.Multi, but for any code. Also, I have more-relaxed rules about return values. So, if I don’t take the time to wrap everything I touch to use the {:ok, value} or {:error, value} response, I can still manage the unhappy path.


```Elixir
with {:ok, result1} <- square(a),
     {:ok, result2} <- half(result1), 
     {:ok, result3} <- triple(result2),
     {:ok, result4} <- do_even_more_stuff(result3)
do
  IO.puts “Success! the result is #{result4}”
else 
  {:error, error} -> IO.puts “Oops, something went wrong: #{error}”
end
```





```Elixir
defmodule User do
  defstruct name: nil, dob: nil

  def create(params) do
  end

end

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




Every statement of with is executed in order. Execution continues as long as left (arrow) right match. As soon as a match fails, the else block is executed. Within the else block we can match against whatever WAS returned. If all statements match, the do block is executed and has access to all the local variables in the with block.

Got it? Here's a test. Why did we have to change our success cases so that they'd return {:ok, dob} and {:ok, name} instead of just dob and name?

If we didn't return {:ok, X}, our match would look like:



```Elixir
def create(params) do
    with dob <- parse_dob(params["dob"]),
         name <- parse_name(params["name"])  # <<--- match here, even if had an error...
    do
      %User{dob: dob, name: name}
    else
      err -> err
    end
  end
```


However, a variable on the left side matches everything. In other words, the above would treat the {:error, "BLAH"} as matches and continue down the "success" path. 
So this is the reason you MUST use {:ok, result} and {:error, error} in "with"
pipes
