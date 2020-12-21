---
title: "Understanding Phoenix Endpoint"
date: 2018-02-25T13:39:04-04:00
tags: ["elixir", "phoenix"]
---

This small article aims be to a rough introduction to how the Phoenix endpoint works. It assumes intermediate knowledge of Phoenix and Elixir.

---

> The endpoint is the boundary where all requests to your web application start. It is also the interface your application provides to the underlying web servers.
> https://hexdocs.pm/phoenix/Phoenix.Endpoint.html

While navigating through the documentation for Phoenix.Endpoint, the paragraph above caught my attention, it affirmed for me the importance of this module. Given it's central role in the framework, I decided to deep dive into it's implementation details and here's what I learned.


---

## Expansion

Calling `use Phoenix.Endpoint` will prepare the ground by adding at compile time the necessary code to start the endpoint's supervision tree, which has it's entry point in the application module: `supervisor(YourAppWeb.Endpoint, [])`
Once the application starts, the rest of the endpoint's supervision tree will be defined as follows:

1.- `start_link/2` will be called on YourAppWeb.Endpoint

2.- `YourAppWeb.Endpoint.start_link` will call Phoenix.Endpoint.Supervisor.start_link registering the name of the supervisor as YourAppWeb.Endpoint

3.- `Phoenix.Endpoint.Supervisor` will define the following processes under the endpoint's supervision tree: Pub Sub Supervisor, Endpoint Server Supervisor, Endpoint Long Poll Supervisor , Endpoint Watcher Worker and Endpoint Config Worker.

## The Endpoint Server

Phoenix depends directly on Cowboy to handle HTTP requests, at the same time, Cowboy depends on Ranch. Ranch is a socket acceptor pool for TCP protocols, the dependency responsible for handling TCP connections via ranch listeners, which are embedded into the endpoint's supervision tree by taking advantage of Ranch's embedded mode:

> Embedded mode allows you to insert Ranch listeners directly in your supervision tree. This allows for greater fault tolerance control by permitting the shutdown of a listener due to the failure of another part of the application and vice versa.
> https://ninenines.eu/docs/en/ranch/1.2/guide/embedded/

It's the endpoint's server responsibility to embed these listeners into the supervision tree. To do so, the server will:

- Derive which endpoint handler will be used. An endpoint handler is a module that must implement `child_spec/3` as part of the `Phoenix.Endpoint.Handler` behaviour.

- Call `child_spec/3` on the derived handler and retrieve the specification of the child process to be embedded into your application. Such handler can be one of: a custom handler, `Phoenix.Endpoint.CowboyHandler` or `Phoenix.Endpoint.Cowboy2Handler`

### Cowboy Routing

To understand how embedding ranch listeners into your application's supervision tree connects to everything else, understanding Cowboy's routing mechanism is key:

>To make Cowboy useful, you need to map URLs to Erlang modules that will handle the requests. This is called routing.When Cowboy receives a request, it tries to match the requested host and path to the resources given in the dispatch rules. If it matches, then the associated Erlang code will be executed.
>https://ninenines.eu/docs/en/cowboy/1.0/guide/routing/

Routing consists of mapping routes to modules that act as handlers, code in those modules will be executed when a route matches. Routes look the following:

{{<highlight erlang>}}
Routes = [Host]
Host = {HostMatch, PathList} | {HostMatch, Constraints, PathList}
PathList = [Path]
Path = {PathMatch, Handler, Opts} | {PathMatch, Constraints, Handler, Opts}
{{</highlight>}}

Each handler specified in each path defines an `init/3` function which will be called by Cowboy when a request matches the route associated to that handler, passing the defined options as parameters.

### Defining routes

Back to embedded mode, when `child_spec/3` gets called on any of the endpoint handlers, the routes for the application will be defined, compiled and passed as options to the start function of the returned supervisor spec.
The returned child spec has the following structure:

{{<highlight erlang>}}
%{
    id: {:ranch_listener_sup, ref},
    start: {:ranch_listener_sup, :start_link, [
      %% start options are defined here, including routing
    ]}
}
{{</highlight>}}

- `ranch_listener_sup` is the supervisor that gets embedded into your application; once it's started, two additional supervisors are defined as part of its supervision tree:

- `ranch_acceptors_sup`: supervises workers which wait for connections, once a connection arrives, it's passed to the connections supervisor.
ranch_conns_sup : supervises workers which handle connections passed by the acceptors supervisor.

The following image depicts `ranch_listener_sup` supervision tree as described above:

![endpoint-supervision-tree](../../img/endpoint-server-tree.png)
    
    
- `0.276.0` is `ranch_listener_sup`

- `0.277.0` is `ranch_conns_sup`

- `0.278.0` is `ranch_acceptors_sup`  which has 5 children acceptors. Under Phoenix the number of acceptors defaults to 100 unless a custom quantity is specified in the endpoint config.

One example of how routes are defined by `child_spec/3` is the endpoint route path:

{{<highlight erlang>}}
{:_, Plug.Adapters.Cowboy.Handler, {YourApp.Endpoint, []}}
{{</highlight>}}

It complies with the definition of `Path` listed above:

- Any path in the host will be matched via the `_` special path match (Path Match)

- When a route matches, `Plug.Adapters.Cowboy.Handler` will be called (Handler)

- `{YourApp.Endpoint, []}` will be passed as the last parameter to `Plug.Adapters.Cowboy.Handler.init/3` (Opts)

### Plug Pipeline

The handler is the starting point of the plug pipeline; after a ranch acceptor takes a connection, Cowboy transfers the connection details to the handler, which converts this details into a struct, also known as `%Plug.Conn{}`

## Conclusion

The endpoint is undeniably one of the fundamental pieces of any Phoenix application, in this post we have seen at a high level why Cowboy is essential to Phoenix and how they blend together.
