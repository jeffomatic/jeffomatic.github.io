---
layout: post
title: Accessing Rails helpers outside views
description: A quick hack to expose Rails view helpers to the rest of your code.
---

<a href="https://www.flickr.com/photos/rg69olds/7876825078" title="fence by R G, on Flickr"><img src="https://farm9.staticflickr.com/8430/7876825078_3d298a8ea2.jpg" width="500" height="333" alt="fence"></a>

Prominent among my gripes about Rails is its handling of view helpers. These are mixed into the view object to provide easy access from template code, but the price of this convenience is a total lack of namespacing, reminiscent of ye olden PHP days.

What's more is that Rails seems to have an excessively narrow opinion about what constitutes a "view": view helpers are easy to call from HTML and email templates, but there's no consideration paid to other possible outlets from your app, including HTTP headers, command line output, file streams, or email subject lines.

Solutions to the latter problem are divers and arcane, frequently involving an undocumented, hard-to-remember expression that dispatches helpers through the controller instance. Of course, this works if you've actually got a controller instance handy, but isn't so great when you're, say, running a Rake task. And in the first place, since most helper methods are just text filters, it seems silly that they should be coupled to this or that object. Why can't we just call them from anywhere, as the generic utilities that they are?

As it turns out, most of the built-in helper methods are mixed into the `ActionView::Helpers` module as instance methods. In order to call them as globals, you need to host them in another module where they appear as class methods. And thus, my favorite workaround:

{% highlight ruby %}
# Put this into extra/my_proj/helpers.rb
module MyProj
  module Helpers
    extend ActionView::Helpers
  end
end
{% endhighlight %}

With this in place, you can call your helpers as globally-accessible methods, as follows:

{% highlight ruby %}
MyProj::Helpers.pluralize(1, 'person')
MyProj::Helpers.simple_format(string)
{% endhighlight %}

The pleasant thing about this solution is that it's easy to remember, and works from whatever context you're in, whether it's the body of a Rake task, a controller action, a method on a model class, or even your view templates.