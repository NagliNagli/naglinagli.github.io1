---
layout: single
title: Broken Access Control on samsung.com subdomain leads to Mass Account Takeover of Samsung employees application accounts
date: 2020-12-18
classes: wide
thumbnail: https://galnagli.com/images/Samsung_teaser.png
header: https://galnagli.com/images/Samsung_teaser.png
header:
  teaser: /images/Samsung_teaser.png
--- 

**Broken Access Control on samsung.com subdomain leads to Mass Account Takeover of Samsung employees application accounts**

![preview](/images/Samsung_teaser.png)

## Greetings

As we have entered the month of december, I have decided to freshen up some targets and decided to give a look at Samsung's Bug Bounty programs as I had good experience with them at the past.

Samsung operates their bounty policy on the following link [Policy](https://security.samsungmobile.com/)

As they state that only vulnerabilities affecting the samsung mobile/tv team services or devices is eligible for bounty, you can't really know what services actually belongs to them, so I have decided to look at the *.samsung.com scope


### Reconnaissance

![recon image](/images/recon1.jpg)

As you might have read from my previous blog post, there is no magic when it comes to reconnaissance.

Utilizing my bash script which integrates the known open source tools for subdomain discovery presented 853 alive probed subdomain results

![recon](/images/853.png)

The nuclei scan for known vulnerabilities didn't return any significant result, so it was time to dive in the subdomains and look manually for intersting information.

At first it would be normal to look at the subdomains which include intersting keywords as dev,admin,stage.

Eventually this led me to explore few subdomains which consisted those keywords, and in particular the vulnerable subdomain https://redacted.samsung.com/

### Manual Inspection

Navigating to https://redacted.samsung.com/ presented us with a static web page with a small login button up top.

![recon](/images/static_page.png)

Observing the login functioniallity was just utilizing the SSO of samsung.com, I was redirected to account.samsung.com and had to enter my account information,
and later on redirected to fill up another few details at the origin of the request.

![login](/images/flow_video.gif)

We are unauthorized to access the data which the subdomain servers, probably because we are not samsung employees and as we need to wait for the manager approval.

So, pretty much a dead end?

### Exploitation

While observing the functionallities my Burp proxy was up and running, and the JS Link Finder extension came clutch.

JS Link Finder is the Burp Extension for a passively scanning JavaScript files for endpoint links. - Export results the text file - Exclude specific 'js' files e.g. jquery, google-analytics

[LinkFinder link](https://portswigger.net/bappstore/0e61c786db0c4ac787a08c4516d52ccf)

Observing the links which the extension found we could notice that there are many rest api endpoints which could be intersting to determine if they are accessible from unauthenticated and unauthorized user perspective

As there were 600 intersting links to look on, doing so manually wouldn't be effective, you can copy the complete list from JS link finder, create a new text file, cutting it to remove the ordering by (" cut -d " " -f 3")

Supplying the list to ffuf we would notice several intersting endpoints which return 200, so this narrowed the list to be compatible with manual observation.

![endpoint](/images/endpoint.png)

The following endpoint proved to be critically severe (It's fixed now):

```javascript
https://redacted.samsung.com/rest/v1/system/list/
```

The page supplied every user which used the login form with his account, with the following details:

```javascript
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
```

There were approximatly 200 users, including administrators.

![samsung_creds](/images/samsung_creds.gif)

Now we need to figure out how the authCode is implemented on the application we are testing?

After issuing the login functioniallity from account.samsung.com we can observe that the following request is being initiated:

![code](/images/code.png)

Replacing dumped auth code with the one I have issued allowed me to bypass the restriction and access the application as the victim account.

![K.O](/images/giphy.webp)

### Impact

- [ ] Accessing an administrator only web app supplying credentials of samsung employees and the system/admin of the system

- [ ] Email addresses and full names of samsung employees using the application

### Remediation

The issue was fixed by samsung's security team while issuing a 403 error when trying to access the page as unauthorized personal

### Timeline

- [ ] Issue found and reported on samsung's mobile bug bounty platform - 12.12.2020
- [ ] Issue assigned to security analyst - 14.12.2020
- [ ] Issue fixed - 15.12.2020
- [ ] Report has been moved from samsung mobile department as its not a service being operated by them - 15.12.2020
- [ ] Recieved a "Thanks" letter from security@samsung.com team, which are not rewarding any bounties for findings - 17.12.2020

![lucky](/images/lucky.jpeg)

![thanks](/images/thanks.png)

### Conclusion

Although I didn't recieve any bounty from the finding, and the fact that I could earn the "thanks" letter from samsung by reporting a low severity issue as well, it was nice to find a critical misconfiguration on such a big company from what seemed to be pretty static page.

It makes you understand that the intersting parts when engaging with bug bounty programs are the unseen ones :-)

## Thanks for sticking out!

Hope you enjoyed reading my writeup, Sharing the blog could be nice and I hope you discovered new ways of approching a target from my blog :-)

You can find me on:

- [ ] Twitter: <https://twitter.com/naglinagli>
- [ ] H1: <https://hackerone.com/nagli>
- [ ] BugCrowd: <https://bugcrowd.com/Nagli>
- [ ] Linkedin: <https://www.linkedin.com/in/galnagli>

![thanks2](/images/seal.jpg)


