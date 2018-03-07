https://swanros.com/2017/03/04/action-fallback-contexts-phoenix1-3-tiny-controllers/



Registers the plug to call as a fallback to the controller action.

A fallback plug is useful to translate common domain data structures into a valid %Plug.Conn{} response. If the controller action fails to return a %Plug.Conn{}, the provided plug will be called and receive the controller’s %Plug.Conn{} as it was before the action was invoked along with the value returned from the controller action.



```Elixir
defmodule MyController do
  use Phoenix.Controller

  action_fallback MyFallbackController

  def show(conn, %{"id" => id}, current_user) do
    with {:ok, post} <- Blog.fetch_post(id),
         :ok <- Authorizer.authorize(current_user, :view, post) do

      render(conn, "show.json", post: post)
    end
  end
end
```

In the above example, with is used to match only a successful post fetch, followed by valid authorization for the current user. In the event either of those fail to match, *with* will not invoke the render block and instead return the unmatched value. In this case, imagine Blog.fetch_post/2 returned {:error, :not_found} or Authorizer.authorize/3 returned {:error, :unauthorized}. For cases where these datastructures serve as return values across multiple boundaries in our domain, a single fallback module can be used to translate the value into a valid response. For example, you could write the following fallback controller to handle the above values:


```Elixir
defmodule MyFallbackController do
  use Phoenix.Controller

  def call(conn, {:error, :not_found}) do
    conn
    |> put_status(:not_found)
    |> render(MyErrorView, :"404")
  end

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(403)
    |> render(MyErrorView, :"403")
  end
end
```
