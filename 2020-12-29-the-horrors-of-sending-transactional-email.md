---
layout: post
title: "The horrors of sending transactional email... and why you should use Postmark"
date: 2020-12-29 17:00:00 +1000
#modified_date: 2020-12-29 17:00:00 +1000
author: Thomas
categories: email
---
I would like to begin this post by first stating that I have not been asked by [Postmark](https://postmarkapp.com) to write this article. I believe that [Postmark](https://postmarkapp.com) offers an excellent service and because of that they deserve my recognition.

As for what transactional email is, Postmark provides a good definition on their website:
> Transactional email is typically a unique, high-priority message sent to a single recipient. They are often triggered by something a user does or doesn’t do.

This is different to marketing/bulk/broadcast email:
> [Broadcast email] is sent to multiple recipients at once. Things like product update announcements or terms of service notices are examples of broadcast emails.

If a user does not receive their transactional email then this will most likely result in a bad user experience. For example if a user cannot verify their email on your website since they never receive a verification email and your website requires a users email to be verified before they can purchase a product, then that user will not be able to buy their product and as a result will most likely never attempt to use your website again due to their bad user experience. This example turned out to be a reality for me.

## A story about my nightmares and experience sending transactional email

Sending transactional email just sucks because there's a lot that can go wrong, and using an email delivery provider which is blacklisted by major email hosting providers is another thing to add to the list. At SteamLevels we were looking for an email sending provider which could reliably send emails to our users for two cases: verification emails (when users add an email to their account), as well as to send them updates about their account status (non promotional). We never sent any spam or promotional mail and our users may only ever receive 2 emails in their lifetime for their account. Several headaches were endured before we finally settled on a provider which has proven to offer this service with great success and reliability.

### Why not just move to a dedicated IP?

When we first started having issues with email delivery this was the first idea we had to solve the issue: if other people are ruining our reputation due to us sharing an IP with them, why not just move to our own IP? We quickly discovered that due to the low volume of emails we send (we send ~5k a month now and back then it was even less) moving to a dedicated IP would create a new set of issues. This is due to IP reputation which is a core part of email delivery, Postmark have [a good blog post](https://postmarkapp.com/blog/the-false-promises-of-dedicated-ips) on the subject where they discuss dedicated IPs better than I could. Because of the low volume of emails we send, we were forced to continue looking for an email provider which had reputable shared IPs.

### Mailgun

We began by using [Mailgun](https://www.mailgun.com) for sending our transaction email (whom promote themselves as a "Transactional Email API Service For Developers"). We quickly discovered that Yahoo and AOL blacklisted Mailgun and as a result none of our customers with a Yahoo or AOL email could receive any of our verification emails. This was extremely problematic as it resulted in us losing customers as it turns out people still use Yahoo and AOL for their email hosting. Yahoo/AOL and Mailgun seem to have a hateful relationship that even [Mailgun wrote a blog post about it](https://www.mailgun.com/blog/yahoo-aol-throttling-mailgun-festivus-airing-grievances-part-two/). None of the suggestions in the post deemed useful as we rarely sent emails to these providers and they still blacklisted us, so it was off to another provider for us.

### SendGrid

This was when we went to [SendGrid](https://sendgrid.com). Everyone loves to talk about how great this provider is so we decided to give it a shot. We noticed we were able to send emails to all major email providers (Gmail, Sendgrid and Outlook) which was a call for celebration, yay... but not for long. It wasn't long before we noticed that only ~93% of our emails were being sent to our users. With further investigation we discovered yet again that our users email providers were blacklisting our SendGrid IPs due to spam being sent from them from other customers. A German postmaster we contacted explained that

> Your provider has added you to a pool of senders that regularly sends spam to our customers and to our own spamtrap addresses.

At this point we were not happy with SendGrid and after some back and forth with their support team they informed us they are unable to move their customers to other shared IP pools manually as it gives the impression of "snowshoe spamming" and that they recommended us paying for a dedicated IP (an extra $75 a month). Actually the support staff said that they would "normally not recommend" having a dedicated IP but it would be the most "decisive way" to fix the issue. They also recommended us to go around emailing each postmaster to whitelist our domain. We were not pleased with their support as buying a dedicated IP would cause more problems due to the low amount of email we send so we decided to look again for another provider.

### Postmark, a new hope

We finally discovered [Postmark](https://postmarkapp.com) after finding their [Why Postmark?](https://postmarkapp.com/why) page on Google which got our attention. We decided to try it and at this point I was tired of transferring email templates across providers and rewriting code to use a new API, but just like Goldilocks the third time was the charm.

Before being allowed to use Postmark we had to fill out a form describing our reasons for using their service and we were manually approved very quickly. This is so their service is only used for transactional email, and so they can maintain their high reliability and deliverability rates. They have wonderful support and offer to talk to you face to face when you first sign up in order to make sure you are getting the best service possible. They also send weekly digests about your deliverability so you can keep track of whether or not you're having any issues with your email service. On top of this, they offer a [free DMARC monitoring service](https://dmarc.postmarkapp.com) which provides a human-readable summary of your DMARC reports. Postmark also provide [dedicated IPs](https://postmarkapp.com/dedicated-ips) for an extra $50 per IP per month, and offer automated warmup of these dedicated IPs to ensure your IP reputation is up to scratch. More information on their dedicated IP offerings is [available here](https://postmarkapp.com/dedicated-ips).

Enough with the sales pitch and back to our experience using them. Over 3 months worth of emails (~12k emails) only 7 emails were flagged as SPAM (all to a Russian email host). We had a bounce rate of 1.8% which is mostly due to mailboxes not existing (user typos) or mailboxes being overquota. It's safe to say that with these rates we are very pleased with Postmark and will not be going anywhere.

### Other providers

That being said there are other email providers out there which could offer a better service depending on your needs. If your application uses Amazon Web Services (AWS) it may be worth looking into Amazon's [Simple Email Service (SES)](https://aws.amazon.com/ses). This is because if you're send your email through SES with an application hosted on an Amazon EC2 it will cost you $0 for the first 62,000 emails you send each month. More information on their pricing is [available here](https://aws.amazon.com/ses/pricing). They also have a cheap rate for dedicated IPs ($24.95) and you can easily tie Amazon's SES with their [Simple Notification Service (SNS)](https://aws.amazon.com/sns), [Lambda](https://aws.amazon.com/lambda) or any other AWS services.

## Marketing/Bulk email

I personally haven't had to send any marketing or bulk email, but Postmark is now offering "broadcast messages" (bulk emailing) through their "message streams" service. More information is [available here](https://postmarkapp.com/message-streams). Since Postmark is so good at sending transactional email then I have high hopes for their bulk/broadcast email service too, and would encourage anyone eager to send marketing email to investigate.

Otherwise due to Amazon's flexible pricing system and cheap dedicated IP offerings it may be worth considering SES as you can pay per email sent and because of this you can be more flexible with your email budget.

I have also heard good things about [Mailchimp](https://mailchimp.com), Postmark also previously recommended them for sending bulk emails.

Due to my previous experiences with SendGrid and Mailgun I would not recommend them to anyone, perhaps SendGrid for bulk emails but be wary of your delivery rates with their service.

## Materials

If you're interested in learning more about email reputation [Mailjet has a good article on the subject](https://www.mailjet.com/blog/news/sender-score-and-email-reputation). 
