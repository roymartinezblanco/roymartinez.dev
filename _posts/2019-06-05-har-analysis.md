---
layout: post
title: HTTP Archive (Har) Analysis
image: /img/posts/201906/httparchive.png
tags: [Google,Chrome, devtools, Analysis,automation]
---

On this blog, I show how to query the contents of a har output to quickly
analyze all request/response headers for the session captured. You can use this
as a base to query all the requests in a session for any headers or other data
you might need within the HTTP Archive.

[Google Chrome Dev Tools](https://developers.google.com/web/tools/chrome-devtools/) is great for
inspecting Site traffic, debugging and so much more but sometimes I find my self
needing more data from it.

Let’s use [Medium.com](https://medium.com/) as an example:

[Medium](https://medium.com/@Medium) seems to be using
[Cloudflare](https://medium.com/@cloudflare) as its CDN (based on the response
headers) and [Cloudflare](https://medium.com/@cloudflare) uses the [response header “cf-ray”](https://support.cloudflare.com/hc/en-us/articles/200170986-How-does-Cloudflare-handle-HTTP-Request-headers-)
to indicate the DC that the request passed through. If I wanted to have a quick
way to view what DC was used for the requests and validate if one or more DC
were used. I would have to look at each request in Dev Tools and look at the
response headers… **but who has time for that?**

Thankfully Google Chrome gives us the capability to export the network session
to a .har file and it has all the request/response headers we will need to get
all the DC’s used by using the tool [jq](https://stedolan.github.io/jq/) and
awk.

**We will be doing the following steps:**

1.  Export Session to har.
1.  Parse the har file with JQ
1.  Extract DC name with AWK

### Export Session to har.

![](https://roymartinez.dev/img/posts/201906/1*e7UONEoWB-Ob6D_x0nQ9Kg.png) 

Open Dev Tools and refresh your site, once you do this you will see the network
tab populated with all the requests.

Right click on any of them and select **“Save all as HAR with content”**, this
will export the session data to a .har file to the directory of your choosing.

### Parse the har file with JQ

![https://stedolan.github.io/jq/](https://roymartinez.dev/img/posts/201906/0*8_VJsutqgdXmay7E.png)

JQ is a JSON processor that allows us to easily query for the fields we need and
also condition what data we get.

First, install it if you don’t have it. For mac, you can use brew:

![](https://roymartinez.dev/img/posts/201906/1*4yIsfJZ3Ape_-S0xWRQktQ.png)


The following will look at all the requests with the domain **“medium.com”** in
the downloaded har file and get the value of the response header **“cf-ray”**.

    $cat ~/Downloads/medium.com.har | jq '.log.entries[] | select(.request.url | contains("medium.com")) | .response.headers[] | select(.name | match("cf-ray";"i")) | .value'

**Let’s take it apart:**

If we look at the har file created it has an array **“entries”** that has all
the requests and it has the responses within it.
![](https://roymartinez.dev/img/posts/201906/1*cCB7uLrAnbHgt6YCVi4EQQ.png)
We will start by telling JQ to get all the requests/responses within entries.


But only look at the ones that have site.com as part of the URL**.**


Once we have the requests we want to extract the data from we again select the
response headers


Then look for the cf-rat header (ignoring Case)

    | select(.name | match("cf-ray";"i"))

And finally, extract the values.

    | .value

We now have successfully extracted the header information we want, but I want to
go a bit further and extract the DC name from the values that as a Hash that is
part of it as seen below.

![](https://roymartinez.dev/img/posts/201906/1*4yIsfJZ3Ape_-S0xWRQktQ.png)


### Extract DC name with AWK

    $awk -F'-' '{gsub(/"/, "", $2); print $2}' | sort | uniq -c | sort -n

**Let’s take it apart:**

With AWK we are going to set “-” as the delimiter with the option “-F ‘/’ ” to
easily split the string and use **“gsub(/”/, “”, $2)”** to clean the output by
removing the quotes from the string from the second field that we are going to
use.

Since we split the string we have 2 fields:

1.  Hash: “4e1e3b4b1b3cc5f0
1.  DC Name: EWR”

Finally, we have the data we want. But in my case, I sorted and counted the
instances of each DC to provide additional insights. (The example would have
been better if more than one DC was used :) but you get the point.)

![](https://roymartinez.dev/img/posts/201906/1*JuBOD1TzeFzt-P8pNmGwsg.png)
