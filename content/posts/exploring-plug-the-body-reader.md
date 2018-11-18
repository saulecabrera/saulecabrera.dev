---
title: "Exploring Plug: The Body Reader"
date: 2018-11-14T18:13:34-05:00
tags:
  - elixir
  - plug
draft: false
---

Ever wondered why sometimes `Plug.Conn.read_body/2` returns `nil`
instead of returning the raw body of the request? Today we'll examine why
this happens and how to elegantly get around it.

## Plug crash course

Before exploring why this happens, let's take a step back and take a naive look on how Plug works.

In it's most basic sense, a plug is a transformation applied to a connection; a plug can be either 
a function or a module that receives information about a request in the form of
`%Plug.Conn{}` and returns a `%Plug.Conn{}`. A connection goes through many of these transformations
throughout it's lifecycle, or what we call a plug pipeline.


## By design

The body can only be read once throughout a connection's lifecycle, this is a [design decision](https://github.com/elixir-plug/plug/issues/691). Caching the body may lead to memory bloat, specially if dealing with large requests.

This means that if a plug reads the body at any given point in time, _every_ subsequent plug that
tries to acess it, will get `nil` as a result instead.

Even tough there are some performance reasons to avoid caching the body, sometimes you need to access
it in some of your endpoints to perform validations on the contents of the request. For this to work,
we need to:

  - Identify at which stage is the body read
  - Instruct plug to cache the body at that stage

These are high level instructions, next we'll explore a practical example.


## A practical example

In plug based projects, Plug provides a behavior that specifies the guidelines on how to parse the request's body, all parsers must comply with this behavior. At this point we assume that this plug reads the body and therefore no subsequent reads will be valid. 

To instruct plug that we want to cache the body before parsing the body, we can pass in the `body_reader` option to the `Parsers` plug:

{{<highlight elixir>}}
  plug Plug.Parsers,
      parsers: [:urlencoded, :multipart, :json],
      pass: ["*/*"],
      body_reader: {BodyReader, :cache, []},
      json_decoder: Poison
{{</highlight>}}


The `body_reader` option accepts a tuple containing a module, a function and arguments, which will get invoked before the body is
parsed and discarded. In the example above we're passing `{BodyReader, :cache, []}`, it's implementation looks like the following:

{{<highlight elixir>}}
  defmodule BodyReader do
    def cache(conn, opts)do
      {:ok, body, conn} = Plug.Conn.read_body(conn, opts)
      conn = Plug.Conn.assign(conn, :raw_body, body)
      {:ok, body, conn}
    end
  end
{{</highlight>}}

Inside `BodyReader.cache/2` we are storing the body into the connection, with `:raw_body` as key;
later on, to access it, we can do so by calling `conn.assigns[:raw_body]`.

