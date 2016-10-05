---
layout: post
title:  "S3 storage and Rails applications"
date:   2016-10-04 14:47:00 +0100
categories: s3 aws rails
---

Using S3 in a Rails app is easy enough. Take user profile photos as an example. I've used the [Paperclip gem](https://github.com/thoughtbot/paperclip) with [Fog storage](http://fog.io/storage/) on several projects, and it works fine. 

But when I tried to get two versions of an app going at the same time, things got complicated.

My client had an existing Rails app that just about worked, but it was creaking at the seams; it needed a rewrite. Its library versions were way out of date:
it used Rails 2, and I'd quoted to update it to Rails 4. 

Was that a mistake? Should I have just chucked the old version away and started again? Maybe. But that's another story.

Here's what I found when it came to the seemingly simple concept of running the old (legacy) version alongside the new version when it came to photo storage.

### Background

Photo image files of users are stored in AWS S3 cloud storage.
A user can upload many photos, each of which is stored in S3 in its original form, together with a number of resized, processed versions, 
including a 100x100px thumbnail and a 300x340px profile image stored as a jpeg.

In the legacy system, an AWS IAM user was originally created which had full access to the production bucket. Uploaded images &mdash; together with their resized versions  &mdash; were stored under directory names that correspond to the database ID of the corresponding record in the photos table. This is standard practice.

In the new system, a similar arrangement is used, but with a different bucket (see next section).

### Bucket naming

In order to allow the two systems (legacy & new) to coexist during testing and commissioning, 
it is necessary to create a second bucket, and to keep the two completely separate. 
This eliminates any possibility of users being registered on the new system with the same photo IDs as existing users on the legacy system.
If this were to occur, users could end up with the wrong photos, leading to all sorts of confusion.

Migrating to a new bucket gave me the chance to use dashes instead of underscores in the bucket name.
This follows the current naming convention recommended by AWS to allow faster lookup by virtual host.

### Bucket access

For a while at least, the new system will require read-only access to the legacy system’s user photos.

What about users on the new system who have been migrated from the old? Initially, they won't have profile photos. This was OK to start with, but now the client is asking why.

Rather than copy all users’ photos from the old bucket to the new (which would bring with it all the old system’s database IDs in the path names,
as well as many redundant images for users who may be archived), it is preferable to copy photos between buckets as required&mdash;migration on demand, if you will. 

To achieve this, the new system requires a new IAM user with full access to the new bucket. It will also require read-only access to the legacy bucket, 
in order to copy selected files on demand.

Applying an AWS policy to enforce read-only access to the legacy bucket removes any possibility that new system can modify legacy photos and hence interfere with the legacy system, 
while at the same time allowing the new system to make copies of legacy photos on demand.

### An AWS S3 policy to enforce copy-only access
Suppose we have two users: `Alice` and `Bob`, and two buckets: `B1` and `B2`.

Alice requires full access to `B1` and no access at all to `B2`, while Bob requires “copy access” to `B1` and full access to `B2`.

By “copy access”, we mean the ability to copy files from `B1` to `B2`. The [Fog aws gem](https://github.com/fog/fog-aws) uses this [AWS API](http://docs.aws.amazon.com/AmazonS3/latest/API/RESTObjectCOPY.html)

> This implementation of the PUT operation creates a copy of an object that is already stored in Amazon S3. A PUT copy operation is the same as performing a GET and then a PUT. Adding the request header, x-amz-copy-source, makes the PUT operation copy the source object into the destination bucket.
>  

At the moment, I am not sure which permissions are required for this to work. I experimented with the [AWS CLI](https://aws.amazon.com/cli/) (a command line tool) to find out.

I set up the two users:

`Nicks-iMac:staff nick$ aws configure --profile alice`

`Nicks-iMac:staff nick$ aws configure --profile bob`

with no permissions at all. So

    Nicks-iMac:staff nick$ aws s3 ls --profile bob

gave

`A client error (AccessDenied) occurred when calling the ListBuckets operation: Access Denied`

As a sanity check I (temporarily) gave Alice full access `AmazonS3FullAccess` using the AWS console in the IAM editor. That worked and I could see all the S3 buckets in my account.

Permissions are granted using by attaching what AWS calls a "managed policy" to the user in question. There is also an "inline policy" but I don't know what that does yet. AWS says you can have up to 10 managed policies.

When I attached `AmazonS3ReadOnlyAccess` to Bob and did

    Nicks-iMac:staff nick$ aws s3 ls --profile bob

I also saw the list of all buckets. So the basics of attaching managed policies seems to work OK. 

How do I get more specific?

#### AWS S3 policies

`AmazonS3ReadOnlyAccess` is shorthand for this

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:Get*",
        "s3:List*"
      ],
      "Resource": "*"
    }
  ]
}
```
I think the `Resource` is the bucket (or, in this case, all buckets, judging by the *). As a test, I generated my own using AWS's policy generator:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1475607142000",
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::b1.nickadams.co.uk"
            ]
        }
    ]
}
```
and attached it to user Bob. This showed up as in Inline Policy in the IAM console (so now I guess I know the difference between a managed policy and an inline one): ![New inline policy in the IAM console](/images/bob-in-aws-console.png)

In plain English, then, this policy says:

*Allow* (the user to whom this policy is attached) to perform the action *s3:ListBucket* on the bucket *b1.nickadams.co.uk*.

Right, so what actions do I need to include to get the copy working?

AWS says that a copy is a GET followed by a PUT. So I would expect to have to allow read access to everything in `b1`. Let's investigate...

AWS has a [walkthrough](http://docs.aws.amazon.com/AmazonS3/latest/dev/example-walkthroughs-managing-access-example1.html) which illustrates a somewhat similar scenario. I also found a [list of actions](http://docs.aws.amazon.com/AmazonS3/latest/dev/using-with-s3-actions.html). I think Bob will need `s3:GetObject` on `b1` and `s3:GetObject` and `s3:PutObject` on `b2`. **The important thing is that Bob must *not* be able to write to b1; that's Alice's job.**

I applied this to Bob (*WARNING* this doesn't work - see below):

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::b1.nickadams.co.uk"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::b2.nickadams.co.uk"
      ]
    }
  ]
}
```

Then I did

`Nicks-iMac:staff nick$ aws s3 ls b1.nickadams.co.uk --profile bob`

and I got

`PRE photos/`

I think this shows me that there's a key (PREfix) "photos" there. Don't worry about it for now.

To upload a sample photo I tried the [S3 cp command](http://docs.aws.amazon.com/cli/latest/reference/s3/cp.html), but it didn't work. I did:

`aws s3 cp ak47.jpg s3://b2.nickadams.co.uk/test1.jpg --profile bob`

and I got

`upload failed: ./ak47.jpg to s3://b2.nickadams.co.uk/test1.jpg A client error (AccessDenied) occurred when calling the PutObject operation: Access Denied`

Pants. After much cursing, experimentation and Googling I saw an example which specified the Resource like this

```
"Resource": [
        "arn:aws:s3:::b2.nickadams.co.uk",
        "arn:aws:s3:::b2.nickadams.co.uk/*"
      ]
```
So when I did 

`Nicks-iMac:tmp nick$ aws s3 cp ak47.jpg s3://b2.nickadams.co.uk/1/test.jpg --profile bob` 

I got 

`upload: ./ak47.jpg to s3://b2.nickadams.co.uk/1/test.jpg`

which looked promising. Now for the [acid test](http://www.phrases.org.uk/meanings/acid-test.html). Bob should not be able to upload to `b2`:

`aws s3 cp ak47.jpg s3://b1.nickadams.co.uk/1/test.jpg --profile bob` gave the access denied error. Success!

What I wanted was for Bob to be able to copy from `b1` to `b2` on demand. So, I uploaded a file to `b1` (which Alice *can* do) and checked that Bob could see it:

```
aws s3 ls s3://b1.nickadams.co.uk/1/test.jpg
2016-10-05 15:40:27      66967 test.jpg
```

Then (as Bob) I copied it&mdash;renaming as I went&mdash;from `b1` to `b2`:

```
Nicks-iMac:tmp nick$ aws s3 cp s3://b1.nickadams.co.uk/1/test.jpg s3://b2.nickadams.co.uk/000/000/001/migrated-test.jpg --profile bob
copy: s3://b1.nickadams.co.uk/1/test.jpg to s3://b2.nickadams.co.uk/000/000/001/migrated-test.jpg
```
I checked it was there:

```
Nicks-iMac:tmp nick$ aws s3 ls s3://b2.nickadams.co.uk/000/000/001/migrated-test.jpg
2016-10-05 15:44:11      66967 migrated-test.jpg
```

### Conclusion
The [policy I eventually applied to Bob](/aws-policies/bob.json) gave him read-write access to bucket `b2` and read-only access to `b1`, as required.















