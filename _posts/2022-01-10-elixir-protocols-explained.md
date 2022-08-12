---
layout: post
title:  Polymorphism in Elixir part-1
description: how achieve polymorphism in Elixir.
date:   2022-02-10 15:01:35 +0300
image:  
tags:   [elixir]
---

In today's post, we will deep dive into the potion of protocols. Since inheritance does not play a big role in the architecture of functional applications like it does in object-oriented ones and
it's true that Elixir and Erlang runtime don't have a concept of inheritance in their data but that doesn't mean that they don't have a higher-order concept.

### Polymorphism in Other Languages

Object-oriented languages provide you with multiple inheritance or interfaces and allow you to declare that objects belonging to multiple different classes,  and all respond to the same method calls even doe underlying data are completely different.

{% highlight elixir %}
// Ruby
class Truck < Vehicle

// Java
public class Truck implements Drivable
{% endhighlight %}

### Polymorphism in Elixir

Elixir has several mechanisms that allow us to write expressive and intuitive code. One of them is *Protocol*.\
Protocols are a mechanism to achieve polymorphism in Elixir. Creating an Protocol allows us to create our own implementation for our own data.\
This way a function could accept any data type that implemented a particular protocol, and it would be able to work with that data type.\
Good part is that it would not have to know anything else about that data type, allowing different data types to put their own spin on a piece of functionality.

One of the most popular protocols in 3rd party libraries comes from the [Json](https://hexdocs.pm/json/readme.html) library, which encodes and decodes JSON data. In library sorce we can find protocal called [Json Encoder](https://github.com/cblage/elixir-json/blob/master/lib/json/encoder.ex). It defines single function `encode` with no definition.

{% highlight elixir %}
defprotocol JSON.Encoder do
 @moduledoc "..."

 @type t :: term
 @type opts :: Json.Encode.opts()

 @fallback_to_any true

 @doc "..."
 @spec encode(t, opts) :: iodata
 def encode(value, opts)
end

{% endhighlight %}
When the *Protocol* is implemented, other parts of library can call *encode()* and it will encode that data to JSON.
{% highlight elixir %}
JSON.Encoder.encode() # <- pass in any data (string, integers, maps, list etc..)
{% endhighlight %} 

You may be wondering what is the point of this formality if we could write a function with multiple function heads that's matches that type of data, like

{% highlight elixir %}

def encode(value) when is_atom(value) do
 ...
end

def encode(value) when is_binary(value) do
 ...
end

def encode(value) when is_integer(value) do
 ...
end
{% endhighlight %}

after all, we have guard classes for most data types in Elixir, the goal is *extensibility*.

Having to implement a protocol for all types may quickly become monotonous and exhausting. You can define fallback behavior for types that don’t implement your protocol by implementing the protocol for Any. Let’s have a look in Json.Encode default implementation.

{% highlight elixir %}
defimpl JSON.Encoder, for: Any do
  @moduledoc """
  Falllback module for encoding any other values
  """

  @doc """
  Encodes a map into a JSON object
  """
  def encode(%{} = struct) do
    ...
  end

  def encode(x) do
    ...
  end
end
{% endhighlight %}




