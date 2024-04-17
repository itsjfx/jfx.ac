---
layout: post
title: How to synchronise Joplin notes to AWS S3
date: 2021-01-02 01:00:00 +1000
tags: [aws, joplin, notes, guide]
---

## Introduction

AWS S3 support is currently in beta for Joplin, but it seems to have all the
functionality necessary and has been working very smoothly across multiple
devices for me. I couldn't find any steps when I first set it up so here's a
guide on how to set it up.

The pricing for S3 [is available here](https://aws.amazon.com/s3/pricing/).

**NOTE: if you're eligible for AWS free tier you will receive 5 GB of storage
for free during your first 12 months**

Make sure you have created an AWS account and added billing information before
attempting to make an S3 bucket.

> 2024 update: This can be done with pretty easily with Cloudformation... but I
> didn't know it at the time. I've kept the "click ops" instructions cause this
> is a pretty simple setup.

## Steps

### Create your bucket

1. Open the AWS Console, make sure you're in the correct/closest region to you,
   then search and open S3.
2. Click "Create bucket" and add a unique name for your bucket. Make sure your
   region is set to your correct/closest region.
    * **Note: The S3 bucket name must be globally unique for the service, so you
      may need to think of something creative**
3. Make sure "Block all public settings" is ticked
4. I'd suggest enabling "Bucket Versioning" but for a short duration as Joplin
has "Note History" built into the program. This may save you later if you have a
sync issue, however I've not encountered one yet. Beware there's (small)
additional costs associated with doing this. You can change this at any time.
5. Enable "Default encryption" with SSE-S3. Even if you have
end-to-end-encryption enabled in Joplin it's a good idea to enable it for the S3
bucket. It also doesn't cost any extra and any processing delays will be
negligible.

### Generating credentials for Joplin

Once your bucket is created you must generate IAM credentials that have access
to your bucket. You must also create a policy which restricts access to only
your bucket.

#### Create a policy for Joplin bucket

1. On the AWS dashboard open the IAM dashboard
2. Click on "Customer managed policies"
3. Click "Create Policy", then do the following in the visual editor:
    1. Click "Service" and select "S3"
    2. Under Actions
       1. Under "List" select `ListBucket`
       2. Under "Read" select `GetBucketLocation` and `GetObject`
       3. Under "Write" select `DeleteObject`, `DeleteObjectVersion`, and
          `PutObject`
       4. Under Resources
          1. Select "Specific" as the type of resource
          2. Click "Add ARN" for bucket
          3. Type in your bucket name in the "Bucket name" field and click "Add"
          4. Click "Add ARN" for object
          5. Type in your bucket name in the "Bucket name" field like the
             previous
          6. Tick the "Any" button under "Object name" and click "Add"

Once these steps are completed click "Review policy", give it a name such as
`joplin-my-notes-policy` and click "Create policy".

#### Creating your IAM user

You can now generate credentials for your devices. I would recommend that each
device should have their own set of credentials/user so you can revoke them
easily and not rely on one key.

To create a user:
1. Search for "IAM" again and once opened select "Users"
2. Click "Add user" at the top and give it a name like `joplin-desktop` and tick
"Programmatic access"
3. Attach the policy you just created to your user. Click "Attach existing
policies directly" at the top then click "Filter policies" and select "Customer
managed".
4. Your custom policy from earlier should be listed, select it and click "Next"
5. Click through "Next", "Review" and then "Create user"
6. Once the user has been created your credentials (Access key ID and Secret
   access key) will be listed

### Setting up Joplin

You now have all the information you need for Joplin.

1. Open Joplin and click Tools -> Options
2. Open the Synchronisation tab
3. Type in your AWS access key ID and secret

Now your Joplin should be syncing with your bucket :)
