---
layout: post
title:  "How Rails disables your submit buttons by default"
date:   2018-03-27 11:53 +0100
categories: Rails simple_form UJS
---

By default Rails will add a data-disable-with attribute to your form's submit button.

The idea is that, while your form is being submitted, the button's text will change to whatever Rails decides the "submit in progress" text should be. 

Mostly, this default behaviour works fine, but sometimes it leads to some odd wording when your users submit the form. They may think there's something wrong when everythig is working fine.

You can change the text that is displayed by adding a `disable_with` option. Setting it to false will prevent any change, preserving the original button text.

If you prefer a little more control and a little less magic, then you can set the default to do nothing with

The main reference is here [https://certbot.eff.org/#ubuntuother-nginx]

```
      location ^~ /.well-known/acme-challenge/ {

        # Set correct content type. According to this:
        # https://community.letsencrypt.org/t/using-the-webroot-domain-verification-method/1445/29
        # Current specification requires "text/plain" or no content header at all.
        # It seems that "text/plain" is a safe option.
        default_type "text/plain";

        # This directory must be the same as in /etc/letsencrypt/cli.ini
        # as "webroot-path" parameter. Also don't forget to set "authenticator" parameter
        # there to "webroot".
        # Do NOT use alias, use root
        root         /var/www/letsencrypt;
      } 
```
