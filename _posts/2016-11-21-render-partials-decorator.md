---
layout: post
title:  "Rendering Rails partials using decorators"
date:   2016-11-21 10:04:00 +0000
categories: render partials rails decorators
---

I sometimes roll my own decorators. You could call them 'view objects'. They augment the model (often, but not always an `ApplicationRecord`, in Rails 5 speak).

It can all get a little confusing, so here's my preferred way of doing things when using partials to simplify my sometimes epic (ugh) RESTful views.

* Render a partial, passing the model
* In the partial, decorate the model
* Render the decorated model using the normal Rails view helpers

In more detail:

1. In the top-level view template or layout, call `render` on a suitably named partial, passing the model as the object of said partial.
2. Sometimes you can just render the model directly, which will use the latter's `to_partial_path` method to locate the default partial.
3. The model shows up in the partial as a local variable named after the partial. For example, if your partial is called 'order_summary', and you pass an `Order` model as its object, the local variable is `order_summary`.
4. In the partial, decorate the model however/if you like. I wrote a `decorate` (global) view helper for this.
5. The decorated model now has all your handy view-related methods that you can use in the partial to do the rendering. As an example, my spiffy `OrderDecorator` has a `total` method which returns the total value of the order as a currency formatted string (pounds and pence). The original model only has the value in pence.

Specifically, I don't decorate the model until I'm in the partial. Doing it this way means that my partial and my decorator work together, rather than fighting each other.

You may end up with a number of similar decorators, each designed for a subtly different context (that is, partial). I used Rails concerns to extract common functionality. I found that most of my view logic and model manipulation code goes into those modules. Sehr gut.

This is just another way of trying to manage the inevitable complexity that builds up quickly in the view layer if you don't watch it.
