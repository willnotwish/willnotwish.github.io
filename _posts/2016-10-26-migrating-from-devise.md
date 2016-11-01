---
layout: post
title:  "Migrating from Devise to Auth0"
date:   2016-10-26 18:51:00 +0100
categories: Auth0 devise rails authentication
---

In an earlier post I described my [initial experiences][5] in trying to get [Auth0][0] working as the authentication provider in an existing, traditional (full page refresh) Rails 4 application.

If you've not read that, maybe you should [now][5].

My conclusion was that I couldn't expect Auth0 and Devise to operate together seamlessly: their approaches are too different.

So, if I want to use Auth0, what Devise functionality do I still need?

Off the top of my head, in no particular order:

* `current_user`, `user_signed_in?` controller and view helpers
* `authenticate_user!` as a controller `before_filter`
* redirection to a login page if a resource is protected by `authenticate_user!` and subsequent redirection to that resource.

Let's think how these might be implemented.

I think I still need [Warden][3]. I have already written a [Warden strategy][4] of sorts to decode Auth0's JWT `id_token`. I still need to integrate Warden as Rack middleware (or maybe it's just the failure app I need).

I've made a start by creating a brand new Rails 5 application. I'll call it RAW (Rails, Auth0, Warden). I added the `warden` and `omniauth-auth0` gems. I created `PublicThing` and `PrivateThing` models. I want everyone (that's unauthenticated as well as authenticated users) to be able to see public things, but only authenticated users to have access to private things. I also want my own `users`, even though I'll be using Auth0 for authentication.

Straightaway, in my layout, I want a login link, if the user has not already logged in. That's what `user_signed_in?` is for.

## ...some time later

OK. So I've written some helpers, an `authenticate_user!` before filter and a Warden strategy. I've learnt loads and it seems to work. I even wrote a simple Warden failure app. Now I know a lot more about using Auth0.

One thing is now clear. There's no *technical* difference bewteen a social login and a social sign up. I mean: signing up is pretty trivial; it's so easy for the user. It's up to my app to gather the profile (or whatever) info it needs from the user to allow them to do whatever it is that the app is designed for.

My next task is to focus on inviting users: giving them an invitation code which entitles them to sign up.

Oh, and I didn't get as far as automatically redirecting to a target page after logging in. That's one for the future.

[0]: https://Auth0.com
[1]: https://github.com/auth0/omniauth-auth0
[2]: https://auth0.com/docs/libraries/lock
[3]: https://github.com/hassox/warden
[4]: https://github.com/hassox/warden/wiki/Strategies
[5]: {% link _posts/2016-10-21-migrating-to-auth0.md %}
