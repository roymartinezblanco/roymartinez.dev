---
layout: post
title: Google Tag Manager + Akamai mPulse.
tags: [Google, gtm, akamai, rum, mpulse]
---

This blog is meant to provide a quick guide on how to enable [Akamai mPulse
](https://www.akamai.com/us/en/products/performance/mpulse-real-user-monitoring.jsp)when
using [Google Tag Manager](https://tagmanager.google.com/).

![](https://roymartinez.dev/img/1*1YXzVCvb2Z8KnAs5qkRYYQ.jpeg)

#### This is a great option when you that already use GTM and maybe don’t have any
Akamai delivery products but keep in mind that if you do then you should use
Edge Injection to avoid losing visibility of the middle mile.

### 1. Create a new Tag

The First step once in the GTM account is to create a new tag.

![](https://roymartinez.dev/img/1*Rl07Gw7-yfTBPyeT9dhJLA.jpeg)

### 2. Set the Tag Configuration for Custom HTML

This will take you to the tag editor that has 2 settings:

* The type of tag
* when the tag will be triggered/added to the page.

Once in this screen click on edit for the “Tag Configuration” and you to see all
the options available. For mPulse, we will use the “**Custom HTML**” type.

![](https://roymartinez.dev/img/1*h-il_yFFJd2EKTkQxJXmbw.png)

Now from the mPulse App Configuration take the HTML snippet and added to GTM.

![](https://roymartinez.dev/img/1*ciKQNLTFwundyhUBXNixxg.jpeg)

### 3. Modify Trigger and add All Pages

At this point, we have the code that will insert mPulse but we now need to tell
it when to insert it.

![](https://roymartinez.dev/img/1*L8jJMT7z-ihh_-UajQATog.jpeg)

GTM has 3 main events that could be used but we are going to use “Page View” as
the event trigger.

* Dom Ready
* **Page View**
* Window Loaded

After the steps above you should have something like the image below and if so
you can go ahead and promote the change to GTM production.

![](https://roymartinez.dev/img/1*LrO6x3aY6ly3NsopAG6HZA.png)


Once the change is live you can test it by looking for the mPulse beacons in
[Chrome Developer Tools
Network](https://developers.google.com/web/tools/chrome-devtools/network/) tab
and filtering for “**/mpulse|akstat/” **and if all goes well we will see an HTTP
POST request with a 204 response code.

![](https://roymartinez.dev/img/1*BfqQLh-FnvXzk_g8I3R-EQ.jpeg)
