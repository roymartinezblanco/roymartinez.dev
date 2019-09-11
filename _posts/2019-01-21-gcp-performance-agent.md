---
layout: post
title: Creating a Web load/Performance agent on Google Cloud
image: /img/gcpblog.jpg
tags: [google,gcp, webperf, webpagetest, sitespeed.io, automation]
---

As part of my day to day, I help my customers with their Web performance by
providing assessments on how to improve the web performance footprint.

Recently I found my self in the need to generate Web Traffic to a Staging
environment to feed data to a [Real User Monitoring (RUM)
solution](https://www.akamai.com/us/en/products/web-performance/mpulse-real-user-monitoring.jsp).
The caveat is that since we are talking about RUM we can’t simply send a simple
[cURL](https://curl.haxx.se/) because it will not be a real representation of
what a user will experience.

The test will need to parse the HTML and download all assets, execute all
js/CSS, render, etc. All of this is needed for
[RUM](https://www.akamai.com/us/en/products/web-performance/mpulse-real-user-monitoring.jsp)
to be executed (since it is deployed as a JS) and for it to capture metrics as
needed. Ideally, I wanted to use an opensource tool or something that meets the
criteria. There are a lot of tools that can be used but they do not allow
spoofing of the IP for us to send traffic to the staging application that has no
DNS record or require a subscription.

![](https://roymartinez.dev/img/1*Bbn0hWuk9_98_OhQSZq8ow.jpeg)

I decided to create a [Google Cloud](https://cloud.google.com/) [Compute Engine
instance](https://cloud.google.com/compute/) simply because its easy to use and
you can get a VM up and running very quickly at a [low cost
](https://cloud.google.com/compute/pricing)$4 for a month’s use (Ideal for
testing).

Thinking that this project is not going to be long term I chose Ubuntu as the
Operating System for the Virtual Machine because I needed something that was
fast to set up.

#### Google Cloud Setup

Go to Compute Engine and Click on “Create Instance”.

I chose the micro but later changed it to small due to the system running out of
memory and finally selected the OS.

Now, access the instance.

Here is the official guide on how to create a VM on Google Cloud:

Now that I have a VM to work with I thought of 2 solutions for my task:

### 1. WebPageTest

> “WebPagetest is a tool that was originally developed by
> [AOL](http://dev.aol.com/) for use internally and was open-sourced in 2008 under
a BSD license. The platform is under active development on
[GitHub](https://github.com/WPO-Foundation/webpagetest) and is also packaged up
periodically and available for
[download](https://www.webpagetest.org/forums/forumdisplay.php?fid=12) if you
would like to run your own instance.”

[WebPageTest](https://webpagetest.org/) was one of the first tools that came to
mind that met the criteria above, especially since you can automate tests on
[WebpageTest with its
API](https://sites.google.com/a/webpagetest.org/docs/advanced-features/webpagetest-restful-apis)
and even use tools like [this wrapper for
NodeJS](https://github.com/marcelduran/webpagetest-api).

This is a great tool for the job since I can automate recurrent tests with it
and use [all the locations
available](https://www.webpagetest.org/getLocations.php?f=html&k=A) within it.
With its API you access to 200 tests “per day”, each test can be up to 10 runs
and each run has a repeat view that would give you a total of 4,000 tests per
day.

#### Configure Webpagetest Wrapper

The setup is pretty simple, first, update your system, then install
[nodejs](https://nodejs.org/en/) and install the wrapper ([Follow the install
guide](https://github.com/marcelduran/webpagetest-api))

    $ sudo apt-get update && apt-get upgrade -y
    $ sudo apt install nodejs -y
    $ sudo npm install webpagetest -g -y

Once this is done, I run a test like seen below:

#### My WebPageTest Setup

Now, that I have a way to create automated tests I can make it more useful for
my use case. I need to make full use of
[webpagetest](https://www.webpagetest.org/) full potential by adding more runs
to my tests by adding the option “ **-r 10**”.

    $ webpagetest test 
     --key ################################## -v -r 10

**Note**: you need to create your API Key here:
[https://www.webpagetest.org/getkey.php](https://www.webpagetest.org/getkey.php)

To automate I could run a script that executes in a loop like “**while true;
do**” but because of the 200 per day limit, I chose to create a cron job, to
control how many times it gets executed and have the cron execute a script.
Because of this limit, I can run 8 runs per hour making it 80 tests.

As an additional step, I installed “[jq”](https://stedolan.github.io/jq/) to
easily capture the testID in the response of the API call in case I need it in
the future.

    $ sudo apt-get install jq -y

With it installed I can send the output of the command to a file for reporting.

    $ echo $(date) $(webpagetest test 
     --key ################################## -v | jq '.data.testId') >> webpagetests.log

Keep in mind that you can customize your logs with all the WPT fields as needed.
I put the command in a script instead of the crontab to easily modify it without
the need to tamper with the cron job.

The cronjob will run every hour at the minute 20.

    20 * * * * sudo -u username /home/username/webpage.sh

With all of this, I should be able to get traffic to the application and data
should start showing up in the RUM solution. The only issue that I see with this
is if you want to add more tests cases/URLs. Because you start to dilute the 200
daily tests by each tests case and if you have 100 URLs this means that you get
2 runs per day for each URL. This might be enough for other cases but I needed a
bit more per test and not to mention that I am still missing the capability to
spoof to the staging environment.

### 2. Sitespeed.io

> “Sitespeed.io is a set of Open Source tools that makes it easy to monitor and
> measure the performance of your web site.”

My final choice was [Sitespeed.io](https://www.sitespeed.io/), that is another
great tool that allows performance testing with awesome reports as seen below
that can be feed to a
[dashboard](https://www.sitespeed.io/documentation/sitespeed.io/graphite/) for
more long term projects.

For my case especially useful because of its use of headless chrome (VM == NO
DISPLAY) and integration with [webpagetest](https://www.webpagetest.org/about)
that allows for scripts to be sent to it, therefore, allowing spoofing.

#### My Sitespeed.io Setup

I chose to install it locally instead of using the docker instance (which would
have been a lot easier) because I needed to spoof where the tests went. But if
you want to use the docker approach [here
is](https://docs.docker.com/install/linux/docker-ce/ubuntu/) a guide on how to
get it working.

Same as before, I started by updating the OS and installing the packages I
needed but after some try and error I noticed that chrome was not installed by
default, therefore, I added the repository and then installed it.

    ## Login as SU
    $ sudo su
    $ sudo echo "deb [arch=amd64]  
     stable main" >> /etc/apt/sources.list.d/google-chrome.list
    $ exit
    ## Now install everything
    $ sudo apt-get update && apt-get upgrade -y
    $ sudo apt install nodejs -y
    $ sudo apt-get -y install google-chrome-stable
    $ sudo apt install nodejs -y
    $ sudo apt-get install python-pip -y
    $ sudo pip install selenium
    $ sudo npm install --unsafe-perm -g sitespeed.io
    $ sudo apt-get install jq -y

At this point I was able to make a simple test (that worked):

    $ sitespeed.io 
     --headless #Use HEADLESS
    [2019-01-21 17:55:58] INFO: Versions OS: linux 4.18.0-1005-gcp nodejs: v8.11.4 sitespeed.io: 7.7.3 browsertime: 3.11.1 coach: 2.4.0
    [2019-01-21 17:56:01] INFO: Starting chrome for analysing 
     3 time(s)
    [2019-01-21 17:56:01] INFO: Testing url 
     iteration 1
    [2019-01-21 17:56:10] INFO: BackEndTime: 521 DomInteractiveTime: 1464 DomContentLoadedTime: 1464 FirstPaint: 1057 PageLoadTime: 3283

    [2019-01-21 17:56:11] INFO: Testing url 
     iteration 2

    [2019-01-21 17:56:17] INFO: BackEndTime: 309 DomInteractiveTime: 759 DomContentLoadedTime: 759 FirstPaint: 541 PageLoadTime: 2020
    [2019-01-21 17:56:18] INFO: Testing url 
     iteration 3
    [2019-01-21 17:56:25] INFO: BackEndTime: 355 DomInteractiveTime: 838 DomContentLoadedTime: 839 FirstPaint: 679 PageLoadTime: 2138
    [2019-01-21 17:56:25] INFO: 59 requests, 1619.64 kb, backEndTime: 395ms (±52.57ms), firstPaint: 759ms (±125.93ms), DOMContentLoaded: 1.02s (±181.97ms), Load: 2.48s (±328.87ms), rumSpeedIndex: 807 (±120.90) (3 runs)
    [2019-01-21 17:56:31] INFO: HTML stored in /home/roy_martinezblanco/sitespeed-result/medium.com/2019-01-21-17-55-58
    [2019-01-21 17:56:31] INFO: Finished analysing 

Because it is installed locally I can spoof by modifying the /etc/hosts forcing
traffic to the staging environment and for webpagetest as I mentioned before
sitespeed.io allows you to send a script to webpagetest via the option [“-
-webpagetest.script”](https://www.sitespeed.io/documentation/sitespeed.io/configuration/)
allowing spoofing to be done there as well.

    --webpagetest.key "##################################" --webpagetest.runs 2 --webpagetest.location "ec2-us-west-1:Chrome" --webpagetest.script "setDns \t 
     \t 1.2.3.4 \n navigate \t {{{URL}}}}"

With sitespeed.io I can now run as many tests as a need but it also allows me to
provide a list of URLs that it will run a test against (simplifying everything).

    $ sitespeed.io urls.txt --headless -n 100

If I provide 10 URLs in the “urls.txt” and told it to do 100 runs, this sums up
to 1000 total per command. You can add this command to a cron job to do it as
many times a day that you need.

    20 * * * * sudo -u username /home/username/webpage.sh

The script I executed, in the end, is a little different because I used a very
small VM and I didn’t need the reports created by
[sitespeed.io](https://www.sitespeed.io/) but I did find useful the execution
out it provides.

In the script, I removed the folder created by
[sitespeed.io](https://www.sitespeed.io/) after each run to avoid using too much
disk space and send the output of the command to a log file to see test run
information.

    #!/bin/bash
    $ sitespeed.io urls.txt --headless -n 100  >> sitespeed.log
    $ rm -fr sitespeed-result/

### Conclusion

I created this blog because I saw no other guide or setup guide for
[sitespeed.io](https://www.sitespeed.io/)/[WebpagetestAPI](https://github.com/marcelduran/webpagetest-api)
without the docker I thought that this might help some of you out there.

I’m sure there are many ways to do this like using chrome by its own or with
puppeteer and I’ll probably make another blog of my experience with the other
approaches.

For me the [sitespeed.io](https://www.sitespeed.io/) approach did the job, I was
able to get traffic to my customer and test all of the URLs without a limit on
how many tests I can execute and can continue to improve going forward.
