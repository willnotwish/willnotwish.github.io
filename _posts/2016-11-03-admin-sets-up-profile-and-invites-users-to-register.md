---
layout: post
title:  "Admins setting up users prior to sign up"
date:   2016-11-03 18:35:00 +0000
categories: Auth0 admin profile invite rails authentication
---

Sometimes an admin needs to "set up" a user before that user has signed up.

Suppose someone shows interest in a job. A recruitment company might be keen for that person to sign up to their recruitment system. During a phone call or interview, a recruitment agent (admin) might want to build a profile of the potential recruit. During or after the call the admin may send the user a link or a code by email or SMS, encouraging them to sign up.

When the user signs up via that link, or uses the code, their profile is already set up for them, based on what the admin entered.

## Implementation

It seems to me that this is a development of my previous [invitation code idea]({% link _posts/2016-11-01-inviting-users-to-register.md %}).
