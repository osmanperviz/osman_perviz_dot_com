---
layout: post
title:  Demistify elixir actor model
description: Deep dive into what exactly the Actor Model is, and how exactly the Actor Model is implemented in Elixir/Erlang.
seo_description: Deep dive into what exactly the Actor Model is, and how exactly the Actor Model is implemented in Elixir/Erlang.
permalink: demistify-elixir-actor-model
description: Deep dive into what exactly the Actor Model is, and how exactly the Actor Model is implemented in Elixir/Erlang.
date:   2021-12-01 15:01:35 +0300
image:  
tags:   [elixir, actor model]
---

In this blog post, I will try to demystify Elixir Actor Model and deep dive into what exactly the Actor Model is, and how exactly the Actor Model is implemented in Elixir/Erlang.
To really understand these concepts, we will be writing code snippets to have a deeper understanding of the concept.

# What is the Actor Model?
It's a conceptual model of concurrent computation that originated in 1973. The actor model is designed to be a message-driven, non-blocking unit of computation. The actor model is not much different than a simple object and they have a few additional mechanisms that allow them to be message-driven and deal with messages that will be sent and delivered in an asynchronous way.
An actor can hold its own state and can decide how to process the next message based on that state.


The Actor Model is not the only concurrency model outside. Some of them are:

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;• CSP (Communication Sequential Processes) implemented in GO Language\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;• STM (Software Transactional Memory) implemented in Haskell\
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;• TT-calculus and others...

# Actor Model on the BEAM
In the context of the BEAM, an actor is a process (not OS process) running in the Erlang virtual machine.

Processes are isolated from each other, run concurrent to one another and communicate via message passing. Each process is compleatly separated from all the other process, it has a separate memory, garbage collector, nearly separate run time and the only way to communicate with the outside world is by sending messages to other processes located locally, or remotely on another connected Node.

# Messages and Mailboxes
##### &nbsp;&nbsp;&nbsp; Messages
&nbsp;&nbsp;&nbsp; • Messages in Elixir are asynchronous\
&nbsp;&nbsp;&nbsp; • When you send a message to an actor, the message is placed instantly in the actor’s mailbox\
&nbsp;&nbsp;&nbsp; • Actor process only one message by the time (FIFO) 

##### &nbsp;&nbsp;&nbsp; Mailboxes

&nbsp;&nbsp;&nbsp; • Mailboxes in Elixir are lock-free queues\
&nbsp;&nbsp;&nbsp; • Works asynchronously (doesn't wait for response from another actor)

# Actor Creation

An actor is created by using the *spawn* function or the *spawn_link* function and it can create also an child actor.

##### Spawn
*Spawn* takes a function and returns a *process identifier*, aka a *PID*. The function passed to *spawn* takes no arguments and its structure is expected to be an infinite loop.

All the messages are consumed via a *receive* block. 
*receive* block processes messages in the order received and allows messages to be pattern matched.


Messages are sent to the processes using the *send/2* function.

{% highlight elixir %}
actor = spawn fn ->
  receive do
    {sender, :ping} -> "Got ping"
  end
end

send(actor, {self, :ping}) #=>  Got ping
{% endhighlight %}

This example above creates an actor that can only respond to a single message. That message MUST be the tuple *{:ping}*. Any other message is ignored.

When the message *{:ping}* arrives, the actor prints out "Got ping" and then
the function of the actor returns. That is interpreted as a *normal* exit, similar to having the *run()* method of a Java thread return.


##### spawn_link

Similar to *spawn*, *spawn_link* creates a new process, but links the current and new process so that if one crashes, both processes terminate.
Linking processes is crucial  to the Elixir  philosophy of letting programs crash instead of trying to rescue from errors. With linking them we can better interactions with our actors.
Linked actors get notified if one of them goes down(by either exiting normally or crashing). To receive this notification, we have to tell the system to *trap the exit* of
an actor; it then sends us a message in the form: *{:EXIT, pid, reason}* when
an actor goes down but ONLY if we start the process using spawn_link.

# Actor State

Since Elixir is immutable, you may be wondering how state is held. Actor state is nothing else than internal actor data, which can be
accessed directly only from an actor-owner. When one actor wants to read the data of another actor, it has to send a message and receive back the state. To maintain state in an actor, we can use pattern matching and recursion.

{% highlight elixir %}

defmodule Counter do
 def loop(count) do
  receive do
   {:next} ->
     IO.puts("Current count: #{count}")	
     loop(count + 1)
   end
 end
end

counter = spawn(Counter, :loop, [1]) 

send(counter, {:next}) => # Current count: 1 
send(counter, {:next}) => # Current count: 2

{% endhighlight %}

# Actor communication

Since an actor is a isolated process with internal state and mailbox, it has its own *PID* (process id), which is a unique actor identifier, used for message sender or receiver specification.

##### Message passing
For direct message send it uses *send* function, which takes two arguments: *PID* of actor-receiver and the message itself. It runs *asynchronously*, so after this function invocation, actor will continue its execution.

{% highlight elixir %}
  send(other_actor_pid, message)
{% endhighlight %}


##### Message receive
For message receive it uses receive construction, which includes:

&nbsp;&nbsp;&nbsp; - Pairs `<msg pattern 1> -><related actions>`. If next message from mailbox matches to a given pattern, related actions is performed. Otherwise, actor waits for a new message matching to any other pattern\
&nbsp;&nbsp;&nbsp; - Statement *after*, which specifies timeout for message delivery. If no new messages in mailbox in this time, the actor stops waiting and continues execution.

{% highlight elixir %}

defmodule Salutator do
  def run do
    receive do
      {:hi, name} ->
        IO.puts "Hi, #{name}
      {_, name} ->
        IO.puts "Hello, #{name}
      after
        1_000 -> "nothing after 1s"
    end
  end
end
		 
pid = spawn(Salutator, :run, [])

send(pid, {:hi, "Mark"})
  {:hi, "Mark"}
=> # Hi, Mark
		
send(pid, {:hello, "Suzie"})
  {:hello, "Suzie"}
=> # Hello, Suzie

{% endhighlight %}


# Parallelism
Parallelism means doing the same operations at the exact same time on independant data.

In Elixir, processes are separate contexts (isolation) and you can have hundreds of thousands of processes on a single CPU (lightweight). If your computer has multiple cores, Elixir will run processes on each of them in parallel.

##### &nbsp;&nbsp;&nbsp; • Parallelism in action

{% highlight elixir %}

defmodule Parallel do
  def map(collection, fun) do
    parent = self()
    
    processes = Enum.map(collection, fn(e) ->
     spawn_link(fn() ->
      send(parent, {self(), fun.(e)})
     end)
    end)

    Enum.map(processes, fn(pid) ->
      receive do
        {^pid, result} -> result
      end
    end)
  end 
end
{% endhighlight %}

Let's create some "heavy" function that will multiply numbers with one second delay..

{% highlight elixir %}

heavy_double = fn(x) -> :timer.sleep(1000); x * 2 end

{% endhighlight %}

In `sequential` call the delay of one second occurred ten times and so the entire call to `Enum.map` took 10 seconds.

{% highlight elixir %}

:timer.tc(fn() -> Enum.map([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], heavy_double) end)
=>  # {10010165, [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]} <= about 10 seconds

{% endhighlight %}

In *parallel* call process got launched per element of the input collection, they all waited one second, 
and then returned their result, so the entire call to *Parallel.map* took 1 second.

{% highlight elixir %}

:timer.tc(fn() -> Parallel.map([1, 2, 3, 4, 5, 6, 7, 8, 9, 10], heavy_double) end)
=> # {1001096, [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]} <= about 1 second 

{% endhighlight %}

*Parallelism* is fundamentally about taking something you can do in a sequential (one process/thread) program and making it faster by doing parts of it at the same time.

# Summary
An actor in Elixir is called a process. You create new processes using one of the variants of the spawn function, send messages to it using the *<-* operator, and receive messages using a receive expression

Processes all contain a mailbox where messages are passively kept until consumed via a receive block. 
Actor State is internal actor data, which can be accessed directly only from an actor-owner. 
When one actor wants to read the data of another actor, it has to send a message and receive back the state.


