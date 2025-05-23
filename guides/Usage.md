# Using the EventStore

First you need to define your own event store module:

```elixir
defmodule MyApp.EventStore do
  use EventStore, otp_app: :my_app
end
```

## Writing to a stream

Create a unique identity for each stream. It **must** be a string. This example uses the [elixir_uuid](https://hex.pm/packages/elixir_uuid) package.

```elixir
stream_uuid = EventStore.UUID.uuid4()
```

Set the expected version of the stream. This is used for optimistic concurrency. A new stream will be created when the expected version is zero.

```elixir
expected_version = 0
```

Build a list of events to persist. The data and metadata fields will be serialized to binary data. This uses your own serializer, as defined in config, that implements the `EventStore.Serializer` behaviour.

```elixir
alias EventStore.EventData
alias MyApp.EventStore

defmodule ExampleEvent do
  defstruct [:key]
end

events = [
  %EventData{
    event_type: "Elixir.ExampleEvent",
    data: %ExampleEvent{key: "value"},
    metadata: %{user: "someuser@example.com"},
  }
]
```

Append the events to the stream:

```elixir
:ok = EventStore.append_to_stream(stream_uuid, expected_version, events)
```

### Appending events to an existing stream

The expected version should equal the number of events already persisted to the stream when appending to an existing stream.

This can be set as the length of events returned from reading the stream:

```elixir
alias MyApp.EventStore

events =
  stream_uuid
  |> EventStore.stream_forward()
  |> Enum.to_list()

stream_version = length(events)
```

Append new events to the existing stream:

```elixir
alias EventStore.EventData
alias MyApp.EventStore

new_events = [ %EventData{..}, ... ]

:ok = EventStore.append_to_stream(stream_uuid, stream_version, new_events)
```

#### Why must you provide the expected stream version?

This is to ensure that no events have been appended to the stream by another process between your read and subsequent write.

The `c:EventStore.append_to_stream/4` function will return `{:error, :wrong_expected_version}` when the version you provide is mismatched with the stream. You can resolve this error by reading the stream's events again, then attempt to append your new events using the latest stream version.

### Optional concurrency check

You can choose to append events to a stream without using the concurrency check, or having first read them from the stream, by using one of the following values instead of the expected version:

- `:any_version` - No concurrency checking; allow any stream version (including no stream).
- `:no_stream` - Ensure the stream does not exist.
- `:stream_exists` - Ensure the stream exists.

```elixir
alias MyApp.EventStore

:ok = EventStore.append_to_stream(stream_uuid, :any_version, events)
```

## Reading from a stream

Read all events from a single stream, starting at the stream's first event:

```elixir
alias MyApp.EventStore

{:ok, events} = EventStore.read_stream_forward(stream_uuid)
```

## Reading from all streams

Read all events from all streams:

```elixir
alias MyApp.EventStore

{:ok, events} = EventStore.read_all_streams_forward()
```

By default this will be limited to read the first 1,000 events from all streams only.

## Stream from all streams

Stream all events from all streams:

```elixir
alias MyApp.EventStore

all_events = EventStore.stream_all_forward() |> Enum.to_list()
```

This will read *all* events into memory, it is for illustration only. Use the `Stream` functions to process the events in a memory efficient way.

## Linking events between streams

Event linking allows you to include events in multiple streams, such as copying an event from one stream to another, but only a reference to the original event is stored.

An event may be present in a stream only once, but may be linked into as many streams as required.

Linked events are used to build the `$all` stream containing every persisted event, globally ordered.

Use each recorded event's `event_number` field for the position of the event within the read/received stream. The `stream_uuid` and `stream_version` fields refer to the event's original stream.

Read source events:

```elixir
alias MyApp.EventStore

{:ok, events} = EventStore.read_stream_forward(source_stream_uuid)
```

Link read events to another stream:

```elixir
alias MyApp.EventStore

:ok = EventStore.link_to_stream(target_stream_uuid, 0, events)
```

You can also pass a list of `event_ids` instead of recorded event structs to link events.

## Deleting streams

There are two ways to delete streams. Soft delete and Hard delete.

Use soft delete when you no longer care about a streams events, but want to preserve the full history of events.

Use hard delete when you want a stream to go away more than a bad case of viral gastroenteritis (for example GDPR compliance).

### Soft delete

Will mark the stream as deleted, but will not delete its events. Events from soft deleted streams will still appear in the globally ordered all events ($all) stream and in any linked streams.

A soft deleted stream cannot be read nor appended to. Subscriptions to the deleted stream will not receive any events but subscriptions containing linked events from the deleted stream, such as the global all events stream, will still receive events from the deleted stream.

#### Examples

Delete a stream at any version:
```elixir
:ok = MyApp.EventStore.delete_stream("stream1", :any_version, :soft)
```

Delete a stream at an expected version:
```elixir
:ok = MyApp.EventStore.delete_stream("stream2", 3, :soft)
```

Delete stream will use soft delete by default so you can omit the type:
```elixir
:ok = MyApp.EventStore.delete_stream("stream1", :any_version)
```

### Hard delete

Will permanently delete the stream and its events. **This is irreversible and will remove data**. Events will be removed from the globally ordered all events ($all) stream and any linked streams.

After being hard deleted, a stream can later be appended to and read as if had never existed.

#### Examples

Since hard deletes are destructive and irreversible they are disabled by default. To use hard deletes you must first enable them for the event store:
```elixir
defmodule MyApp.EventStore do
  use EventStore, otp_app: :my_app, enable_hard_deletes: true
end
```

Or via config:
```elixir
config :my_app, MyApp.EventStore, enable_hard_deletes: true
```

Hard delete a stream at any version:
```elixir
:ok = MyApp.EventStore.delete_stream("stream1", :any_version, :hard)
```

Hard delete a stream that should exist:
```elixir
:ok = MyApp.EventStore.delete_stream("stream2", :stream_exists, :hard)
```
