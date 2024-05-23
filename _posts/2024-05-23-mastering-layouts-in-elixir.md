---
layout: post
title: Mastering Layouts in Phoenix for Elixir LiveView Apps
seo_description: Discover the versatility of Phoenix Layouts for structuring your Elixir LiveView applications. Whether you're crafting simple static pages or building complex, dynamic user interfaces, Phoenix Layouts provide the essential framework for seamless, real-time user interactions. Learn how to enhance your web development process with Phoenix Layouts today!
description: Discover the versatility of Phoenix Layouts for structuring your Elixir LiveView applications. Learn how to enhance your web development process with Phoenix Layouts today!
permalink: mastering-layouts-in-elixir
date:   2024-05-22 15:01:35 +0300
image:  "images/layout-blog-post.jpg"
tags:   [elixir, liveview]
---

Phoenix Layouts offer a powerful way to structure your Elixir LiveView application. From simple, static pages to complex, real-time user interactions, layouts cover a vast spectrum of possibilities. But how well do you understand these layout options and when to use them? Let's explore!
### Introduction to Elixir LiveView
Elixir's LiveView is a revolutionary tool that empowers developers to build interactive, real-time applications without the need for JavaScript. When integrated with Phoenix, it opens a world of opportunities, one of which includes multiple layout options.
### Phoenix Layouts Unveiled
In a Phoenix application, layouts are akin to templates that determine the structure and appearance of web pages. They enable code reusability, maintain consistent design across pages, and can even adapt dynamically to pop-out sections such as a sidebar or a navigation bar.
### Mapping the Phoenix Layout Landscape
Phoenix provides several layout choices: the root layout, the normal layout, live layouts, and page-specific layouts.

###### The Root Layout
The root layout serves as a parent template for all pages in the application. It proves particularly useful for sections of your app that remain constant, such as headers or footers.

{% highlight elixir %}
# Setting the root layout in live_view router
 defmodule MyAppWeb.Router do
   use MyAppWeb, :router

   pipeline :browser do
    ...
    plug :put_root_layout, {MyAppWeb.LayoutView, "root.html"}
   end
   ...
 end
{% endhighlight %}
In this example, ```root.html``` is the root layout for all LiveViews, and it remains static across all pages.

###### The Normal Layout
The `normal layout` is usually used for static content. It is ideal for delivering a consistent appearance across non-live or `static` pages.
{% highlight elixir %}
# lib/my_app_web/layout_view.ex
defmodule MyAppWeb.LayoutView do
  use Phoenix.View

  def render("app.html", assigns) do
    ~L"""
      ...
      <main role='main'>
       <%= render @view_module, @view_template, assigns %>
      </main>
      ...
    """
    end
  end
{% endhighlight %}
In `app.html.leex`, the dynamic content is rendered within the <main> tag in line with our layout's design.

###### Live Layouts
Live Layouts are ideal for delivering real-time, dynamic content. Below you can see a `LiveLayout` example for a chat application:
{% highlight elixir %}
# lib/my_app_web/live/live_layout.ex
defmodule MyAppWeb.LiveLayout do
  use MyAppWeb, :live_layout

  def render(assigns) do
    ~L"""
      <div class="chat-container">
        <%= live_render(@socket, MyAppWeb.UserListLive) %>
        <%= @inner_content %>
      </div>
    """
  end
end
{% endhighlight %}
The `UserListLive` LiveView is always rendered in the chat container irrespective of which LiveView is shown on the right.

###### Page-specific Layouts

Page-specific layouts let you define a particular layout for individual pages.
{% highlight elixir %}
# lib/my_app_web/live/product_live/show.ex
defmodule MyAppWeb.ProductLive.Show do
 ...
 @layout {MyAppWeb.LayoutView, :product}
 ...
end
{% endhighlight %}
Here, we defined a layout specifically for the Product page view.

###### Challenge Conquered: Combining `live_session` with Layouts

Phoenix also lets you define a layout within a `live_session`. This is especially useful if you want all LiveViews within the session to use the same LiveLayout.

{% highlight elixir %}
# lib/my_app_web/router.ex
defmodule MyAppWeb.Router do
 ...
 live_session :session, layout: {MyAppWeb.LayoutView, :session} do
   live "/dashboard", DashboardLive
   live "/profile", ProfileLive
 end
 ...
end
{% endhighlight %}

In this snippet, both DashboardLive and ProfileLive would utilize the same `:session` layout.

Exploring these different layout options, we see Phoenix Layouts provide flexibility to structure your Elixir LiveView apps. Remember, the goal is not to use all layout options, but to use the right layout options where they fit best.





