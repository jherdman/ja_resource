# JaResource

[![Build Status](https://travis-ci.org/vt-elixir/ja_resource.svg?branch=master)](https://travis-ci.org/vt-elixir/ja_resource)
[![Hex Version](https://img.shields.io/hexpm/v/ja_resource.svg)](https://hex.pm/packages/ja_resource)

A behaviour to reduce boilerplate code in your JSON-API compliant Phoenix
controllers without sacrificing flexibility.

Exposing a resource becomes as simple as:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource # or add to web/web.ex
  plug JaResource
end
```

JaResource intercepts requests for index, show, create, update, and delete
actions and dispatches them through behaviour callbacks. Most resources need
only customize a few callbacks. It is a webmachine like approach to building
APIs on top of Phoenix.

JaResource is built to work in conjunction with sister library
[JaSerializer](https://github.com/vt-elixir/ja_serializer). JaResource
handles the controller side of things while JaSerializer is focused exclusively
on view logic.

See [Usage](#usage) for more details on customizing and restricting endpoints.

## Rationale

JaResource lets you focus on the data in your APIs, instead of worrying about
response status, rendering validation errors, and inserting changesets. You get
robust patterns and while reducing maintenance overhead.

At Agilion we value moving quickly while developing quality applications. This
library has come out of our experience building many APIs in a variety of
fields.

## Installation

If [available in Hex](https://hex.pm/docs/publish), the package can be installed by:

  1. Adding ja_resource to your list of dependencies in `mix.exs`:

        def deps do
          [{:ja_resource, "~> 0.1.0"}]
        end

  2. Ensuring ja_resource is started before your application:

        def application do
          [applications: [:ja_resource]]
        end

  3. ja_resource can be configured to execute queries on a given repo.  While not required, we encourage doing so to preserve clarity:

        config :ja_resource,
          repo: MyApp.Repo

  4. JaSerializer / JSON-API setup. JaResource is built to work with JaSerializer. Please refer to https://github.com/vt-elixir/ja_serializer#phoenix-usage to setup Plug and Phoenix for JaSerializer and JaResource.


## Usage

For the most simplistic resources JaSerializer lets you replace hundreds of
lines of boilerplate with a simple use and plug statements.

The JaResource plug intercepts requests for standard actions and queries,
filters, create changesets, applies changesets, responds appropriately and
more all for you.

Customizing each action just becomes implementing the callback relevant to
what functionality you want to change.

To expose index, show, update, create, and delete of the `MyApp.Post` model
with no restrictions:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource # Optionally put in web/web.ex
  plug JaResource
end
```

You can optionally prevent JaResource from intercepting actions completely as
needed:

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource
  plug JaResource, except: [:delete]

  # Standard Phoenix Delete
  def delete(conn, params) do
    # Custom delete logic
  end
end
```

And because JaResource is just implementing actions, you can still use plug
filters just like in normal Phoenix controllers, however you will want to
call the JaResource plug last.

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource

  plug MyApp.Authenticate when action in [:create, :update, :delete]
  plug JaResource
end
```

You are also free to define any custom actions in your controller, JaResource
will not interfere with them at all.

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource
  plug JaResource

  def publish(conn, params) do
   # Custom action logic
  end
end
```

### Changing the model exposed

By default JaResource parses the controller name to determine the model exposed
by the controller. `MyApp.UserController` will expose the `MyApp.User` model,
`MyApp.API.V1.CommentController` will expose the `MyApp.Comment` model.

This can easily be overridden by defining the `model/0` callback:

```elixir
defmodule MyApp.V1.PostsController do
  use MyApp.Web, :controller
  use JaResource

  def model, do: MyApp.Models.BlogPost
end
```

### Customizing records returned

Many applications need to expose only subsets of a resource to a given user,
those they have access to or maybe just models that are not soft deleted.
JaResource allows you to define the `records/1` and `record/2`

`records/1` is used by index, show, update, and delete requests to get the base
query of records. Many controllers will override this:

```elixir
defmodule MyApp.V1.MyPostController do
  use MyApp.Web, :controller
  use JaResource

  def model, do: MyApp.Post
  def records(%Plug.Conn{assigns: %{user_id: user_id}}) do
    model
    |> where([p], p.author_id == ^user_id)
  end
end
```

`record/2` receives the `conn` and the id param and returns a
single record for use in show, update, and delete.
The default implementation calls `records/1` with the `conn`, then narrows the query to find only the record with the expected id.
This is less common to customize, but may be useful if using non-id fields in the url:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def record(conn, slug_as_id) do
    conn
    |> records
    |> MyApp.Repo.get_by(slug: slug_as_id)
  end
end
```

### 'Handle' Actions

Every action not excluded defines a default `handle_` variant which receives
pre-processed data and is expected to return an Ecto query or record. All of
the handle calls may also return a conn (including the result of a render
call).

An example of customizing the index and show actions (instead of customizing
`records/1` and `record/2`) would look something like this:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_index(conn, _params) do
    case conn.assigns[:user] do
      nil -> where(Post, [p], p.is_published == true)
      u   -> Post # all posts
    end
  end

  def handle_show(conn, id) do
    Repo.get_by(Post, slug: id)
  end
end
```

### Filtering and Sorting

The handle_index has complimentary callbacks filter/4 and sort/4. These two
callbacks are called once for each value in the related param. The filtering
and sorting is done on the results of your `handle_index/2` callback (which
defaults to the results of your `records/1` callback).

For example, given the following request:

`GET /v1/articles?filter[category]=dogs&filter[favourite-snack]=cheese&sort=-published`

You would implement the following callbacks:

```elixir
defmodule MyApp.ArticleController do
  use MyApp.Web, :controller
  use JaSerializer

  def filter(_conn, query, "category", category) do
    where(query, category: ^category)
  end
  
  def filter(_conn, query, "favourite_snack", snack) do
    where(query, favourite_snack: ^favourite_snack)
  end

  def sort(_conn, query, "published", direction) do
    order_by(query, [{^direction, :inserted_at}])
  end
end
```

Note that in the case of `filter[favourite-snack]` JaResource has already helpfully converted the filter param's name from dasherized to underscore (or from [whatever you configured](https://github.com/vt-elixir/ja_serializer#key-format-for-attribute-relationship-and-query-param) your API to use).

### Paginate

The handle_index_query/2 can be used to apply query params and render_index/3 to serialize meta tag.

For example, given the following request:

`GET /v1/articles?page[number]=1&page[size]=10`

You would implement the following callbacks:

```elixir
defmodule MyApp.ArticleController do
  use MyApp.Web, :controller
  use JaSerializer

  def handle_index_query(%{query_params: params}, query) do
    number = String.to_integer(params["page"]["number"])
    size = String.to_integer(params["page"]["size"])
    total = from(t in subquery(query), select: count("*")) |> repo().one()

    records =
      query
      |> limit(^(number + 1))
      |> offset(^(number * size))
      |> repo().all()

    %{
      page: %{
        number: number,
        size: size
      },
      total: total,
      records: records
    }
  end

  def render_index(conn, paginated, opts) do
    conn
    |> Phoenix.Controller.render(
      :index,
      data: paginated.records,
      opts: opts ++ [
        meta: %{
          page: paginated.page,
          total: paginated.total
        }
      ]
    )
  end
end
```

### Creating and Updating

Like index and show, customizing creating and updating resources can be done
with the `handle_create/2` and `handle_update/3` actions, however if just
customizing what attributes to use, prefer `permitted_attributes/3`.

For example:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def permitted_attributes(conn, attrs, :create) do
    attrs
    |> Map.take(~w(title body type category_id))
    |> Map.merge("author_id", conn.assigns[:current_user])
  end

  def permitted_attributes(_conn, attrs, :update) do
    Map.take(attrs, ~w(title body type category_id))
  end
end
```

Note: The attributes map passed into `permitted_attributes` is a "flattened"
version including the values at `data/attributes`, `data/type` and any
relationship values in `data/relationships/[name]/data/id` as `name_id`.

#### Create

Customizing creation can be done with the `handle_create/2` function.

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_create(conn, attributes) do
    Post.publish_changeset(%Post{}, attributes)
  end
end
```

The attributes argument is the result of the `permitted_attributes` function.

If this function returns a changeset it will be inserted and errors rendered if
required. It may also return a model or validation errors for rendering
or a %Plug.Conn{} for total rendering control.

By default this will call `changeset/2` on the model defined by `model/0`.

#### Update

Customizing update can be done with the `handle_update/3` function.

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def handle_update(conn, post, attributes) do
    current_user_id = conn.assigns[:current_user].id
    case post.author_id do
      ^current_user_id -> {:error, author_id: "you can only edit your own posts"}
      _                -> Post.changeset(post, attributes, :update)
    end
  end
end
```

If this function returns a changeset it will be inserted and errors rendered if
required. It may also return a model or validation errors for rendering
or a %Plug.Conn{} for total rendering control.

The record argument (`post` in the above example) is the record found by the
`record/3` callback. If `record/3` can not find a record it will be nil.

The attributes argument is the result of the `permitted_attributes` function.

By default this will call `changeset/2` on the model defined by `model/0`.

#### Delete

Customizing delete can be done with the `handle_delete/2` function.

```elixir
def handle_delete(conn, post) do
  case conn.assigns[:user] do
    %{is_admin: true} -> super(conn, post)
    _                 -> send_resp(conn, 401, "nope")
  end
end
```

The record argument (`post` in the above example) is the record found by the
`record/2` callback. If `record/2` can not find a record it will be nil.

### Custom responses

It is possible to override the default responses for create and update actions
in both the success and invalid cases.

#### Create

Customizing the create response can be done with the `render_create/2` and
`handle_invalid_create/2` functions. For example:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def render_create(conn, model) do
    conn
    |> Plug.Conn.put_status(:ok)
    |> Phoenix.Controller.render(:show, data: model)
  end

  def handle_invalid_create(conn, errors),
    conn
    |> Plug.Conn.put_status(401)
    |> Phoenix.Controller.render(:errors, data: errors)
  end
end
```

### Update

Customizing the update response can be done with the `render_update/2` and
`handle_invalid_update/2` functions. For example:

```elixir
defmodule MyApp.V1.PostController do
  use MyApp.Web, :controller
  use JaResource

  def render_update(conn, model) do
    conn
    |> Plug.Conn.put_status(:created)
    |> Phoenix.Controller.render(:show, data: model)
  end

  def handle_invalid_update(conn, errors) do
    conn
    |> Plug.Conn.put_status(401)
    |> Phoenix.Controller.render(:errors, data: errors)
  end
end
```
