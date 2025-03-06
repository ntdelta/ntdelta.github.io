---
layout: post
title: "AutoT212: Autopilot for Trading212"
subtitle: Defeating common apk protections to get rich quick
cover-img: /assets/img/autopilot/header.jpg
thumbnail-img: /assets/img/autopilot/auto.jpg
share-img: /assets/img/autopilot/minip.jpg
tags: [Frida, Android, Stock Market, Pelosi Tracker, Autopilot]
excerpt: Many US politicians trade on privileged information that the general public is not privy to. This is no secret and is well documented online. So well documented, that someone has made a business out of it.
---

## Introduction

Many US politicians trade on privileged information that the general public is not privy to. This is no secret and is well documented online. So well documented, that someone has made a business out of it. The Autopilot app allows users to copy politician's trades automatically and get rich quick. At first glance, it seems honorable. Sticking it to the big guy. Leveling the playing field. 

![alt text](/assets/img/autopilot/appsc.png)

However, they charge users to perform these copy trades. I don't want to spend money on my money printer, so I embarked to do it for free.

## Goal

The aim of the project is to create a Python library that can scrape the Autopilot portfolios and automatically synchronize them with Trading212 Pies. This would allow me to print money off the back of the politician's trades without paying the gatekeepers a subscription fee.

## Shortcut 1 : Mitm

The fastest way to success with Android apps is a mitm proxy. Configure your device to route traffic through it and copy/paste the POST requests out. Unfortunately, these days most modern apps implement cert pinning which means this shortcut will fail.

## Shortcut 2 : APK

If this fails, the next best option is to pull the apk from some sketcky mirror site and use JADX to manually find API routes. The Autopilot company has issued DMCA takedown requests to these sites, so we couldn't do it this way either.

## Shortcut 3 : Google Play

The last shortcut is to download the apk from Google Play and pull it over adb. We can then look at it statically and pull out the API routes. This worked, although the split apk revealed something gross inside:

![graphql](/assets/img/autopilot/graphql.png)

GraphQL. GraphQL is bad enough to forward engineer. I wasn't willing to invest any significant amount of time into this project so decided I was going to have to do it dynamically.

## You shall not pass

The app will not run on a rooted device. It will also not run if you are running frida. This is implemented by a library called `libdetection_lib.so`, a tiny binary shipped with the app. 

It runs very early on in the application's startup via a call to `runSecurityChecks`.
![alt text](/assets/img/autopilot/security.png)

At this point, we could just hook it with Frida and move on - but these are the IDA screenshots anyway:
![alt text](/assets/img/autopilot/runsecuritychecks.png)

![alt text](/assets/img/autopilot/frida_detector.png)

A simple Frida hook can bypass these security checks, and any cert unpinning script will enable you to MITM the application traffic.

```js
// Small delay to allow lib to load
setTimeout(function() {
    Interceptor.attach(Module.getExportByName('libdetection_lib.so', 'Java_finance_iris_autopilot_app_AutopilotApplication_runSecurityChecks'), {
        onEnter: function(args) {
        },
        onLeave: function(retval) {
          retval.replace(0);
        }
    });
}, 500);
```

## Getting the good stuff

Now we have unpinned the cert and bypassed the frida detections, we can see the API requests in our mitm proxy:

![mitm](/assets/img/autopilot/mitm.png)

And our desired API query which retrieves the portfolios and their allocations!
![alt text](/assets/img/autopilot/alloc.png)

We can then export the request as cURL and import it into Postman. Here we can easily mess with the headers and uncover something interesting. By removing all the headers and slowly adding them back, we identify that the header which allows responses to be retrieved is the `coordinate` header. This is a long GUID
which is hard-coded into the apk `BuildConfig`.

![alt text](/assets/img/autopilot/pelosi.png)

## Money Printer

Now the hard part is done, we can create our money printer. I used the Trading212 API for this, and created my Pies with the allocation proportion equal to that of the Autopilot tracker.

Each tracker is rebalanced automatically via the scraping package nightly, ensuring our money printer is always printing at maximum capacity:
![practice account!!!](/assets/img/autopilot/printer.png)

The [code for this project can be found on GitHub](https://github.com/ntdelta/autot212)

## What if we are their money printer

For some reason, the API returns the number of paid subscribers to the trackers. The Pelosi tracker has 62,830 paid subscribers at this time...
![alt text](/assets/img/autopilot/paid.png)