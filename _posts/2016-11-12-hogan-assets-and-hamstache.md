---
layout: post
title:  "Hogan assets, hamstache and sprockets-rails"
date:   2016-11-11 18:21:00 +0000
categories: Hogan rails hamstache sprockets
---

I'm getting a little tired of it taking such a long time to get set up with hamstache these days. For a few years now I've used the `hogan_assets` [gem][1]. It has served me well.

But it doesn't seem to work with `sprockets-rails` 3, which is what Rails 4 (later versions, I think) and Rails 5 uses.

The symptom is that `HoganAssets` is undefined. This js object is created when a `.hamstache` file is compiled by Hogan into javascript.

On my last few projects I've wasted at least three hours trying to find out what's wrong.

Turns out that `.hamstache` files are not being compiled, so `HoganAssets` is not being defined. If you check your javascript sources in the browser, you won't find any of your templates. It's as if the whole `templates` directory is silently ignored.

Why is this happening? I don't understand it fully, but I think it's down to the template processor (`tilt` in this case) not being registered with sprockets. Ho hum.

I don't want to get sidetracked. For once.

The quick fix - for now - is to lock `sprockets-rails` at one of the 2.x versions in my `Gemfile`, like this:

`gem 'sprockets-rails', '2.3.3'`

It works, but really it isn't the right thing to do. When the server starts up, I see

```
DEPRECATION WARNING: Sprockets method `register_engine` is deprecated.
Please register a mime type using `register_mime_type` then
use `register_compressor` or `register_transformer`.
https://github.com/rails/sprockets/blob/master/guides/extending_sprockets.md#supporting-all-versions-of-sprockets-in-processors
 (called from block (2 levels) in <class:Engine> at /Users/nick/dev/rails5/raw/vendor/bundle/gems/hogan_assets-1.6.0/lib/hogan_assets/engine.rb:7)
DEPRECATION WARNING: Sprockets method `register_engine` is deprecated.
Please register a mime type using `register_mime_type` then
use `register_compressor` or `register_transformer`.
https://github.com/rails/sprockets/blob/master/guides/extending_sprockets.md#supporting-all-versions-of-sprockets-in-processors
 (called from block (2 levels) in <class:Engine> at /Users/nick/dev/rails5/raw/vendor/bundle/gems/hogan_assets-1.6.0/lib/hogan_assets/engine.rb:7)

```

Trouble is that the `hogan_assets` gem is no longer actively maintained, or so it would seem.

Right now, I'm on a tight deadline do I can't investigate any alternatives. But I need to. Pronto.

[1]: https://github.com/leshill/hogan_assets
