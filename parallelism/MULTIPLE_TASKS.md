Multiple Tasks
==============



```Elixir
defmodule Parallel do
  def pmap(collection, func) do
    collection
    |> Enum.map(&(Task.async(fn -> func.(&1) end)))
    |> Enum.map(&Task.await/1)
  end
end
```

We can run this function to get the squares of numbers from 1 to 10000:


```Elixir
iex(1)> c("parallel.ex")
iex(2)> res = Parallel.pmap 1..10000, &(&1 * &1)
[1, 4, 9, 16, 25, 36, 49, 64, 81, 100, 121, 144, 169, 196, 225, 256, 289, 324,
 361, 400, 441, 484, 529, 576, 625, 676, 729, 784, 841, 900, 961, 1024, 1089,
 1156, 1225, 1296, 1369, 1444, 1521, 1600, 1681, 1764, 1849, 1936, 2025, 2116,
 2209, 2304, 2401, 2500, ...]
 
