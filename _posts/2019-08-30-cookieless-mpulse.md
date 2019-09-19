---
layout: post
title: Cookieless mPulse
tags: [GDPR,EU, RUM, Analysis,Cookie, mPulse, Property Manager]
image: /img/posts/201908/mpulselogo.png
bigimg: /img/posts/201908/eubg.jpg
---


When working with our customers that have a global user base, it can be safe to assume that they will have user privacy settings per region. For example, a customer had a requirement that we don't create cookies on the user's browser if they have opted out of any analytics.

***Note*: In this customers use case they set a cookie in the user's browser that indicates if they have opted out or not.**

This does not mix well with mPulse, because it not only looks at page views but also looks at the user's sessions and this is done by it creating the **"RT"** cookie on the user's browser. To comply with this requirement we could simply create match criteria around the mPulse Property Manager behavior and exclude it from getting executed.

***Warning*: This means that we lose all visibility on mPulse for those users that opted out via their privacy Settings/Cookie.**


The proposed solution is to make mPulse cookieless for the users that opt-out of having any cookies getting set on their browser. 

***Warning*: This means we lose session data but we don't lose all visibility into their performance and other insights captured by mPulse.**


To do so, all we need to do is create a new Property Manager rule that matches when the opt-out cookie exists and create a second mPulse behavior within it. In this mPulse behavior set the config override to remove the RT cookie with the following snippet.

![PM Override](https://roymartinez.dev/img/posts/201908/pmoverride.png)

### JSON Snippet

    {                    
        "RT": {           
            "cookie": ""  
        }                 
    }                    

For our use case, this created the question of how many users are opting out. So we created a custom dimension to provide the visibility that they did not have.

![Privacy Dimension](https://roymartinez.dev/img/posts/201908/dimension.png)
