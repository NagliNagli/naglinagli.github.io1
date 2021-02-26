---
layout: single
title: Poisoning your Cache for 1000$ - Approach to Exploitation Walkthrough
date: 2021-02-25
classes: wide
thumbnail: https://galnagli.com/images/cache-poison.jpg
header: https://galnagli.com/images/cache-poison.jpg
header:
  teaser: /images/cache-poison.jpg
--- 

**Poisoning your Cache for 1000$ - Approach to Exploitation Walkthrough**

![preview](/images/cache-poison.jpg)

## General

Cometh the month of February I was surprised to open my email seeing that I have recieved a new private program invite from Bugcrowd, as this is a rare occasion for many during the last couple of months.

The program was fairly new, and I received the invite 2 days upon their official launch, meaning that there already was first sweep for bugs from the first batch of security researchers.

I had good feeling about the program as I could find Reflected XSS on it in roughly 2 minutes of work, as the login page had vulnerable "returnTo" parameter which looked like the following:

```javascript
https://example.com/login?returnTo=javascript:alert(document.domain)
```

I gave the program a few hours of hunting and managed to report a few Reflected XSS and Stored XSS issues which eventually got closed as dupes.

However, one of those dupes led me to find a Web Cache Poisoning vulnerability which was escalated to 0 interaction stored XSS, In this blog I'll explain the approach to the exploitation.

The content and whole idea of the blog is based on [Practical Web Cache Poisoning](https://portswigger.net/research/practical-web-cache-poisoning) Research of [albinowax](https://twitter.com/albinowax)

Which exposed me to the Cache Poisoning bug class, although I felt alittle skeptical and didn't really think i'd encounter one of those in the wild.

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
