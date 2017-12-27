---
layout: post
title: Implementing a peer-to-peer network in Elixir - The Server
categories: code elixir
published: false
---

In the past week or so I've been implementing a peer-to-peer network using Elixir
as part of a project I'm working on. I thought about writing this post series because I haven't found much resources addressing such cases online and second to document the whole process for future reference.

Even if you don't want to implement a peer-to-peer network, this is still a great exercise
to learn more and experimenting OTP applications and concepts.

We'll be implementing a peer-to-peer network that allows peers to send a text message to other peers and
receive the same message back. Simple enough, right?

This series is splitted in three parts:

0. [Implementing a peer-to-peer network in Elixir - The Server]({{ page.url | relative }}) (current)
0. ~~Implementing a peer-to-peer network in Elixir - The Client~~
0. ~~Implementing a peer-to-peer network in Elixir - Enhancements~~

In this post we cover our peer-to-peer network server-side logic. In the end, you will have a working
TCP server that listens and accepts connections and echoes back every message it receives.

When we're done, you'll be able to connect and test it using Telnet.

## Create the project

Use [Mix]() to create a new project with the `--sup` flag, to generate an OTP application skeleton
that includes a supervision tree and the `application` callback setup. For the sake of simplicity,
I'll name this project `network`:

```bash
$ mix new network --sup
```

## Setup dependencies

For this project, the only dependency that we will need is Ranch[^ranch]. Update **mix.exs** to include it:

```elixir
defp deps do
  [
    {:ranch, "~> 1.4"}
  ]
end
```

When done, fetch the dependency:

```bash
$ mix deps.get
```

## Listening for connections

To have have someone connecting to our server, we have to be listening for and accepting them as
they arrive. This is where the Ranch[^ranch] library comes handy.

Create **lib/network/server.ex**:

```elixir
defmodule Network.Server do
  @moduledoc """
  A simple TCP server.
  """
  
  use GenServer

  alias Network.Handler

  require Logger

  @doc """
  Starts the server.
  """
  def start_link(args) do
    GenServer.start_link(__MODULE__, args, name: __MODULE__)
  end

  @doc """
  Initiates the listener (pool of acceptors).
  """
  def init(port: port) do
    opts = [{:port, port}]

    {:ok, pid} = :ranch.start_listener(:network, :ranch_tcp, opts, Handler, [])

    Logger.info(fn ->
      "Listening for connections on port #{port}"
    end)

    {:ok, pid}
  end
end
```

On the `init/1` function is where the magic happens. We're using the `:ranch.start_listener/5` function
to create a pool of acceptor processes that will accept incoming connections and, when it does, spawn a new process to handle it with the specified protocol (`Network.Handler`).

The five arguments the `:ranch.start_listener/5` requires are:

0. `:network` --- unique name that identifies the listener
0. `:ranch_tcp` --- the transport[^transport]
0. `[{:port, port}]` --- transport's options
0. `Network.Handler` --- protocol handler
0. `[]` --- handler's options

## Handling connections

Because Ranch[^ranch] makes us abstract the protocol handling into it's own module --- which is very
useful because of the fact that it minimizes code complexity --- that's what we'll do now.

Create **lib/network/handler.ex**:

```elixir
defmodule Network.Handler do
  @moduledoc """
  A simple TCP protocol handler that echoes all messages received.
  """

  use GenServer

  require Logger

  # Client

  @doc """
  Starts the handler with `:proc_lib.spawn_link/3`.
  """
  def start_link(ref, socket, transport, _opts) do
    pid = :proc_lib.spawn_link(__MODULE__, :init, [ref, socket, transport])
    {:ok, pid}
  end

  @doc """
  Initiates the handler, acknowledging the connection was accepted.
  Finally it makes the existing process into a `:gen_server` process and
  enters the `:gen_server` receive loop with `:gen_server.enter_loop/3`.
  """
  def init(ref, socket, transport) do
    peername = stringify_peername(socket)

    Logger.info(fn ->
      "Peer #{peername} connecting"
    end)

    :ok = :ranch.accept_ack(ref)
    :ok = transport.setopts(socket, [{:active, true}])

    :gen_server.enter_loop(__MODULE__, [], %{
      socket: socket,
      transport: transport,
      peername: peername
    })
  end

  # Server callbacks

  def handle_info(
        {:tcp, _, message},
        %{socket: socket, transport: transport, peername: peername} = state
      ) do
    Logger.info(fn ->
      "Received new message from peer #{peername}: #{inspect(message)}. Echoing it back"
    end)

    # Sends the message back
    transport.send(socket, message)

    {:noreply, state}
  end

  def handle_info({:tcp_closed, _}, %{peername: peername} = state) do
    Logger.info(fn ->
      "Peer #{peername} disconnected"
    end)

    {:stop, :normal, state}
  end

  def handle_info({:tcp_error, _, reason}, %{peername: peername} = state) do
    Logger.info(fn ->
      "Error with peer #{peername}: #{inspect(reason)}"
    end)

    {:stop, :normal, state}
  end

  # Helpers

  defp stringify_peername(socket) do
    {:ok, {addr, port}} = :inet.peername(socket)

    address =
      addr
      |> :inet_parse.ntoa()
      |> to_string()

    "#{address}:#{port}"
  end
end
```

There are some particularities about this module that are very interesting. First, you may've noticed we've implemented the `GenServer` behaviour because of the functions and callbacks defined, although we don't use the `GenServer.start_link/3` function and instead use `:proc_lib.spawn_link/3`.

Before moving into more details on that, let's see the `init/3` function. It's all clear at first sight: we acknowledge the connection with `:ranch.accept_ack/1`, set the connection to be active and then... we enter a loop?

Sure! We need to be in a loop waiting for new messages to arrive from the connection and upon receiving a message we do whatever processing it requires, entering the loop and waiting for new messages again.

As we implement the `GenServer` behaviour we must use `:gen_server.enter_loop/3` which turns our process into a `:gen_server` process and enters the `:gen_server` process receive loop.

Now going back, why `:proc_lib.spawn_link/3`? If you are aware of the `GenServer` behaviour you know that you must define a `start/3` or `start_link/3` function to start the server and that once it has started it will call the `init` callback. So far so good.

The issue happens because of the way that behaviour works. According to the [`GenServer:start_link/3` documentation](https://hexdocs.pm/elixir/GenServer.html#start_link/3):

> To ensure a synchronized start-up procedure, this function does not return until `c:init/1` has returned.

That would raise a big issue when we need to enter a loop, because when you enter the loop it will never return until something bad happen and an error is returned. Thus why we are using `:proc.spawn_link/3`, because instead of spawning the process synchronously it will spawn it asynchronously and we won't have any issues.

Actually, the only processes that can use `:gen_server.enter_loop/3` are those started with this particular function.

## Starting the server

Update **config/config.exs** to include the server configuration:

```elixir
use Mix.Config

config :network, :server,
  port: String.to_integer(System.get_env("PORT") || "5555")
```

Update **lib/network/application.ex** to include `Network.Server` in the application's supervision tree:

```elixir
defmodule Network.Application do
  @moduledoc false
  
  use Application
  
  def start(_type, _args) do
    # Get configuration
    config = Application.get_env(:network, :server)

    children = [
      # Add it to supervison tree
      {Network.Server, config}
    ]
  
    opts = [strategy: :one_for_one, name: Network.Supervisor]
    Supervisor.start_link(children, opts)
  end
end
```

## Testing

Open a terminal and start the application:

```bash
$ mix run --no-halt
00:00:00.000 [info] Accepting connections on port 5555
```

Open another terminal and connect using `telnet`:

```bash
$ telnet 127.0.0.1 5555
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^['.
```

Nice, we've successfully connected to the server. On the terminal our application is
running we should also see a message informing us just that:

```bash
00:00:00.000 [info] Peer 127.0.0.1:00000 connecting
```

Now try to send any message through the telnet session, and it will be echoed back. On
the application's terminal you'll see (for example):

```bash
00:00:00.000 [info] Received new message from 127.0.0.1:00000: "Hello, opencode.space!\r\n". Echoing it back
```

And if you close the terminal running `telnet` our application also gets notified:

```bash
00:00:00.000 [info] Peer 127.0.0.1:00000 disconnected
```

## Conclusion

In only `~153 LOC` we have successfully implemented a TCP server that echoes every message it receives. Pretty neat, isn't it?

On the next part of this series we will be covering how to implement a client to connect to the server in order to achieve a peer-to-peer network.

Stay tunned for updates!

## Notes

[^ranch]: [Ranch]() is a socket acceptor pool for TCP protocols. For more information visit the library's [User Guide]() or [Function Reference]()

[^transport]: TODO
