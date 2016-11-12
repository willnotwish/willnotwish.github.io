---
layout: post
title:  "Restricting which products a user can buy"
date:   2016-11-11 18:21:00 +0000
categories: Stripe rails products access code
---

In a previous post I talked about how to integrate with Stripe. My experience was that the sample code Stripe provided for a Rails integration wasn't really up to the job. It wasn't RESTful, for a start.

Anyway, what I need now is to show a potential customer the range of products he or she could buy using a Stripe checkout. But I have a particular - and somewhat unusual - scenario in which I only want invited customers to be able to buy a limited range of products.

Suppose there are P products in total, but only Q of them are available to the general public. R are "restricted", in that a potential customer must enter a code (not a _discount_, rather an _invitation_ code) to be able to select an R-rated product.

First guess: generate a product resource with all the normal attributes (name, description, code, price in pence). The simplest restriction I can think of is to give a restricted product a restriction_code. A product is "restricted" if it has a non-blank restriction code. This may be a bit of an over simplification. Let's see how it goes.

Thinking about it come more, I can see that's not going to work. I need a certain group of potential customers to see different subsets, depending on the code they enter. I need a better design.

Try this.

Given an `Audience` (identified by a `code`), a related subset of products is displayed. A general audience (potential or existing customers with no code or an unrecognised code) sees a particular subset (the so-called "general" products). Invited potential customers (audience A, say), see the product A subset, whereas those with a B code see the B subset.

A product `has_many` audiences. An audience `has_many` products. An `AudienceProduct` is the join model. Because I may need to vary the price according to the audience, I'll add an optional `price_in_pence` attribute to the `AudienceProduct`.

A "general" audience is one with no code.

[0]: https://stripe.com/docs/checkout
[1]: https://stripe.com/checkout
[2]: https://stripe.com/docs/checkout/rails
[3]: https://github.com/stripe/stripe-ruby
[4]: https://github.com/stripe/stripe-ruby/blob/master/lib/stripe/customer.rb
[5]: 