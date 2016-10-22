---
layout: post
title:  "Dynamic data in a Jekyll site"
date:   2016-10-18 17:59:00 +0100
categories: static jekyll calendar event
---

I'm getting on OK with Jekyll and I like its simplicity. But even a static site should be DRY. I thought I'd experiment with [Jekyll data files][1] to avoid repeating myself.

[1]: https://jekyllrb.com/docs/datafiles/ 

### The issue
I have a Rails app which is used by an admin to set up a calendar of recurring events. I also have a static, Jekyll-based site which needs to show a read-only view of that calendar.

I don't want the content author who's responsible for updating the Jekyll site to have to copy and paste from the Rails app in order to generate content. But at the same I want to give that author some freedom in the way that events are presented. Potentially, the target audience of the static site is different from that of the Rails site.

The important thing is for event dates and times to be consistent between the two calendars. Apart from that, they can look as different as the situation requires.

Think of the Rails app as the master, and the Jekyll app as the slave.

### The idea
Suppose the admin had the ability to download master calendar data as a JSON or CSV file. Perhaps that master data could be used as a base for generating the slave view.

### Generating JSON in the Rails app
This should be very easy. My views generally render HTML by default and JSON on request.

This URL works fine ```/admin/events```

Unfortunately I get an ```ActionController::UnknownFormat``` error when I try ```/admin/events.json```

Looks like I spoke too soon. Let's look at the controller...

Hmm. It looks a bit old school — probably the oldest bit of code in the whole app.

I added ```respond_to :html, :json``` to the top of the controller, and now the format is recognised, but there's an error in the ```/views/admin/events/index.json.jbuilder``` file. These days, I try to use [AMS](https://github.com/rails-api/active_model_serializers) but looks like I was using JBuilder when I wrote this stuff originally. I don't want to get sidetracked, so I'll just edit the builder file for now.

I got it working. Here's an excerpt from the downloaded file:

```
[ 
    {"name":"Monday night winter","date":"12-Dec-2016","time":"19:30","url":"http://localhost:3000/events/107"},
    {"name":"Tempo","date":"13-Dec-2016","time":"18:45","url":"http://localhost:3000/events/135"}
]
```
It's not exactly [JSON API](http://jsonapi.org/) but it's a start.

### Reading the data in Jekyll

According to the [docs][1], if I put a JSON file in the ```_data``` directory it's then available in Jekyll templates. If I name the file ```events.json``` then I should be able to access it as ```site.data.events``` I think. Let's try it.

If I write
{% raw %}
```
{% for event in site.data.events %}
  * {{event.name}} on {{event.date}} at {{event.time}}
{% endfor %}
```
{% endraw %}

I see this
{% for event in site.data.events %}
  * {{event.name}} on {{event.date}} at {{event.time}}
{% endfor %}

Очень хорошо. Looks like it's a go-er.
