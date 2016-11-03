---
layout: post
title:  "Inviting users to register"
date:   2016-11-01 12:31:00 +0000
categories: Auth0 invite rails authentication
---

I need a way for users *I* choose to sign up to my Rails app. My (simple) app should only be used by a handful of people. I don't want just anyone signing up for it. At least, anyone can sign up, but only those I choose can use it.

Previously I used the idea of an *invitation code*. I used to send a unique code (by email/sms/whatever) to someone, ask them to sign up and to enter the code as they did so.

The sign up screen has an input box for the code; you couldn't sign up unless you have one. Each code was unique.

This has worked well in the past for me when using Devise.

But now I'm using Auth0+Warden. Signup using a social login (Facebook, Google, etc.) is trivial for the user.

What I want now is for a freshly signed-up user to enter their invitation code before thay can do anything.

## Models

I have a `User` model, and a `Profile` model. I need the invitation code. What options do I have?

I think I need an `Invitation` model. It should have a unique `code`. This is the code I give to the user to enter when (correctly, immediately after) they register. It also needs a flag to say whether it's been used or not. (I could use a timestamp for that.) An invitation code cannot be used twice. I think it should have a `belongs_to` relationship with a `User`. When the user is set (*i.e.*, a foreign key exists) the invitation is said to be `accepted`.

To keep things RESTful, I'll write an `InvitationAcceptance` model (not database backed, just ActiveModel compliant: it quacks like ActiveRecord) which encapsulates the `User` and the invitation code. I'll have a corresponding `InvitationAcceptanceController` which "saves" the `InvitationAcceptance` model by marking the invitation as accepted.

There's another scenario to condier: when an admin sets a user up, and then invites them to sign up. That's covered in my next post.

[0]: https://Auth0.com
[1]: https://github.com/auth0/omniauth-auth0
[2]: https://auth0.com/docs/libraries/lock
[3]: https://github.com/hassox/warden
[4]: https://github.com/hassox/warden/wiki/Strategies
[5]: {% link _posts/2016-10-21-migrating-to-auth0.md %}
