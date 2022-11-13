---
layout: post
title:  Polymorphism in Elixir part-2
description: In the second part of this two-part series, we are digging more into the practical part of protocols, and see how the protocols work in reality and how we can implement them.
date:   2022-01-20 15:01:35 +0300
image:  
tags:   [elixir]
---

In [first part](https://osmanperviz.com/elixir-protocols-explained-part-1) of our two-part series, we more focuse on teretical part, in second we will digg more into the practical part of protocols, and see how the protocols work in reality and how we can implement them.

Here I will present one live example and it used in one of our projects.

{% highlight elixir %}
defimpl LoggableEvent, for: Any do
  def to_log(%event_type{} = event) do
    event_type = event_type |> Atom.to_string() |> String.split(".") |> Enum.at(-1)

    event_data =
      event
      |> Map.from_struct()
      |> Enum.reduce("", fn {key, value}, event_as_string ->
        event_as_string <> " #{key}=" <> inspect(value)
      end)

    ~s(#{event_type}:#{event_data})
  end

  def to_log(_event), do: raise(ArgumentError, "Implement the LoggableEvent Protocol")
end
{% endhighlight %}