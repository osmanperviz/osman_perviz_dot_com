---
layout: post
title:  Building Concurrent ETL Pipelines with Elixir and GenStage
seo_description: Detailed tutorial about ETL in Elixir using GenStage and data validation.
permalink: etl-data-pipelines-in-elixir
description: Detailed tutorial about ETL in Elixir using GenStage and data validation.
date:   2022-10-15 15:01:35 +0300
image:  
tags:   [elixir, etl, genstage]
---

Elixir, the functional and concurrent programming language, in conjunction with GenStage, its computation pipeline architecture, elegantly lends itself to ETL (Extract, Transform, Load) operations — the bloodstream of data-driven work. In this guide, we'll roll up our sleeves, bring out our keyboards, and build a live data processing pipeline using Elixir and GenStage.
### **Transforming Data the Elixir Way**
To understand how powerful working with Elixir is, we first need to introduce some common source data. In this example, we will use trial data generated from a fake log stream, populating it in real time.

{% highlight elixir %}
defmodule LogStream do  
    def start_link do
        # log stream generates a new log entry every 1 second
        Task.start_link(fn -> generate_logs() end)
    end

    defp generate_logs do
        Stream.cycle(["INFO", "WARN", "ERROR"])
        |> Stream.zip(Stream.interval(1_000))
        |> Enum.each(fn {level, _} -> IO.puts("#{timestamp()} #{level} some interesting log...") end)
    end

    defp timestamp do
        DateTime.now |> DateTime.to_string
    end
  end
end
{% endhighlight %}
Log entries will be formatted as: 
{% highlight elixir %}
"{current timestamp} {log level} some interesting log..."
{% endhighlight %}

### **A Deeper Dive into GenStage for Data Extraction in Elixir**
Introduced in Elixir 1.3, GenStage is a behavior abstraction for working with Producer and Consumer models, offering back-pressure and other complex features essential for high-volume data processing tasks like ETL. GenStage was built to support data-flow computations like concurrent processing, queuing, staged processing, and system event handling among others.

##### Understanding Producer/Consumer Dynamics in GenStage
In the GenStage parlance, the producer sends data downstream while the consumer receives data upstream. Work is scheduled at the optimal pace, offering mechanisms to regulate the flow of data and prevent overloading.

Three main roles bring a GenStage system to life:

* Producer: Source of events. It offers a subscription mechanism for other stages to attach.
* Consumer: Sink of events. It holds subscriptions to producers and consumes their data.
* Producer-Consumer: A hybrid role, it works as both a consumer to a producer and a producer to a consumer, enabling chaining between stages.

We introduce Elixir's GenStage by setting up a producer that continuously reads from our log stream. This producer will be the basis of our extraction process in our ETL workflow.

{% highlight elixir %}
defmodule LogProducer do
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, :ok, name: __MODULE__)
  end

  def init(:ok) do
    {:producer, [], dispatcher: GenStage.BroadcastDispatcher}
  end

  def handle_call({:push_log, log}, _from, state) do
    # the log will be pushed to all consumers
    {:reply, :ok, [log], state}
  end
end
{% endhighlight %}

We set up the dispatcher as `GenStage.BroadcastDispatcher` to ensure the log is sent to all consumers, not just one.
### **Transforming Data with Multiple GenStage Producers/Consumers**
Our log stream will be loaded with different log levels, and we want to process them separately. For that, we create a GenStage producer/consumer for each log level.
{% highlight elixir %}
defmodule LogProcessor do
  use GenStage

  def start_link(log_level) do
    GenStage.start_link(__MODULE__, log_level, name: String.to_atom("LogProcessor.\#{log_level}"))
  end

  def init(log_level) do
    GenStage.sync_subscribe(self(), to: LogProducer, max_demand: 1)
    {:producer_consumer, log_level}
  end

  def handle_events([log], _from, log_level) do
    [timestamp, level, message] = String.split(log)
    if level === log_level do
      IO.puts "Processing \#{log_level} log..."
      # Here is where the actual transformation would happen
      # For now, just print out the message part
      IO.puts message
    end
    # log is processed, ask for next log
    {:noreply, [], log_level}
  end
end
{% endhighlight %}
### **Loading Data with GenStage Consumers**
Vanilla GenStage consumers suit the loading component after the data has been through the extraction and transformation stages. We will set up one consumer that subscribes to all transformed log streams.

{% highlight elixir %}
defmodule LogConsumer do
  use GenStage

  def start_link do
    GenStage.start_link(__MODULE__, :ok, name: __MODULE__)
  end

  def init(:ok) do
    Enum.each(~w(INFO WARN ERROR), fn level ->
      GenStage.sync_subscribe(self(), to: String.to_atom("LogProcessor.#{level}"), max_demand: 1)
    end)
    {:consumer, nil}
  end

  def handle_events(logs, _from, state) do
    # Actual implementation of loading log to the final destination would go here
    # For now, just print out the length of the logs
    IO.puts("Loaded: #{length(logs)} logs")
    {:noreply, [], state}
  end
end
{% endhighlight %}

### *Ensuring Data Validity and Consistency with Elixir*
Data validation and consistency checks are critical stages in any ETL (Extract, Transform, Load) pipeline. Without these, the data delivered at the end of the pipeline could be unreliable or even misleading. Fortunately, Elixir provides multiple approaches to ensure data validity and consistency, making it well-suited to ETL operations.

#### Data Consistency with Elixir
Ensuring data consistency typically involves verifying that the data adheres to defined rules or schemas. Elixir provides solutions like Structs and the Ecto library for such challenges.

* Structs, in Elixir, provide a way of defining custom data types with pre-set values and validations where necessary.

Here's a simple example of defining a struct for a log entry:

{% highlight elixir %}
defmodule LogEntry do
  defstruct level: "INFO", message: "", timestamp: DateTime.utc_now
end
{% endhighlight %}

A struct ensures that each LogEntry must contain these fields and automatically assigns them default values if none are provided.

* Ecto, on the other hand, is a database wrapper and a fantastic query builder. It also provides a powerful system for data mapping and validation called Changesets.
Imagine adding a Changeset to our data entry to ensure that each log message is a non-empty string. Here's what that might look like:

{% highlight elixir %}
defmodule LogEntry do
  use Ecto.Schema
  
  @enforce_keys [:timestamp, :level, :message]
  defstruct [:timestamp, :level, :message]
  
  import Ecto.Changeset
  
  def changeset(entry, params \\ %{}) do
    entry
    |> cast(params, [:timestamp, :level, :message])
    |> validate_required([:timestamp, :level, :message])
    |> validate_length(:message, min: 1)
  end
end
{% endhighlight %}

#### Data Validation in Elixir with Custom Validators

For more complex rules that can't be satisfied with built-in validators, Elixir also allows us to write custom validator functions.

Consider the need for the message to be at least 10 characters and to always begin with the phrase "Log Entry: ". A custom validator for this requirement would look as follows:

{% highlight elixir %}
def validate_message_format(changeset) do
  validate_change(changeset, :message, fn :message, message ->
    if message =~ ~r/^Log Entry: .{10,}$/ do
      []
    else
      [{:message, "must begin with 'Log Entry: ' and be at least 10 characters long"}]
    end
  end)
end
{% endhighlight %}

The function validate_change takes the changeset and the attribute to be validated as arguments, and if the changed value doesn't match the prescribed pattern, it returns an appropriate error message.

Implementing these techniques in your ETL pipeline will ensure data coming out of your ETL pipeline follows the specified format rules and contains valid entries.

This guide only scratches the surface of Elixir's powerful concurrency and GenStage's reliable back-pressure handling.I like to hear more about how you’ve managed your ETL needs, particularly data consistency and validation, with Elixir. Share your ways, passionate thoughts, and experiences with us!