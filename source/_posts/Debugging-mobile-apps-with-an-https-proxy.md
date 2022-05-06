---
title: Debugging mobile apps with an HTTPS proxy
date: 2019-05-27 22:00:00
tags: [analytics, android, ios]
---

As a part of my previous job, I occasionally had to intercept HTTPS requests made from a mobile application. This is required to troubleshoot some issues and guarantee that the correct data is being sent. I will explain how I do it and talk about analytics while I briefly analyze a popular app.

## Analytics

How much time users spend on the app? How many times do they return per month? Does performance have a significant impact on other metrics? Which button is clicked the most from our A/B test? These are questions that can be answered by analytics and help in making data-driven decisions which previously relied on intuition.

As storage and computing resources are getting cheaper, data collection and processing increases. Many apps track all the relevant events and aggregate this data for further analysis.

Let's take a look at the Airbnb app. They are hitting 2 different tracking endpoints. Events are tracked regarding navigation, performance, searches, app lifecycle, and contain information about the device such as the model, screen size, network type, language, etc... On the app, these events are sent in batches to reduce bandwidth usage, which is not the case on their website as scrolling through the main page will trigger dozens of tracking requests, as we can check in the dev tools. An example of the tracked events of the app can be found in the image bellow.

{% asset_img "airbnb-tracking.png" "Airbnb Tracking" %}

We can observe an object with a key called "advertising-ID" with the value containing several zeros. One feature not known by many people and enabled by default in Android and iOS is the advertising ID. It is a random unique identifier that ad companies can use to track users, similar to a tracking cookie on the web. In the phone settings we can request a new advertising ID or even disable it completely, forcing apps to rely on other data to track users (IP address, device fingerprinting, generated identifiers).

To disable the advertising ID follow these steps:

- Android
  - *Settings > Google > Ads > Opt out of Ads Personalization*
- iOS
  - *Settings > Privacy > Advertising > Limit Ad tracking*

## mitmproxy

One useful open-source HTTPS proxy is [mitmproxy](https://mitmproxy.org). It allows to intercept and modify requests through the terminal or web interface. Another popular free alternative with a focus on web security is [Burp Suite](https://portswigger.net/burp). This way we can reverse engineer the APIs used by apps.

The quickest way to start mitmproxy is with Docker:

```bash
docker run --rm -it -v ~/.mitmproxy:/home/mitmproxy/.mitmproxy -p 8080:8080 -p 127.0.0.1:8081:8081 mitmproxy/mitmproxy mitmweb --web-iface 0.0.0.0
```

Now we can access the UI at *localhost:8081*. To set the proxy on both iOS and Android devices, go to "*Settings > Wi-Fi*", tap the connected network and enter the computer's IP address as the server/hostname and *8080* as the port. In case the connection is failing, the computer's firewall may be the issue.

If the root certificate was not previously installed, visit *mitm.it* on your browser and click the image of your operating system to install the certificates. For Android, a dialog will appear to choose a certificate name and select "VPN and apps" as the "Credential use". For iOS go to "*Settings > General > Profiles*", tap on the created profile to install the certificate and then enable it in "*Settings > General > About > Certificate Trust Settings*".

Nowadays most popular apps employ a security technique called [Certificate Pinning](https://www.owasp.org/index.php/Certificate_and_Public_Key_Pinning). If the certificate is not trusted by the app, connections are dropped. To perform man in the middle attacks we now have to tamper with the app to disable certificate pinning using toolkits like [objection](https://github.com/sensepost/objection) and [Frida](https://www.frida.re).

## Closing thoughts

mitmproxy is a very useful tool. Addons like this one can be coded in Python:

```python
from mitmproxy import http, ctx
from random import randint

# simulate failure in 25% of tracking requests
# Usage (has live reload): mitmweb -s intercept.py
def request(flow: http.HTTPFlow):
  url = flow.request.url
  if 'tracking' in url and randint(1, 100) < 25:
    ctx.log.warn('Fail ' + url)
    flow.response = http.HTTPResponse.make(500, '', {})
```

I think that analytics is very important to figure out how users are interacting with the software. The data collected can be analyzed to provide a better service by identifying issues and trends, however care should be taken to respect user's privacy and comply with *GDPR*.
