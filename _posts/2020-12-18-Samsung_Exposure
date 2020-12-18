---
layout: single
title: Broken Access Control on samsung.com subdomain leads to leakage of employees and customers authCodes, email addresses and full name.
date: 2020-12-18
classes: wide
header:
  teaser: /images/Samsung_teaser.png
tags:

  -BugBounty
--- 

**Broken Access Control on samsung.com subdomain leads to leakage of employees and customers authCodes, email addresses and full name**

![preview](/images/Samsung_teaser.png

## Greetings

As we have entered the month of december, I have decided to freshen up some targets and decided to give a look at Samsung's Bug Bounty programs as I had good experience with them at the past.
Samsung operates their bounty policy on the following link (https://security.samsungmobile.com/)
As they state that only vulnerabilities affecting the samsung mobile/tv team services or devices is eligible for bounty, you can't really know what services actually belongs to them, so I have decided to look at the *.samsung.com scope


![recon image](recon image)

### Reconnaissance

As you might have read from my previous blog post, there is no magic when it comes to reconnaissance.
Utilizing my bash script which integrates the known open source tools for subdomain discovery presented 853 alive probed subdomain results

![recon](/images/853.png)

The nuclei scan for known vulnerabilities didn't return any significant result, so it was time to dive in the subdomains and look manually for intersting information.

At first it would be normal to look at the subdomains which include intersting keywords as dev,admin,stage.

Eventually this led to me explore few subdomains which consists those keywords, and in particular the vulnerable subdomain https://admin.csr.samsung.com/

### Manual Inspection

Navigating to https://admin.csr.samsung.com/ presented us with a static web page with a small login button up top.

![recon](/images/static_page.png)

Observing the login functioniallity was just utilizing the SSO of samsung.com, I was redirected to account.samsung.com and had to enter my account information, and later on redirected to fill up another few details at the origin of the request.

![login](/images/flow_video.mov)

We are unauthorized to access the data which the subdomain servers, probably because we are not samsung employees and as we need to wait for the manager approval.

So, pretty much a dead end?

### Exploitation

While observing the functionallities my Burp proxy was up and running, and the JS Link Finder extension came clutch.

JS Link Finder is the Burp Extension for a passively scanning JavaScript files for endpoint links. - Export results the text file - Exclude specific 'js' files e.g. jquery, google-analytics
(https://portswigger.net/bappstore/0e61c786db0c4ac787a08c4516d52ccf)

Observing the links which the extension found we could notice that there are many rest api endpoints which could be intersting to determine if they are accessible from unauthenticated and unauthorized user perspective

As there were 600 intersting links to look on, doing so manually wouldn't be effective, you can copy the complete list from JS link finder, create a new text file, cutting it to remove the ordering by "cat jstest.xt | cut -d " " -f 3"

Supplying the list to ffuf we would notice several intersting endpoints which return 200, so this narrowed the list to be compatible with manual observation.

![endpoint](/images/endpoint.png)

The following endpoint prove to be critically severe:
(It's fixed now)

```javascript
https://admin.csr.samsung.com/rest/v1/system/list/
```

The page supplied every user which used the login form with his account, with the following details:

{"userId":150,
"login":"example@samsung.com",
"authCode":"aaa13QIsoczffAF6YvAOJkuzXXXXXXXXXXXXXXXXXXXXXXX",
"fullName":"Israel Israeli",
"email":"example@samsung.com",
"locale":"ko","timezone":"Asia/Seoul",
"datetimePattern":"YYYY-MM-DD HH:mm",
"statusId":3,"epid":null,
"loginFailCount":0,
"orgId":null,
"deleteYn":"N",
"createUserId":4,
"createDate":"2020-10-17T05:44:47Z",
"updateUserId":null,
"updateDate":null,
"lastAuthCodeUpdateDate":"2020-10-17T05:44:47Z",
"token":null,
"dept":null,
"empYn":null,
"reqRole":null}

There were approximatly 200 users, including administrators.

Now we need to figure out how is the authCode implemented on the application we are testing?

After issuing the login functioniallity from the account.samsung.com we can observe that the following request is being initiated:

![code](/images/code.png)

Replacing dumped auth code with the one I have issues allowed me to bypass the restriction and access the application as the victim account.

![K.O](/images/giphy.webp)

##Impact


## Key Takeaways

So, after presenting to you the severe yet simple IDOR i have found on the DoD, those are the key takeaways i want you to take from my blogpost.

### Think outside of the box

![get_started](/images/outside_thebox.jpg)

When you encounter a program with alot of assets, don't just stick to the normal content discovery and subdomain recon tools, try to think and reflect at first where i might have the bigger chance to find vulnerable endpoints and misconfigurations.
Google Dorking is a great asset to express your creativity while supplying various of search queries to find specific and accurate data about your target

### Avoid rabbit holes

![hole](/images/rabbit_hole.jpg)

We need to be mature enough to understand if a specific endpoint is hardend, or we might find vulnerabilities in it.

The vast majority of DoD login pages are using the same login functioniallity, asking you to supply a CaC card and is hardend with many security measures.

Although someone might find a misconfiguration in this process, we should identify that it's less likely to find one and invest more on our recon to find the web pages which are stick out differently to the common coding and work ethics of the company.

Investing our time on domain specific and unique designs insted of the wide and common functioniality which is being used on most of DoD websites will make our researching time more efficent and fruitful, and will help us avoid being burnt-out.

### Dedication

I really recommend that you set yourself monthly/weekly goals, some days or weeks you won't find anything, and one day you will find a Critical bug without any preperation and expectation.

As you might notice, by the time this report was triaged on the 27th of August, i had 192 reputation points, and i haven't gotten my first reward from bug bounties.

Today, as of the day i have written this report on November 7th 2020:

- [ ] I have reported bugs which rewarded me 4 digits in total
- [ ] I have passed yesterday the 1K reputation mark on H1, 704 of those points on the DoD program
- [ ] I have reported vulnerabilities to major companies including H1, BugCrowd, Samsung and more..

![1k](/images/1K_rep.png)

During that span of time I have done many things to boost my Web Application Security level, such as:

- [ ] Completing most of Portswigger labs <https://portswigger.net/web-security>
- [ ] Building my own Automation tool on my VPS using the great open source tools of <https://twitter.com/pdiscoveryio>
- [ ] Reading alot of tweets, writeups, videos from fellow bug bounty hunters in the community.

The point here is not to brag about myself, is to inspire you to put those hours and dedication to the things which drives you and makes you wake up at night.


## Thanks for sticking out!

Hope you enjoyed reading my writeup, and i hope you can implement one of the tips i have supplied below to boost up your Bug Bounty Hunting Game!

If you did so, Please share my blog to spread it upon the Bug Bounty Community :-)

You can find me on:

- [ ] Twitter: <https://twitter.com/naglinagli>
- [ ] H1: <https://hackerone.com/nagli>
- [ ] BugCrowd: <https://bugcrowd.com/Nagli>
- [ ] Linkedin: <https://www.linkedin.com/in/galnagli>

![thanks](/images/theend.jpg)


