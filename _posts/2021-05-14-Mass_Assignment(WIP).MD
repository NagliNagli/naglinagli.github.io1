---
layout: single
title: Mass Assignment exploitation in the wild - Escalating privileges in style
date: 2021-02-25
classes: wide
thumbnail: https://galnagli.com/images/Mass-Assignment.jpg
header: https://galnagli.com/images/Mass-Assignment.jpg
header:
  teaser: /images/cache-poison.jpg##
--- 

![preview](/images/Mass-Assignment.jpg)

## General

As I have been accepted to Synack's Red Team at the beginning of march, the opportunity emerged required me and other bounty hunters to conduct a "mental shift", as most of the lucrative bounty programs on the public platforms will be the wildcard scoped ones, it's mostly the contrary while hunting on Synack.

Working with the model of "QR" periods, most hunters will receive 2~3 new application testing invites each week which starts with 8 hours Quallity Rule, meaning it's not about being the first to report the issue, it's about the best report and the most parameters invovled.

those invites mostly invlove a post-authentication webapp testing for a certain functioniality of big organisations.

As we are as researchers the first security engagement the application faces, it often exposes some unique vulnerabilities and exploitations.

During one of those engagements I managed to find a Mass Assignment vulnerability, which helped me escalate privileges from normal user without any permissions to a fully permissive adminsitrator account.



### Mass Assignment

As the vulnerability of "Mass Assignment" is not often talked about and there are not many writeups / disclosed reports of it, I'll briefly explain the core nature of it.

as per [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

```
Software frameworks sometime allow developers to automatically bind HTTP request parameters into program code variables or objects to make using that framework easier on developers. This can sometimes cause harm.

Attackers can sometimes use this methodology to create new parameters that the developer never intended which in turn creates or overwrites new variable or objects in program code that was not intended.
```

Adding to that, as per [OWASP API top 10](https://owasp.org/www-project-api-security/), Mass Assignment is ranked at the 6th place, following it's defenition:

```
Binding client provided data (e.g., JSON) to data models, without proper properties filtering based on an allowlist, usually leads to Mass Assignment. Either guessing objects properties, exploring other API endpoints, reading the documentation, or providing additional object properties in request payloads, allows attackers to modify object properties they are not supposed to.
```


To understand the attack vector better, let's take a look at the following image [credit](https://global-uploads.webflow.com/5ffc6b565fe11e76be021da1/602eda3ff4abd10d2e127f9c__56aRngXYuute5buEI-7RY2kkbixkjMPYxw1_dX0D_t863mTh6kwzlZTCKrOuNGpmKVRg-1_hWgXDiqk4-ddN5bV8ZM-lkvfWgb2FOnN7WCGithAQ9Es-pl6x-s6UQ_NWgHSEMAu.png)

![Mass_assignment_example](/images/Mass_Example.png)

The image above demonstrates in a scenario which was very similar to my finding.

as we can see we are dealing with an API which accepts JSON objects from a client, those are common on API's when we want to update our account information, for example changing our phone number or email address.

We are being presented with a scheme which mostly is supported in the UI, as we are being shown text fields to fill for address,email,first name and so on.

On the same user update panel, usually there are administrator functionalities as well for the site owner, and on those the JSON Object expands with the displayed UI with more options as is legitimated for administrator account, he might have the ability to set roles for other users, and choose their way of authentication.

It's pretty clear now to see how the vulnerability exploitation will come apart, as the sensitive fields are not presented to the regular user in his UI or upon his user update request, there might be a misconfiguration in the Authorization model within the application, which will accept the unknown JSON fields from none permissive user account and will process it as it was an administrator.

Taking it to the example above, imagine the user adds to his request the following key:value

```
"role":"admin"
```

As the server won't enforce any authorization check, the user will become an administrator in a sight of a single PUT request.

You might have wondered by now how can we as regular users without any permissions have guessed the hidden key:values pairs which might be hidden and exploitable?

There are 2 main vectors to find those, which can be divided into "Blackbox" and "Whitebox" approach

### Blackbox Approach

As for the "Blackbox" testing, we can use Burp's [Param Miner](https://portswigger.net/bappstore/17d2949a985c4b7ca092728dba871943) extension, when we have HTTP request which sends data within JSON format, we are being presented with the "Guess JSON parameters" option, which will bruteforce for common JSON params and see if it affects the server response, by issuing the Param Miner probe we can guess for those hidden sensitive fields.

![json_param](/images/json_param.png)

### Whitebox Approach

If we are being presented within our testing with adminsitrator credentials, we should navigate throught the supplied account, mostly into the profile update fields or new user invitations, mainly looking for those PUT or POST json based requests and note down the json parameters supplied in those.

Later, we should approach the application the same way with normal user account, by matching and comparing the query params available for none-permissive account to an administrator we can determine and craft payloads based on the differences.

## Exploitation

After going through the foreword, we can dive in the actual vulnerability I managed to find during my assesment.

As for certain application testing for a client within Synack, we were presented with a post-auth application with several user roles, there were editors, viewers and administrators.

As a general approach with like-wise application I'd look for potential endpoints which could be vulnerable to Privilege Escalations, as those often require learning the application behaviour and will most likely will have less competition going over the Quality Rule results later on.

We were presented with Adminsitrator Account only throughout the following testing.

As I was navigating throughout the application, I found an endpoint which gave us the ability to invite new users to the our organization environment

```
https://example.com/app/users/addNew
```

Therfore I invited myself for an additional account with a "Viewer" permissions only, so I could conduct the testing in an efficent manner.

![user-invite](/images/user-invite.png)

### Getting the first foothold.

As I was getting the duplicate notifications from Bugcrowd I decided to dig deeper into the application, as even for the fact I was invited 2 days from most  of hackers to the program, I still believed that there are more bugs to find due to the variety of functionialities the applicaiton presented.

Indeed I was proved that this is the case when I found 2 IDOR's which rewarded me nicely, and made me to go deeper on the "weird" looking page which I got duped for my Reflected XSS a day earlier.

Examination of the page:

![inital_page](/images/initial_page.png)

The page reflected some of the headers from my request, including the referer header, useragent, timestamp and IP address field which I could confirm that is mine.

It was being served on one of the main targets subdomains, and I have gotten to the specific endpoint by navigating through some waybackmachine endpoints, therefore I didn't have any query params on my initial request to the endpoint.

As it was reflecting some params and looked like a page which definitely shouldn't serve any purpose for it being public, I decided to run [ParamMiner](https://github.com/PortSwigger/param-miner) to get some query params, using the **Guess Everything** option.

After the scan finished there were some newly discovered query params which led me eventually to duplicate Reflected XSS, due to the fact that it was possible to guess those query params by inspecting the page contents.

A few minutes later I noticed that the scan returned to me **Cache poisoning 3** Flag, indicating that it's firm that there is Web Cache Poisoning issue on the page.

![cache_alert](/images/cache_alert.png)

I started with a quick fingerprinting checks, and saw that the target is running on CloudFlare, and that my requests are indeed being returned with the **CF-Cache-Status: HIT** response, which means that the response to the page will be presented from the cache.

Why did we get the Cache Poisoning alert? 

This is due to the fact that CloudFlare supports [X-Forwarded-For](https://support.cloudflare.com/hc/en-us/articles/200170986-How-does-Cloudflare-handle-HTTP-Request-headers-) Header in it's requests, which will append the input being inserted inside the parameter to the existing IP addresses of the client.

The scanner identified that the cache is being interpreted and evaluated with the **HIT** response, and that we have Unkeyed params which can be used to differ between requests, and we can serve the same page with additional context to victims who query the specific cached endpoint.

Now, into the exploitation part.

### Exploitation

At this point I have strong indication that the page is vulnerable to Web Cache Poisoning, although I need to show some impact being presetend from it.

As we saw from the earlier request to the page, it reflected back few headers from my original request, like the referer header or the IP addresses which made the request to the frontend proxy

First thing which came to my mind is what will happen if I add 

```javsascript
"><script>alert("nagli")</script>
```

to the X-Forwarded-For header? will it reflect in the response?

![hit_request](/images/hit_request.png)

And it was the case, I managed to inject XSS payload from my header request, which got reflected in the page and got **HIT** indication fromm the **CF-Cache-Status header**.

In order to verify that the content is being served from the cache, we should initiate a second request without the **X-Forwarded-For** Header this time, and to see if the response remains the same.

![2nd_req](/images/2nd_req.png)

It's a success, I managed to cache the XSS to other participants (In this case on my WAN), which means that every device which was connected to my router at that time was infected with stored XSS when he navigated to the infected endpoint.

One crucial thing to note out that took me some time to understand is the fact that not every extension from the page will get cached from cloudflare, you can refer to this [Understanding Cloudflare Cache](https://support.cloudflare.com/hc/en-us/articles/200172516-Understanding-Cloudflare-s-CDN) in order to know  more about CloudFlare caching mechanism.

What this actually meant to my exploitation is the fact that my original endpoint was an HTML one, which looked like the following

```
https://subdomains.example.com/somefolder/someendpoint.html
```

Therefore, CloudFlare will never serve it from it's cache, as for my praticular case the endpoints were the same on all requests within the subdomain, so I decided to craft my payload on an endpoint which will be cached eventually, like the following:

```
https://subdomains.example.com/somefolder/someendpoint/nagli.css
```

As CloudFlare will happily cache css files, it had no problem with my exploitation payload.

![cache_xss](/images/cache_xss.png)

Eventually I have stored XSS vulnerability which can be used to exfilitrate cookies from the main domain, which would lead to account takeover from the main site upon navigating to the cached endpoint.

I will give a few tips on writing Web Cache Poisoning report, as it took some back and fourth with Bugcrowd's triagers until we came to conclusion to triage it as P2 issue, because of the uniqueness of the issue.

### Submitting the report

I have submitted my report to Bugcrowd as **Web Cache Poisoning via X-Forwarded-For Header to Stored XSS on subdomain.example.com**

Upon my first submission I didn't take into consideration the file types which cloudflare caching accepts, which madem y reproduction steps not reproducible at first.

Also, as it was only affecting my nearby area and devices connected to my Wifi spot, I couldn't just craft a point and send it to the triager with alert box popping.

Reading this great [Cache Poisoning Report](https://hackerone.com/reports/303730) gave me some idea about explaining the nature of the problem in clearer manenr, and explaining the reproduction steps to it's best way.

![poisoning_areas](/images/poisoning_areas.png)

I have included those lines of wisdom which explain really well the idea behind the cdn "regions", and where as attackers we can craft our PoC to attack and target other regions.

Although we shouldn't guess the region of our triager, so it should be enough demonstrating the impact within our local network, giving the following clear reproduction steps

```
Steps to Reproduce:
1. Intercept the request to the following page https://subdomain.example.com/somefolder/someendpoint/nagli.css using burpsuite or any other tool.
2. Add the X-Forwarded-For header: "><script>alert(1)</script>
3. Get the request to the Burp repeater and send requests until you get "CF-Cache-Status: HIT" from the server
4. Remove the X-Forwarded-For header and send the request again, note that XSS payload is still being served from the cache
5. Navigate to the cached endpoint from different browser and note that alert will execute.
```

Those steps did the job just fine, and managed to get my report triaged as P2.

![reward](/images/reward.png)

### Timeline

- [ ] Web Cache Poisoning issue submitted - 29/01/2021
- [ ] Triager couldn't reproduce the issue - 06/02/2021
- [ ] Clearer reproduction steps submitted - 07/02/2021
- [ ] Triage as P2 - 09/02/2021
- [ ] 1000$ Reward - 09/02/2021


### Conclusion

I didn't try to go deep on Web Cache Poisoning as a concept in general because there are many great resources for that knowledge, the main idea about this blog post is to show how I approached and practically managed to exploit what considers to be rare vulnerability and noting it down into more friendly steps.

## Thanks for sticking out!

Some Social Links:

- [ ] Twitter: <https://twitter.com/naglinagli>
- [ ] HackerOne: <https://hackerone.com/nagli>
- [ ] Bugcrowd: <https://bugcrowd.com/Nagli>
- [ ] Linkedin: <https://www.linkedin.com/in/galnagli>

Credit to CloudFlare for the Poisoning image.
