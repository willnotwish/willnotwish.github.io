---
layout: post
title:  "Integrating Stripe in a Rails app"
date:   2016-11-09 10:52:00 +0000
categories: Stripe Auth0 rails
---

I need to integrate Stripe. I've done it before, but I can't remember how as it was some time ago.

## Requirements

I need event attendees to be able to buy tickets. I want to use Stripe for this. I want to see if I can get Stripe and Auth0 working together.

The flow:

* A user logs in or signs up
* They choose their ticket (just one for now)
* They check out using Stripe
* They receive a confirmation email

## Implementation

The [Stripe checkout guide][0] talks about the user giving their email address. I may already know that from the sign up. If not, checkout is a good place to ask for it. No-one expects to buy anything online without providing an email address.

Let's start with what I think will be the quickest way, using [Stripe's checkout widget][1].

Stripe provide a [Rails-specific guide][2]. Excelente, except it's a bit rubbish:

{% highlight ruby %}
def new
end

def create
  # Amount in cents
  @amount = 500

  customer = Stripe::Customer.create(
    :email => params[:stripeEmail],
    :source  => params[:stripeToken]
  )

  charge = Stripe::Charge.create(
    :customer    => customer.id,
    :amount      => @amount,
    :description => 'Rails Stripe customer',
    :currency    => 'usd'
  )

rescue Stripe::CardError => e
  flash[:error] = e.message
  redirect_to new_charge_path
end
{% endhighlight %}

The `new` method is OK (!), but `create` looks a bit odd. I think I'd better wrap the Stripe `charge`. And the Stripe `customer`, come to that.

Looking at the Stripe gem's [source code][3], I see that the [Stripe Customer class][4] contains the necessary references to the charges. So maybe storing the `charge` is unnecessary. Instead, how about just the `customer`?

Let's try adding `has_one :customer` to my `User` class. Or maybe that's too complex. Is there any way I could store the Stripe `customer` reference in my `User`?

Looks like a string `id` is generated when the `Stripe::Customer` object is created. This id can be used to retrieve said `Customer`, so let's add it to the User modle directly for now. Maybe some more indirection would be required in a production app, but this is just to get something going.

`Nicks-iMac:raw nick$ bin/rails g migration add_stripe_customer_id_to_users stripe_customer_id:string:index`



[0]: https://stripe.com/docs/checkout
[1]: https://stripe.com/checkout
[2]: https://stripe.com/docs/checkout/rails
[3]: https://github.com/stripe/stripe-ruby
[4]: https://github.com/stripe/stripe-ruby/blob/master/lib/stripe/customer.rb
[5]: 