---
layout: post
title:  Building Concurrent ETL Pipelines with Elixir and GenStage
seo_description: Unlock the power of Elixir for building efficient, concurrent ETL pipelines with our in-depth tutorial. Discover how to leverage GenStage for robust data processing, ensuring high performance and reliability in your data-driven applications. From extracting real-time log streams to transforming and loading data with precision, learn the Elixir way of handling ETL tasks. This guide also delves into essential data validation techniques, ensuring your data's integrity and consistency. Whether you're new to ETL processes or looking to enhance your existing pipelines with Elixir's concurrency capabilities, this tutorial offers valuable insights and practical examples to elevate your data processing workflows.
permalink: etl-data-pipelines-in-elixir
description: This guide details how to build efficient, concurrent ETL pipelines using Elixir and GenStage, offering insights into data validation and processing to enhance data-driven applications.
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

#### The Asynchronicity Advantage

GenStage takes concurrency built into Elixir and capitalizes on it, offering a series of abstractions to manage the concurrency across different stages of a data processing pipeline.

GenStage works by splitting the roles into stages, with each stage acting as a mini application, handling its own data processing and passing on the results to the next stage. Asynchronous here means each stage works independently and in conjunction with each other stage.

Comfortably handling back-pressure or the build-up of data at any stage is GenStage's selling point. For example, if the consuming stage can't keep up with the producing stage, GenStage reduces the load sent to the consumer, preventing data flow issues, a feature lacking in central-thread languages like Ruby, where one thread handles multiple actions sequentially.

#### Why GenStage Over Single-Threading/Multi-Threading
The preference arises from the flexibility and performance GenStage offers.

* Speed & Efficiency: GenStage's asynchronous nature means all available CPU resources are efficiently used. This approach delivers a notable speed advantage over single-threaded languages.

* Resilience & Fault Tolerance: Unlike typical scenarios in a multithreaded environment, if one Elixir (GenStage) process fails, it doesn't crash the entire application. Elixir's support for advanced features like Supervisor trees makes it resilient against failures.

* Scalability: When it comes to scalability, GenStage outperforms both single-threaded and multi-threaded languages. GenStage efficiently handles a large number of processes without a torrent of threads overburdening the system resources.

This guide only scratches the surface of Elixir's powerful concurrency and GenStage's reliable back-pressure handling.I like to hear more about how you’ve managed your ETL needs, particularly data consistency and validation, with Elixir. Share your ways, passionate thoughts, and experiences with us!