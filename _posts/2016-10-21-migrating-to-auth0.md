---
layout: post
title:  "Using Auth0 in an existing Rails app"
date:   2016-10-21 10:21:00 +0100
categories: Auth0 devise rails authentication
---

In my quest to develop smaller, simpler apps that do one thing well, rather than big, complicated apps that do their best but sometimes don't get finished (by me, I mean), I'm experimenting with "outsourcing" user authentication.

Currently, I'm investigating [Auth0][0].


I followed a few of Auth0's excellent tutorials and got my authentication system up and running in no time. I even got Auth0 to talk to my existing (legacy) database so it can import users on the fly. Great. **Update** Hmm. I've since found out that this facility is a paid feature so I may not be using it. We'll see.

I have a few questions/things I don't understand:

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

Most of these columns are to do with Devise's local authentication mechanism, so presumably they won't be needed with Auth0. The two `legacy_` columns are left over from when I migrated users from a previous system; they won't be needed any more. Leave the `promotion_id` column for now as I'm not convinced it should have been included in the first place.

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

As I understand it, Auth0 identifies a user via a `user_id` attribute in its [normalized profile](https://Auth0.com/docs/user-profile/normalized). Auth0's docs say:

> user_id is the user's unique identifier. This is unique per Connection, but the same for all apps that authenticate via that Connection.
> 

What this means (I think) is that the `user_id` is different when a user authenticates via Facebook from when he/she does so via Twitter. But it's the same no matter which app they use. Hmm. We'll see.

To add some context in this case (the app is for my running club), I want to develop two (new) small apps: one to show race results and another to show the club's events calendar. A prospective member doesn't use either of those apps to join the club. Instead, they use the existing website (the "big app"). But once they've signed up and paid, they're entitled to use either of the smaller apps without having to create separate accounts.

Thinking about it some more, I still need a `User` Active Record model stored locally in my application database. My `User` model has several `has_many` relationships with other models. I'll need an additional `Auth0_user_id` field though. I'll use this to identify which of my `Users` has just logged in when I get Auth0's `user_id` following a successful authentication.

How does that work? I need to read Auth0's documentation again [goes away...]

### The Auth0 callback — part 1

In the `callback` action of my `Auth0Controller` the sample code from Auth0 shows the auth data (actually an `OmniAuth::AuthHash` instance) being stored in the session:

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

Within the auth data is Auth0's `user_id`. Somewhere. Hmm. I debugged the action a bit more. It seems the data is a hash with the following keys:

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

The `Profile` model includes "presence" validations for many of its attributes which are mandatory (that'll be most of them, then). I can't expect Auth0's `lock` widget to know about all the profile attributes needed by my application. If I am to use said widget for logins and sign ups (which I want to do, as it looks great), I'll need a new user to complete their profile some time after their initial sign up.

It makes sense to keep the validations on the `Profile` (since that's not going to change, I suspect), and remove the presence validator on the `User` model so an associated `Profile` is no longer mandatory when creating a new user in the Auth0 callback. The profile will be added some time later.

This is an important change in my application logic.

In this particular case, a new user registering online with the club is not allowed to run with us until they have provided the information that the `Profile` requires (*e.g.*, emergency contact info). As previously, they'll have to complete their profile before they turn up to run. The difference is that now it's a two-step process; there could be some time between them signing up and completing their profile.

It's a bit like saying that a new user can't use a system unless they provide billing details (*e.g.*, a credit card). That's a common enough scenario.

Maybe this new way of doing things will encourage me to think of ways in which a system can be used by a user signing up with a bare minimum of profile information. That will help my systems pick up users.

A downside to this approach is that I will have users in my system with no profile. The original system was not designed for that. Some code may blindly assume that a profile exists when it doesn't. I will have to watch out for that. It won't be an issue for any future systems though.

### The Auth0 callback — part 2

Turns out that if no match is found when using `uid` as the key, I need to check for a matching email instead. That's needed to identify an existing user who has no account with Auth0. Some identity providers (notably Twitter) don't provide an email address though. I'm not sure how to handle that use case right now.

### Aside — a quick recap
After a frustrating day trying to make sense of the relationships between oauth2 providers, omniauth, Warden, Devise, Rails and Auth0, I awoke early this morning to a rare moment of clarity. Here's my current understanding:

The combination of [Auth0's lock widget][2] and [Omniauth gem][1] calls back my Rails application in two ways:

1. Lock instructs Auth0 to call back your server at the URL you specify when creating it. It also emits an *authenticated* event (in js). So I can interact with Lock both on the client (Javascript) and the server (Rails).
2. To use Lock in a Rails app you also need the [OmniAuth strategy gem for auth0][1]. Without it, Lock works OK, and your Rails app will be called back, but no omniauth data will be included in the request. This is pretty obvious when you think about it. Hmph.
3. The URL of the Rails callback is also needed in the oauth2 initializer. My usual way to stay DRY doing things like this is to store the necessary constant values in Ruby code somewhere (secrets.yml is a possible place) and then refer to them as data attributes in the HTML. They can be picked up from the DOM by external JS. Pretty standard stuff.
4. Devise is based on [Warden][3]. A [Warden strategy][3] is one way of indicating to Devise whether a user is logged in or not. There's a world of difference between a Warden strategy and an omniauth strategy. Don't get confused. As I write this paragraph, I haven't yet figured out how to write a Warden strategy, or even if I need one. That's my next task.

### Devise — still
I'm not sure I still need Devise. I think all it's doing for me now is providing helpers like `current_user` and allowing me to protect my controllers with `authenticate_user!`. Time will tell, but, at the moment, I need a way to tell whether a user is logged in or not.

Let's assume that I have identified the local `User` from Auth0's provided data. How do I tell Devise that the user is authenticated?

I think I need a [Warden strategy][4]. What is the simplest I can think of? Maybe to store the `id_token` in the session and just check that. How does Devise do it for `database_authenticable` models?

In Devise's [database_authenticable strategy][5], we find:

{% highlight ruby %}
      def authenticate!
        resource  = password.present? && mapping.to.find_for_database_authentication(authentication_hash)
        hashed = false

        if validate(resource){ hashed = true; resource.valid_password?(password) }
          remember_me(resource)
          resource.after_database_authentication
          success!(resource)
        end

        mapping.to.new.password = password if !hashed && Devise.paranoid
        fail(:not_found_in_database) unless resource
      end
{% endhighlight %}

The password comparison is done in `validate`, which calls the model's `valid_password?` method to check for a match.

If `validate` succeeds then `success!` is called, otherwise `fail`. It would appear that what I need to do, then, is to check for an `id_token` in the session and call `success!` if it's present and correct.

### Debugging my Warden strategy
I wrote a strategy which decodes the `id_token` using `JWT`'s `decode` method. This worked, and the `User` is found in my `users` table (I moved the functionality for this from the Auth0 callback to the Warden strategy in the end). But now I have an infinte loop of redirects! What the hell!?

Let's review what (I think) is supposed to happen.

First, with no user logged in, the header partial in the Rails layout calls the Devise helper `user_signed_in?`.

Looking at [Devise's helpers](https://github.com/plataformatec/devise/blob/master/lib/devise/controllers/helpers.rb) I see that `user_signed_in?` wraps a call to `current_user`. The latter calls Warden's `authenticate` method (*without* a bang `!` on the end). The method *I* provided is `authenticate!` (*with* a bang). `authenticate` is not in Warden's [base strategy](https://github.com/hassox/warden/blob/master/lib/warden/strategies/base.rb). Where is it? I couldn't find it in Devise anywhere...

Turns out `authenticate` is in the [Warden proxy](https://github.com/hassox/warden/blob/master/lib/warden/proxy.rb). According to the documentation:

{% highlight ruby %}
    # Run the authentication strategies for the given strategies.
    # If there is already a user logged in for a given scope, the strategies are not run
    # This does not halt the flow of control and is a passive attempt to authenticate only
    # When scope is not specified, the default_scope is assumed.
    #
    # Parameters:
    #   args - a list of symbols (labels) that name the strategies to attempt
    #   opts - an options hash that contains the :scope of the user to check
    #
    # Example:
    #   env['warden'].authenticate(:password, :basic, :scope => :sudo)
    #
    # :api: public
    def authenticate(*args)
      user, _opts = _perform_authentication(*args)
      user
    end
{% endhighlight %}

Ahah. In the Warden proxy there is also an `authenticate!` method which throws `:warden` in order to redirect the user to a login page. Or something. Try this:

{% highlight ruby %}
    # The same as +authenticate+ except on failure it will throw an :warden symbol causing the request to be halted
    # and rendered through the +failure_app+
    #
    # Example
    #   env['warden'].authenticate!(:password, :scope => :publisher) # throws if it cannot authenticate
    #
    # :api: public
    def authenticate!(*args)
      user, opts = _perform_authentication(*args)
      throw(:warden, opts) unless user
      user
    end
{% endhighlight %}

So... I would expect `authenticate` to be called when checking (passively) if a user is logged in, and `autheticate!` to be called to redirect the user to log in if they've not already done so. Cool-io.

But I'm still seeing my strategy being called over and over, with redirect after redirect.

Some time later...

Turns out that a Devise "hook" was logging me out by "throwing a warden" because the account wasn't confirmed. This is fair enough, but now I'm getting the nagging feeling that trying to integrate Devise and Auth0 is a dumb idea. Devise does what it does really well, but I don't need the bulk of its functionality. What do I need? See my next post.








[0]: https://Auth0.com
[1]: https://github.com/auth0/omniauth-auth0
[2]: https://auth0.com/docs/libraries/lock
[3]: https://github.com/hassox/warden
[4]: https://github.com/hassox/warden/wiki/Strategies
[5]: https://github.com/plataformatec/devise/blob/master/lib/devise/strategies/database_authenticatable.rb

