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
