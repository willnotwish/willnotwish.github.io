---
layout: post
title:  "Using auth0 in an existing Rails app"
date:   2016-10-21 10:21:00 +0100
categories: auth0 devise rails authentication
---

In my quest to develop smaller, simpler apps that do one thing well, rather than big, complicated apps that sometimes don't get finished, I'm experimenting with "outsourcing" user authentication.

Currently, I'm investigating [auth0][0].

[0]: https://auth0.com

I followed a few of auth0's excellent tutorials and got my authentication system up and running in no time. I event got auth0 to talk to my existing (legacy) database so it can import users on the fly. Great.

But I have a few questions/things I don't understand:

* What becomes of my User model in my app?
* Where is the user profile stored?
* Do I still need Devise?

In the app I'm migrating, I have separate `User` and `Profile` models. Here's the `users` schema:

```
  create_table "users", force: :cascade do |t|
    t.string   "email",                  limit: 255, default: "",               null: false
    t.string   "encrypted_password",     limit: 255, default: "",               null: false
    t.string   "reset_password_token",   limit: 255
    t.datetime "reset_password_sent_at"
    t.datetime "remember_created_at"
    t.integer  "sign_in_count",          limit: 4,   default: 0,                null: false
    t.datetime "current_sign_in_at"
    t.datetime "last_sign_in_at"
    t.string   "current_sign_in_ip",     limit: 255
    t.string   "last_sign_in_ip",        limit: 255
    t.string   "confirmation_token",     limit: 255
    t.datetime "confirmed_at"
    t.datetime "confirmation_sent_at"
    t.string   "unconfirmed_email",      limit: 255
    t.integer  "failed_attempts",        limit: 4,   default: 0,                null: false
    t.string   "unlock_token",           limit: 255
    t.datetime "locked_at"
    t.integer  "profile_id",             limit: 4
    t.datetime "created_at"
    t.datetime "updated_at"
    t.datetime "legacy_registered_at"
    t.string   "legacy_username",        limit: 255
    t.integer  "promotion_id",           limit: 4
  end

```

Most of these columns are to do with Devise's local authentication mechanism, so presumably they won't be needed with auth0. The two `legacy_` columns are left over from when I migrated users from a previous system; they won't be needed any more. Leave the `promotion_id` column for now as I'm not convinced it should have been included in the first place.

You can see that the User model `belongs_to :profile` from the `profile_id` key.

The `profiles` table is more application specific:

```
  create_table "profiles", force: :cascade do |t|
    t.string   "first_name",               limit: 255
    t.string   "last_name",                limit: 255
    t.string   "contact_number",           limit: 255
    t.string   "emergency_contact_name",   limit: 255
    t.string   "emergency_contact_number", limit: 255
    t.datetime "opted_in_at"
    t.date     "date_of_birth"
    t.string   "gender",                   limit: 255
    t.integer  "postal_address_id",        limit: 4
    t.datetime "created_at"
    t.datetime "updated_at"
    t.integer  "photo_id",                 limit: 4
    t.text     "medical_conditions",       limit: 65535
  end

```

As I understand it, auth0 identifies a user via a `user_id` attribute in its [normalized profile](https://auth0.com/docs/user-profile/normalized). auth0's docs say:

> user_id is the user's unique identifier. This is unique per Connection, but the same for all apps that authenticate via that Connection.
> 

What this means (I think) is that the `user_id` is different when a user authenticates via Facebook from when he/she does so via Twitter. But it's the same no matter which app they use. Hmm. We'll see.

To add some context in this case (the app is for my running club), I want to develop two (new) small apps: one to show race results and another to show the club's events calendar. A prospective member doesn't use either of those apps to join the club. Instead, they use the existing website (the "big app"). But once they've signed up and paid, they're entitled to use either of the smaller apps without having to create separate accounts.

Thinking about it some more, I still need a `User` Active Record model stored locally in my application database. My `User` model has several `has_many` relationships with other models. I'll need an additional `auth0_user_id` field though. I'll use this to identify which of my `Users` has just logged in when I get auth0's `user_id` following a successful authentication.

How does that work? I need to read auth0's documentation again [goes away...]

### The auth0 callback (*part 1*)

In the `callback` action of my `Auth0Controller` the sample code from auth0 shows the auth data (actually an `OmniAuth::AuthHash` instance) being stored in the session:

{% highlight ruby %}
class Auth0Controller < ApplicationController
  def callback
    # This stores all the user information that came from Auth0
    # and the IdP
    session[:userinfo] = request.env['omniauth.auth']

    # Redirect to the URL you want after successfull auth
    redirect_to '/dashboard'
  end

  def failure
    # show a failure page or redirect to an error page
    @error_msg = request.params['message']
  end
end
{% endhighlight %}

Within the auth data is auth0's `user_id`. Somewhere. Hmm. I debugged the action a bit more. It seems the data is a hash with the following keys:

{% highlight ruby %}
(byebug) oau.keys
["provider", "uid", "info", "credentials", "extra"]
{% endhighlight %}

Turns out that the `uid` is the `user_id`. Simple enough. The `info` key contains

{% highlight ruby %}
(byebug) oau['info']
#<OmniAuth::AuthHash::InfoHash email="swappoint.issues@gmail.com" first_name="Nick" image="https://lh3.googleusercontent.com/-YkyUDZPFebU/AAAAAAAAAAI/AAAAAAAAABo/5HX94Vdwxrs/photo.jpg" last_name="Adams" location="en-GB" name="Nick Adams" nickname="swappoint.issues">
{% endhighlight %}

Having been given the `user_id` I guess I now look up the corresponding `User` in my local database. If there isn't one, I'll assume that it's a new user registering, so I'll create a new record for them. Perhaps. Read on...

I won't worry about the profile data for now.

Or maybe I will.

### Model changes

In the existing system, a new user provides **all** the information the club needs (including, for example, an emergency contact number) **at the time of sign up**. There is no notion that a new user may sign up just by providing credentials such as email and password, and then provide profile information later. To enforce this restriction in the existing system, the `User` model requires a `Profile` to be provided at creation time.

The `Profile` model includes "presence" validations for many of its attributes which are mandatory (that'll be most of them, then). I can't expect auth0's `lock` widget to know about all the profile attributes needed by my application. If I am to use said widget for logins and sign ups (which I want to do, as it looks great), I'll need a new user to complete their profile some time after their initial sign up.

It makes sense to keep the validations on the `Profile` (since that's not going to change, I suspect), and remove the presence validator on the `User` model so an associated `Profile` is no longer mandatory when creating a new user in the auth0 callback. The profile will be added some time later.

This is an important change in my application logic.

In this particular case, a new user registering online with the club is not allowed to run with us until they have provided the information that the `Profile` requires (*e.g.*, emergency contact info). As previously, they'll have to complete their profile before they turn up to run. The difference is that now it's a two-step process; there could be some time between them signing up and completing their profile.

It's a bit like saying that a new user can't use a system unless they provide billing details (*e.g.*, a credit card). That's a common enough scenario.

Maybe this new way of doing things will encourage me to think of ways in which a system can be used by a user signing up with a bare minimum of profile information. That will help my systems pick up users.

A downside to this approach is that I will have users in my system with no profile. The original system was not designed for that. Some code may blindly assume that a profile exists when it doesn't. I will have to watch out for that. It won't be an issue for any future systems though.

### The auth0 callback (*part 2*)

Turns out that if no match is found when using `uid` as the key, I need to check for a matching email instead. That's needed to identify an existing user who has no account with auth0. Some identity providers (notably Twitter) don't provide an email address though. I'm not sure how to handle that use case right now.

Let's assume that I have identified the local `User` from auth0's provided data. So how do I tell Devise that the user is authenticated?
