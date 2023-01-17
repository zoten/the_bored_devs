# Logger.metadata Gotcha in Elixir

> or, how not to shoot yourself in the foot with basics

<small>(originally posted in [Not So Senior](https://zoten.substack.com/p/loggermetadata-gotcha-in-elixir))</small>

I love `Logger`.

Yes, I have some “_complains”_. I’d like to be able to have a slightly more fine-grained control over it by default, e.g. module-specific levels, but nothing that a good ol’ custom backend cannot solve (with some boilerplate to copy-paste, probably) (1).

Also, I love `Logger`’s [`metadata`](https://hexdocs.pm/logger/1.14.3/Logger.html#metadata/1).

You just set them once and ***bam***, your entire process workflow is now filled with useful information from start to finish.

Sometimes it goes beyond, and __that__ may be a problem.

How does this happen? Let’s take a basic and flat-architected nanoservice that consumes events from a [RabbitMQ](https://www.rabbitmq.com/) instance, does stuff and then goes to the next message. I’ll make up a reasonable library’s API (called _SomeRabbitConsumer_) to fake it. Let’s assume each consumer is configured to start as a GenServer variation, so managing concurrent consumers is on the user’s hands to configure.

``` elixir
defmodule Metalog.Consumer do
  @moduledoc """
  RabbitMQ consumer
  """

  @impl SomeRabbitConsumer
  @spec handle_message(binary(), map(), any()) :: {:ok, any()}
  def handle_message(payload, meta, state) do
    with {:ok, event} ← decode_message(payload, meta),
         {:ok, _} ← do_handle_message(event) do
         {:ok, state}
    end
  end
    
  defp do_handle_message(%SomeEvent{} = event) do
    Logger.info(“Received a SomeEvent!”)
    Metalog.Domain.handle_some_event(event)
  end
    
  # …other handlers
    
  defp decode_message(payload, _), do: {:ok, Jason.decode!(payload)}
end
```

We have a callback where we decode the message and do some domain logic. Pretty basic. But what happens when the domain module is simply something like this?

``` elixir
defmodule Metalog.Domain do
  @moduledoc """
  Domain logic
  """

  @impl SomeRabbitConsumer
  @spec handle_some_event(SomeEvent.t()) :: :ok
  def handle_some_event(%SomeEvent{id: request_id, data: data) = event) do
    Logger.metadata(request_id: request_id)
    other_event_logic(data)
    :ok
  end
end
```

We set some metadata in the `Logger.metadata` call, so that any other Logger call in `other_event_logic/1` will include those (e.g.the `request_id` in this example, that will enable us to filter execution spans).

However, there’s a problem now. As we pictured it, we defined this simple architecture:

![Architecture]([https://github.com/zoten/the_bored_devs/assets/logger_metadata_gotcha_in_elixir/arch-1.webp](https://github.com/zoten/the_bored_devs/blob/main/assets/logger_metadata_gotcha_in_elixir/arch-1.webp))

We have a single process (the RabbitMQ consumer) and everything happens inside it. What happens is we log an innocent line in the consumer module, then set some metadata in our domain function and execute things, potentially logging other lines. What we all would expect is an output in the form (poetic license on formatting of metadata)

``` bash
> [info] metadata: [] message: Received a SomeEvent!
> [info] metadata: [request_id: 1] message: I’m into ‘other_event_logic’ function!
```

and at the first run that’s exactly what we would see. But the second message would yield to different result:

``` bash
> [info] metadata: [request_id: 1] message: Received a SomeEvent!
> [info] metadata: [request_id: 2] message: I’m into ‘other_event_logic’ function!
```

Ugh, what happened? Remember, we are inside the same process. That means that its state does not vanish, and is waiting for us when the execution terminates, the control passes to the message receiver and another message is de-queued from the broker. We now have a dirty log.

This is a simple case, of course, but put this into a more complicated code structure where that `Logger.metadata` call ends up more deep into the codebase, and you have a problem. You’ll find yourself seeing messages with span identifier you can’t match, or error information and stacktraces from older executions.

## Does this single-process consumer even make any sense?

That’s a legit question. Developers thirsty for optimizations will surely argue we could (or even should) spawn an ephemeral process per message and manage everything this way. While we could surely optimize our consumers to make advantage of Elixir’s parallelism (but hey, let’s keep this for another post!), there are cases when you don’t want to consume messages in parallel because of some logical dependency, or you don't want to buy yourself parallel problems - or just bad architecture you inherited.

## So, how do I solve this?

Depending on your application’s structure, there are a couple of solutions

### Explicitly set metadata

Just don’t call `Logger.metadata/1` at all. Explicitly call `Logger` using inline notation, such as in

``` elixir
Logger.info(“Message”, request_id: request_id)
```

**Pros:**

 * you’re not polluting your process’ metadata store
 * you are explicit in what you’re logging

**Cons:**

 * you may not have or want all the information you need to log along the chain (e.g. span ids that do not make sense in inner domain functions as arguments)

### Reset metadata

This is my preferred one, but requires some diligence in coding. It is making use of the function `Logger.reset_metadata/0` for emptying the metadata store.

I like to put a call in each “_entry point_” of the application that is a long-lived process. For example, this usually doesn’t need to apply to Phoenix controllers or GraphQL resolvers, since those requests already happen in a separate process. In our example, our `handle_message` function would look like

``` elixir
  # …

  @impl SomeRabbitConsumer
  @spec handle_message(binary(), map(), any()) :: {:ok, any()}
  def handle_message(payload, meta, state) do
    Logger.reset_metadata()
    
    with {:ok, event} ← decode_message(payload, meta),
         {:ok, _} ← do_handle_message(event) do
         {:ok, state}
    end
  end

  # …
```

**Pros:**

 * clean code, clean `Logger` calls

**Cons:**

 * refactorings need some caution. Moving the execution of a function into a separate process (e.g. using `Tasks`) may make you lose all your precious metadata
 * it’s not immediate to understand what an inner `Logger` call will contain. You have to trust the software architecture

### Moving to a different process

Of course, you can take your execution code and move it to a separate, ephemeral process, even if a single one. In our example our `handle_message` function would look like

``` elixir
  def handle_message(payload, meta, state) do
    Task.async(fn →
      with {:ok, event} ← decode_message(payload, meta),
           {:ok, _} ← do_handle_message(event) do
           {:ok, state}
      end
    end)
    # await, etc.
  end
```

Doing this, our metadata call would have effects only on the task’s spawned process.

**Pros:**

 * Auto cleans itself
 * If handled properly, a task's crash will not kill the consumer

**Cons:**

 * This random free process seems somewhat "dirty"
 * Slightly uglier code :)

## Wrapping up

This antipattern caused me and my team more than some minute of headache to realize where the problem was - just imagine those guys querying logs and database for an id that was perfectly fine. The proposed solutions are all pretty simple, but as you can see they all depend somehow on your coding diligence and are not error-proof. Our solutions, architecture and proposals may differ from your needs: why don’t you share your experience?

## Footnotes

 - (1) I discovered while writing that FlexLogger exists. It’s a 2017 project, so I think before Elixir’s switch to using Erlang’s :logger, but it may still work. Have to try it!
