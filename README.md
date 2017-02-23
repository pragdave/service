# Service

Erlang encourages us to write our code as self-contained servers and
applications. Elixir makes it even easier by removing much of the
boilerplate needed to create an Erlang GenServer.

However, creating these servers is often more work than it needs to
be. And, unfortunately, following good design practices adds even more
work, in the form of duplication.

The Service library aims to make it easier for newcomers to craft
well designed services. It doesn't replace GenServer. It is simply a
layer on top that handles the most common GenServer use cases. The
intent is to remove any excuse that people might have for not writing
their Elixir code using a ridiculously large number of trivial
services.

# Basic Example

You can think of an Erlang process as a remarkably pure implementation
of an object. It is self contained, with private state, and with an
interface that is accessed by sending messages. This harks straight
back to the early days of Smalltalk.

Diet.Service draws on that idea as a notation. Here's a simple service
that implements a key-value store.

~~~ elixir
defmodule KVStore do

  use Service.Anonymous, state: %{}

  def put(store, key, value) do
    set_state(Map.put(store, key, value)) do
      value
    end
  end
  
  def get(store, key) do
    store[key]
  end

end
~~~

The first parameter to `put` and `get` is the current state, and the second is
the value being passed in.

You'd call it using

~~~ elixir
rt = KVStore.run()

KVStore.put(rt, :name, "Elixir")
KVStore.get(rt, :name)   # => "Elixir"
~~~

Behind the scenes, Service has created a pure implementation of our
totaller, along with a GenServer that delegates to that
implementation. What does that code look like? Add the `:show_code`
option to our original source.

~~~ elixir
defmodule KVStore do

  use Service.Anonymous, 
      state: %{},
      show_code: true

  def put(store, key, value) do
    # . . .
~~~

During compilation, you'll see the code that will actually be run:


~~~ elixir

# defmodule RunningTotal do

  import(Kernel, except: [def: 2])
  import(Service.Common, only: [def: 2, set_state: 2])
  @before_compile({Service.Anonymous, :generate_code_callback})
  def run() do
    run(%{})
  end
  def run(state) do
    {:ok, pid} = GenServer.start_link(__MODULE__, state, server_opts())
    pid
  end
  def init(state) do
    {:ok, state}
  end
  def initial_state(default_state, _your_state) do
    default_state
  end
  def server_opts() do
    []
  end
  defoverridable(initial_state: 2)

  use(GenServer)

  def put(store, key, value) do
    GenServer.call(store, {:put, key, value})
  end
  def get(store, key) do
    GenServer.call(store, {:get, key})
  end
  def handle_call({:put, key, value}, _, store) do
    __MODULE__.Implementation.put(store, key, value)
    |> Service.Common.create_genserver_response(store)                                                                  
  end
  def handle_call({:get, key}, _, store) do
    __MODULE__.Implementation.get(store, key)
    |> Service.Common.create_genserver_response(store)                                                                         
  end
  
  defmodule Implementation do
    def put(store, key, value) do
      set_state(Map.put(store, key, value)) do
        value
      end
    end
    def get(store, key) do
      store[key]
    end
  end

# end
~~~

### Testing

We can test this implementation without starting a separate
process by simply calling functions in `KVStore.Implementation`. You
simply have to supply the state, and allow for the fact that the responses
will include the updated state if `set_state` is called.

~~~ elixir 
alias KVI KVStore.Implementation

@state %{}

test "we can put a KV pair, then get it, retrieves the correct value" do
  { :reply, Elixir, state } = KVI.put(@state, :name, Elixir)
  assert KVI.get(state, name) == Elixir
end
~~~

## Creating a Named (Singleton) Service

It's sometimes convenient to create a global, named, service. Logging
is a good example of this, as are registry services, global caches, and
the like. 

We can make out KV store a global _named service_ with some trivial changes:

~~~ elixir
defmodule NamedKVStore do

  use Service.Named, 
      state_name:   :kvs, 
      state:        %{}, 

  def put(key, value) do
    set_state(Map.put(kvs, key, value)) do
      value
    end
  end
  
  def get(key) do
    kvs[key]
  end

end
~~~

Notice there's a big of magic here. A named service can be called from
anywhere in your code. It doesn't require you to remember a PID or any
other handle, as the service's API invokes the service process by
name. However, the service process itself contains state (the map in
the KVStore example). The client doesn't need to know about this
internal state, so it is never exposed via the API. Instead, it is
automatically made available inside the service's functions in a
variable. By default, this variable is called `state`, but the
NamedKVStore example changes this to something more meaningful, `kvs`.



You'd call the named KV store service using

~~~ elixir
NamedKVStore.put(:name, "Elixir")
NamedKVStore.put(:engine, "BEAM")  
NamedKVStore.get(:name)            # => "Elixir"
~~~

## Pooled Services

Named services can be turned into pools of workers
by changing to `Service.Pooled`.

~~~ elixir
defmodule TwitterFeed do

  use Service.Pooled,
      pool: [ min: 5, max: 20 ]

  def fetch(name) do
    # ...
  end
end
~~~

Calls to `TwitterFeed.fetch` would run in parallel, up to a maximum of
20.


## Inline code

Finally, we can tell Service not to generate a server at all.

~~~ elixir
defmodule RunningTotal do

  use Service.Inline, state: 0

  def add(total, value) do
    set_state(value+total)
  end
end
~~~

In this case, `set_state(value+total)` becomes just `value+total`.

The cool thing is we can switch between not running a process, running
a single server, or running a pool of servers by changing a single
declaration in the module.


# Types of Services

## Named vs. Anonymous

There's a fundamental difference between named and anonymous servers.
The difference isn't one of implementation—the two forms are basically
the same. It's a difference of intent.

A named server is effectively a singleton—a global object. If it has
state, that state is only meaningful globally—it is not the property
of any one client process. Loggers, time services, and so on are all
examples of named, singleton servers.

Anonymous services have a different intent. Because they are accessed
via a handle (a reference or) pid, they have an owner—the process that
has that handle. Ownership can be shared, but that sharing is
controlled by the original creator of the server.

The state of anonymous servers can therefore be specific to a
particular owner. If we use a server to have a conversation with a
remote service, for example, we own that server, and it can maintain
the state of the connection between requests.

## Solo vs Pooled

A solo service is a single process. When a client makes a request,
that process is executed. No other client can run code in that service
until it is idle again.

A pooled service contains zero or more workers. Each worker is
equivalent to a solo service, in that it handles work requests for one
client at a time. However, a pooled service also contains a pool
manager (or scheduler). Requests for the pool service are not sent to
the worker directly. Instead, they are sent to the scheduler. If it
has a free worker process, it forwards on the request. If it does not,
it delays execution of the caller until a worker becomes free (or a
timeout occurs).

A pooled service can be configured to support some maximum number of
workers. It will dynamically create new workers to handle requests if
all existing workers are busy _and_ if the worker count is less than
the maximum.

Pooled services are useful when you want to achieve parallelism and
the work performed by the service is likely to run for a nontrivial
time. They are also useful as a way to hold on to expensive resources
(such as database connections).

# Coding Model

Permuting named and anonymous services with solo and pooled
provisioning gives us four categories of service.

From a coding standpoint, though, the most significant difference is
between named and anonymous services (whether they are solo or
pooled).

A named service has its own state. That state is not passed to it on
each request.

An anonymous service (logically) does not have a single state.
Instead, the state is established for it each time it is called
(typically by passing it a server pid).

This means that our APIs will be different. For a named service, such
as a logger, we might write:

~~~ elixir
Logger.info("Starting")
~~~
    
For an anonymous service, such as a database connection, we need to
create it to get a handle, and then pass that handle to it on each
request.

~~~ elixir
{ :ok, handle } = DBConnection.start_link(«params»)
# ...
DB.insert(handle, «stuff»)
~~~

Because of this, Diet.Service comes in two flavors, one for named
services and one for anonymous ones.

## Defining a Named Service

~~~ elixir
defmodule MyLogger do

  defstruct device: :stdio

  use Diet.Service.Named,
      state: [ logger: %MyLogger{} ]

  def info(msg) do
    IO.puts(logger.device, "-- #{msg}")
  end
  
  def set_device(new_device) do
    set_state(%{ logger | device: new_device}) do
      :ok
    end
  end

end
~~~

By default, `MyLogger` will be spawned when the application starts.
The service will (by default) have the same name as the module
(`Elixir.MyLogger` in this case).

Its state is defined by the clause:

~~~ elixir
state: [ logger: %MyLogger{} ]
~~~

Inside the module's public functions, the state will be made available
via the variable `logger`.[^fn-magic] The initial value of the state will be the
struct defined in this same module.

[^fn-magic]: Yes, this is magic, and it's probably frowned on by José. However, doing
    this makes the API into the module consistent (but that's a
    discussion that doesn't fit in a footnote). It's really no
    different to the implicit `this` variable in OO code.

## Defining an Anonymous Service

~~~ elixir
defmodule KVStore do

  use Diet.Service.Anonymous
      state: %{}

  def put(store, key, value) do
    set_state(Map.put(store, key, value), do: value
  end
  
  def get(store, key) do
    store[key]
  end

end
~~~

Here's how you'd use this:

~~~ elixir
handle = KVStore.run()
KVStore.put(handle, :name, "José")
IO.puts KVStore.get(handle, :name)
~~~

What if you wanted to pass some initial values into the store? There
are a couple of techniques. First, any parameter passed to `run()`
will by default become the initial state.

But if your service has some specific internal formatting that must be
applied to this state, simply provide an `init_state()` function. This
receives the default state and the parameter passed to `run()`, and
returns the updated state.


~~~ elixir
defmodule KVStore do

  use Diet.Service.Anonymous
      state: %{}

  def init_state(default_state, initial_values) when is_list(initial_values) do
    initial_values |> Enum.into(default_state)
  end
  
  # ...
end
~~~

Call this with:

~~~ elixir
handle = KVStore.run(name: "Jose", language: "Elixir")

KVStore.put(handle, :name, "José")

IO.puts KVStore.get(handle, :name)      # => José
IO.puts KVStore.get(handle, :language)  # => Elixir
~~~

## Creating Pooled Services

You can create both named and anonymous pooled services. Simply add
the `pool:` option:

~~~ elixir
defmodule DBConnection do
  defstruct conn: nil, created: fn () -> DateTime.utc_now() end

  use(Diet.Service.Named,
      state: [ db: %DBConnection{} ],
      pool:  [ min: 2, max: 10, retire_after: 5*60 ])

  def init_state(state, _) do
    connection_params = Application.fetch_env(app, :db_params)
    conn = PG.connect(connection_params)
    %{ state | conn: conn }
  end

  def execute(stmt) do
    PG.execute(db.conn, stmt)
  end
  # ...
end
~~~

This creates a pool of database connections. It will always have at
least 2 workers running, and may have up to 10. Idle workers will be
culled after 5 minutes.

Here's how it is called:

~~~ elixir
result = DBConnection.execute("select * from table1")
~~~

You can also create anonymous pools. As with other anonymous services,
you'll need to keep track of the handle.

Here's the same database connection pool code, but set up to allow you
to create different pools for different databases.

~~~ elixir
defmodule DBConnection do
  defstruct conn: nil, created: fn () -> DateTime.utc_now() end

  use(Diet.Service.Anonymous,
      state: [ db: %DBConnection{} ],
      pool:  [ min: 2, max: 10, retire_after: 5*60 ])

  def init_state(state, connection_params) do
    conn = PG.connect(connection_params)
    %{ state | conn: conn }
  end

  def execute(stmt) do
    PG.execute(db.conn, stmt)
  end
  # ...
end
~~~

The only changes were to alter the type to `Diet.Service.Anonymous`
and to change the `new` function to accept the database connection to
be used.


# Supervision

The supervision rules differ between solo and pooled services, and
also between named and anonymous services.

## Solo Service Supervision

By default, solo services are created without supervisors. They are
linked to the process that created them.

You can have Diet create supervisors for you. 

### Named Solo Service Supervision

For a named
service, Diet creates a static one-for-one supervisor. You can
override the supervision options.

~~~ elixir
defmodule Logger do

  use(Diet.Service.Named,
      supervise: [ name: :log_super ])
 
  # ...
~~~

### Anonymous Solo Service Supervision ###

Anonymous services are created on an ad-hoc basis, and so Diet uses a
default supervision strategy of `simple_one_for_one`. 


## Pooled Supervision

When you create a pooled service, Diet always creates a couple of
supervisors and a scheduler process. This looks like the following:


                           +--------------------+
                           |                    |
                           |  Pool Supervisor   |
                         / |                    |\
                        /  +--------------------+ \
                       /                           \
    +--------------------+                       +--------------------+
    |                    |                       |                    |
    |    Scheduler       |                       | Worker Supervisor  |
    |                    |                       |                    |
    +--------------------+                       +--------------------+
                                                           |
                                                           |
                                                           v
                                                        +-----------------+
                                                    +------------------+  |
                                               +--------------------+  |  |
                                               |                    |  |  |
                                               |      Workers       |  |--+
                                               |                    |--+
                                               +--------------------+


When your client code accesses a pooled service, it is actually
talking to the scheduler process.


## State Protection

If you add the option `protect_state: true`, Diet will automatically
create an additional top-level supervisor and a vault process that does
nothing but save the state of services between requests. Should a
service crash, the supervisor will restart it using the saved state
from the vault, rather than using the default state.





