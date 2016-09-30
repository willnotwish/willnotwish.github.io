---
layout: post
title:  "Active record scopes, STI and included modules"
date:   2016-09-30 11:48:00 +0100
categories: activerecord modules scopes
---

When I started to include modules in my models recently to help me get organised, I found that AR's STI complicated my carefully crafted class methods (aka *scopes*).

### The problem

For my running club's website, I needed an **Event** base class to represent "something" that takes (or took) place at a given time on a given date. I used STI to define a few events such as Social Event, a Christmas Party, a Training Run and a Race, all of which inherit from the base class Event.
I intend to create more of these derived classes over  time.

<!-- The events table has a type column, as required by STI. -->

At my club, sometimes a run needs back markers: people who run at the back to make sure no-one gets lost. We have a "back marking rota" to allocate this task to different people. Not all events have back markers, only those that involve lots of people running. To handle this, I wrote a model concern called BackMarkable, and included it in those classes which modelled events which needed back marking, such as our popular Monday night runs. To allow an admin to create a back marking rota, I needed a list of events that were back markable. For a long time, I couldn't figure out how to write a suitable class method[^1]. 

But then I had a light bulb moment &mdash; what I really needed was a class method which would return the events from the events table whose (derived) classes included the BackMarkable concern.

The *type* field is part of AR's STI implementation: it uses it to track the (derived) class of the Event. I can grab Events of a given class by adding a WHERE clause easily enough. In fact, this is how AR does it.

{% highlight ruby %}
irb(main):037:0> Events::TrainingRun.all.to_sql
=> "SELECT `events`.* FROM `events` WHERE `events`.`type` IN ('Events::TrainingRun')"
{% endhighlight %}

But other derived classes also included the BackMarkable module; I wanted those as well. I wanted *all* events that were back markable.

### The solution

It goes like this:

1. Give me all the classes derived from Event
2. Filter these to give a list containing those that include the BackMarkable concern
3. Give me all matching events

For step 1, I used the [descendants](http://edgeguides.rubyonrails.org/active_support_core_extensions.html#subclasses-descendants) method from Rails' ActiveSupport.

{% highlight ruby %}
irb(main):041:0> Event.descendants.map(&:name)
=> ["Events::Race", "Events::SocialEvent", "Events::TrainingSession", "Events::TrainingRun"]
{% endhighlight %}

I applied the filter in step 2:

{% highlight ruby %}
irb(main):045:0> Event.descendants.select {|k| k.included_modules.include?(BackMarkable) }.map( &:name )
=> ["Events::TrainingRun"]
{% endhighlight %}

and wrote the query (3):

{% highlight ruby %}
irb(main):054:0> Event.where( type: Event.descendants.select {|k| k.included_modules.include?(BackMarkable) }.map( &:name ) ).to_sql
=> "SELECT `events`.* FROM `events` WHERE `events`.`type` = 'Events::TrainingRun'"
{% endhighlight %}

This still gives only TrainingRuns, because they are the only events which include BackMarkable. Later, when I want to add a new type of event which needs back marking, it will be included in the query without me having to remember to add it explicitly to the WHERE clause. Here's an example:

{% highlight ruby %}
irb(main):055:0> class Foo < Event; include BackMarkable; end
=> Foo(id: integer, starts_at: datetime, duration: integer, created_at: datetime, updated_at: datetime, location_id: integer, route_id: integer, type: string, template_id: integer, name: string, aasm_state: string, description: text, precedence: integer, distance_id: integer, race_name_id: integer)
irb(main):056:0> Foo.included_modules.include? BackMarkable
=> true
irb(main):057:0> Event.where( type: Event.descendants.select {|k| k.included_modules.include?(BackMarkable) }.map( &:name ) ).to_sql
=> "SELECT `events`.* FROM `events` WHERE `events`.`type` IN ('Foo', 'Events::TrainingRun')"
{% endhighlight %}

Using a WHERE clause ensures that the scope is chainable, so I can apply other scopes as well. Suppose I want all future back markable events for the back marking rota:

{% highlight ruby %}
irb(main):058:0> Event.where( type: Event.descendants.select {|k| k.included_modules.include?(BackMarkable) }.map( &:name ) ).future.to_sql
=> "SELECT `events`.* FROM `events` WHERE `events`.`type` IN ('Foo', 'Events::TrainingRun') AND (`events`.`starts_at` > '2016-09-30 09:57:26')"
{% endhighlight %}

[^1]: At one point I contemplated adding a boolean flag to the events table, in order to indicate whether an event was back markable or not. But this felt wrong.





<!-- You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

To add new posts, simply add a file in the `_posts` directory that follows the convention `YYYY-MM-DD-name-of-post.ext` and includes the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: http://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
 -->