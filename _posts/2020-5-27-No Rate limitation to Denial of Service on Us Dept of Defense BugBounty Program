---
layout: single
title: No-Rate and Input limitations on password reset page chained into Denial Of Service attack on one of US Dept of Defense website.
date: 2020-5-27
classes: wide
header:
  teaser: /images/firstbug.jpg
tags:

  -BugBounty
--- 

Hello everyone, in this blog post i will share my first disclosed bug, which was found on US Dept of Defense Bug Bounty program.

Report link: <https://hackerone.com/reports/862681>

![first_bug](/images/firstbug.jpg)

## Summary:
No-Rate limit on the website password reset page, chained with the ability to send Unrestriced input length by the user,
leads to Denial Of Service.
It comes into effect when you can send very long strings as input (2M chars) and with no rate limit using Burp Intruder.

## Walkthrough
While testing for vulnerabilities on web application, i often encounter the password-reset page as an interesting place to explore.
![passwordreset](/images/password_reset.png)

- [ ] Enter your 2nd email address as input , and capture the request on burp.

- [ ] Pass your request to Burp Intruder using CTRL+I, and check whether Mass-Spam is available due to no-rate limit implementation.

![mail_spam](/images/mail-spam.jpg)

- [ ] This error itself might get triaged at some Bug Bounty programs as P4, but as it wasn't on DoD Scope, i had to chain it into more severe vulnerability.

- [ ] At this point, i tried to check whether there is input length restriction on the email field, and found out that there isn't such thing.

- [ ] I decided to craft Long string and mass-spam it, to check the server response for high scale POST requests.

![contentlength](/images/contentlength.png)
![200](/images/DOD200.PNG)

- [ ] As you may notice, at the response the server notifies that you must enter email address shorter than 254 charcters, although there is no enforcement on that rule.

- [ ] Now, the waiting game begins...

![fewmoments](/images/fewmoments.jpg)

- [ ] Eventually, a "Spike" had occurred, which indicated that the site is experiencing Denial of Service (503,502 Response Codes).

![spike](/images/SPIKE.PNG)

## Suggested Mitigation/Remediation Actions
- [ ] Limiting the password reset request to once every X minutes.
- [ ] Use CAPTCHA verification after X requests.
= [ ] Asserting random password-reset link for each request.

## Conclusion
As an addition to your bug bounty methodology, i suggest adding this simple check to your checklist, as it requires no special tools,
or sophisticated thinking, and could turn out as Rewardable report on paying Bug Bounty Programs.

Hope you enjoyed reading through the report.

![thankyou](/images/thankyou.jpg)

Cheers,
Gal.
