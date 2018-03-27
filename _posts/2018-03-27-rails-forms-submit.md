---
layout: post
title:  "How Rails disables your submit buttons by default"
date:   2018-03-27 11:53 +0100
categories: Rails simple_form UJS
---

By default Rails (5.1 for sure but probably earlier versions, too) will add a `data-disable-with` attribute to your form's submit button.

The idea is that, while your form is being submitted, the button's text will change to whatever Rails decides the "submit in progress" text should be.

Mostly, this default behaviour works fine, but sometimes it leads to some odd wording when your users submit the form. They may think there's something wrong when everything is working fine.

## Rails form helpers

You can change the text that is displayed by adding a `disable_with` option, shown here with simple_form in a slim template

```
= f.button :submit, value: 'Log in', data: { disable_with: 'Working...' }
```

Setting it to false will prevent any change, preserving the original button text, like this

```
= f.button :submit, value: 'Log in', data: { disable_with: false }
```

You can set the default to do nothing, forcing you to add `disable_with` explicitly if you want the text to change. The application-level config for this is

```
# Prevent Rails helpers from adding disable_with text by default
config.action_view.automatically_disable_submit_tag = false
```

Reference is [here](http://edgeguides.rubyonrails.org/configuring.html).

## The UJS driver

Over in js land, the Rails UJS driver (jQuery version shown here) swaps the text on demand, like this.

```
  disableFormElement: function(element) {
    var method, replacement;

    method = element.is('button') ? 'html' : 'val';
    replacement = element.data('disable-with');

    if (replacement !== undefined) {
      element.data('ujs:enable-with', element[method]());
      element[method](replacement);
    }

    element.prop('disabled', true);
    element.data('ujs:disabled', true);
  },
```

Note how the driver adds a `disabled` attribute, allowing you to style the button to show a submit is in progress, while at the same time preventing the user from clicking again and resubmitting the form by mistake.

To get this code to run, you need a `data-disable-with` attribute in the first place. So if you've overridden the default -- as shown above -- then you must explicitly add a data-disable-with value, otherwise the button won't be disabled.

## Conclusion

Perhaps the best bet is to leave the default as is, but add your own `data-disable-with` text if necessary (even if it's the same as the original) on a case-by-case basis.

If you're  going to write your own js to do some more sophisticated stuff, then you could adjust the default behaviour accordingly. I'm gradually moving away from the asset pipeline for javascript in favour of yarn/webpack in a Rails 5.1 stylee, so I may end up turning off the default "disable with" functionality in favour of my own hand-rolled solution. I'll keep you posted...

