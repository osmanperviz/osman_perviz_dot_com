---
layout: post
title:  Polymorphism in Elixir part-1
permalink: elixir-protocols-explained-part-1
seo_description: Dive into the world of Elixir polymorphism with our comprehensive guide. Uncover the power of protocols in achieving polymorphism in functional programming, a key concept where traditional inheritance doesn't apply. Learn how Elixir's unique approach allows for flexible and intuitive code through the implementation of protocols, enabling functions to interact seamlessly with various data types. This post, the first in a two-part series, lays the theoretical groundwork, setting the stage for a practical exploration of protocol implementation in Elixir. Whether you're new to Elixir or looking to deepen your understanding, join us in exploring the dynamic capabilities of polymorphism in Elixir.
description: Dive into the world of Elixir polymorphism with our comprehensive guide. Uncover the power of protocols in achieving polymorphism in functional programming, a key concept where traditional inheritance doesn't apply. Learn how Elixir's unique approach allows for flexible and intuitive code through the implementation of protocols, enabling functions to interact seamlessly with various data types. This post, the first in a two-part series, lays the theoretical groundwork, setting the stage for a practical exploration of protocol implementation in Elixir. Whether you're new to Elixir or looking to deepen your understanding, join us in exploring the dynamic capabilities of polymorphism in Elixir.
date:   2022-01-10 15:01:35 +0300
image:  
tags:   [elixir, protocols ]
---

In today's post, we're going to explore the concept of protocols in depth. Unlike object-oriented applications, where inheritance is a key architectural element, functional applications, such as those built with Elixir and Erlang, don't rely on inheritance in the same way. This is because the Elixir and Erlang runtime environments don't incorporate the concept of inheritance into their data structures. However, this doesn't mean they lack advanced concepts altogether.

### Polymorphism in Other Languages

Object-oriented languages provide you with multiple inheritance or interfaces, allowing you to declare that objects belonging to multiple different classes, and all respond to the same method calls even doe underlying data are completely different.

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

#### Protocol Structuring
When it comes to organizing protocols, practices vary. Some developers prefer to keep the protocol in its own separate file, while others choose to include it within the same file as another module. My preferred approach is to maintain a dedicated directory, lib/protocols/, for storing actual protocol definitions. The organization within this directory generally follows the standard pattern, except that it skips the first layer of the namespace. For example, if my protocol is named Foo.Bar, I would place it in lib/protocols/bar.ex. In this directory, I also include the defimpl for all native and foreign data types.

As for the protocol implementation, I believe it should be kept as close to the data layer as possible. This means implementing the protocol in the files that define the struct. This approach helps in maintaining a clear and logical structure, ensuring that the protocol implementations are directly associated with the relevant data types.

{% highlight elixir %}
defmodule Foo.Struct do
  defstruct some_field: 0

  defimpl Foo.Bar do
    # actual implementation
  end
end
{% endhighlight %}



Since, in the first part of this two-part series, we focused more on the theoretical part related to protocols, in the next part, we will focus more on the practical part and actually, see how the protocols work in reality and how we can implement them.
