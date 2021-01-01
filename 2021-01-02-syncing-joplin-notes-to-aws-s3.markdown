---
layout: post
title: "How to synchronise Joplin notes to AWS S3"
date: 2021-01-02 01:00:00 +1000
#modified_date: 2020-12-29 17:00:00 +1000
author: Thomas
categories: aws joplin notes guide
---
AWS S3 support is currently in beta for Joplin but it seems to have all the functionality necessary and has been working very smoothly across multiple devices for me. I couldn't find any steps when I first set it up so here's a guide on how to set it up.

The pricing for S3 [is available here](https://aws.amazon.com/s3/pricing/).

*It's worth noting that if you're eligible for AWS free tier you will receive 5 GB of storage for free during your first 12 months.*

Make sure you have created an AWS account and added billing information before attempting to make an S3 bucket.

### Create your bucket

Once on the AWS dashboard at the top search for "S3" and go to the S3 dashboard for your region.

After that go to "Create bucket" and add a unique name for your bucket like `my-notes`. Make sure your region is set to your correct region. You will want to have "Block all public settings" ticked.
![AWS settings](/assets/img/syncing-joplin-notes-to-aws-s3/s3-config-1.png)

I would recommend turning off "Bucket Versioning" as Joplin has "Note History" built into the program which will allow you to easily recover your documents. You can turn on Bucket Versioning if you are worried about application failures with Joplin and syncing across devices, however just be careful about the added expenses. You can change this at any time.

I would recommend turning on "Default encryption" for your bucket also. Even if you have end-to-end-encryption enabled in Joplin it's always a good idea to enable it for the S3 bucket. It also doesn't cost any extra and any processing delays will be negligible.

![AWS settings 2](/assets/img/syncing-joplin-notes-to-aws-s3/s3-config-2.png)

### Generating credentials for Joplin

Once your bucket is created you must generate IAM credentials that have access to your bucket. Before you can do this however you must create a policy which restricts access to only your bucket.

#### Create a policy for Joplin bucket

On the AWS dashboard search for "IAM". You should get a screen similar to the one below. Once here click on "Customer managed policies".

![IAM menu](/assets/img/syncing-joplin-notes-to-aws-s3/iam-menu.png)

Once there click "Create policy" and complete the following steps in the "Visual editor":

1. Click "Service" and select "S3"
2. Under Actions
	1. Under "List" select "ListBucket"
	2. Under "Read" select "GetBucketLocation" and "GetObject"
	3. Under "Write" select "DeleteObject", "DeleteObjectVersion" and "PutObject"
3. Under Resources
	1. Select "Specific" as the type of resource
	2. Click "Add ARN" for bucket
	3. Type in your bucket name in the "Bucket name" field and click "Add"
	4. Click "Add ARN" for object
	5. Type in your bucket name in the "Bucket name" field like the previous step
	6. Tick the "Any" button under "Object name" and click "Add"

After adding the service and adding actions

![Create policy 1](/assets/img/syncing-joplin-notes-to-aws-s3/policy-menu-1.png)

The add ARN for bucket

![Create policy 2](/assets/img/syncing-joplin-notes-to-aws-s3/policy-menu-2.png)

The add ARN for object

![Create policy 3](/assets/img/syncing-joplin-notes-to-aws-s3/policy-menu-3.png)

Once these steps are completed click "Review policy", give it a name such as `joplin-my-notes-policy` and click "Create policy".

#### Creating your IAM user

Now with your policy you can finally generate credentials for your devices. I would recommend that each device should have their own set of credentials/user so you can revoke them easily and also not rely on one key.

To create a user search again for "IAM" and once opened select "Users" instead this time. Click "Add user" at the top and give it a name like `joplin-desktop` and tick "Programmatic access"

![IAM user 1](/assets/img/syncing-joplin-notes-to-aws-s3/iam-user-1.png)

From here you need to add the policy you just created to your user. Click "Attach existing policies directly" at the top then click "Filter policies" and select "Customer managed".

![IAM user 2](/assets/img/syncing-joplin-notes-to-aws-s3/iam-user-2.png)

Your custom policy from before should be listed and from there you can select it and click "Next" at the bottom. From here you can click "Next", "Review" and then "Create user". Once the user has been created your credentials (Access key ID and Secret access key) will be listed.

#### Adding it to Joplin

You should now have all the information you need for Joplin.

Open Joplin and click Tools -> Options and open the Synchronisation tab. From here fill it out like below.

![Joplin sync](/assets/img/syncing-joplin-notes-to-aws-s3/joplin-sync.png)

Now your Joplin should be syncing with your S3 bucket. After syncing you should be able to refresh your bucket and see all your Joplin files and notebooks there.
