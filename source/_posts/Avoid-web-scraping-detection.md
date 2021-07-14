---
title: Avoid web scraping detection
date: 2020-04-18 16:00:00
tags: [scraping, javascript, node.js]
hidden: true
---

Web scraping is the process of extracting useful data from a website. Google built their company on this. Many websites take measures to prevent scraping for various reasons, while still allowing big search engines to crawl their content. I will share my opinion about this subject and how some websites detect web scraping.

## Motivation for web scraping

There are many legitimate reasons to scrape a website and I have a short story of mine. A few years ago, I really enjoyed watching memes from 9gag during my long train commute to university. Sadly, I had very low mobile data, so I created an automation to scrape a few pages of memes during the night and transfer them to my phone so I could watch them offline. I inspected their internal API and created a javascript client, which I open-sourced and shared with the community on [NPM](https://www.npmjs.com/package/9gag). This small contribution is now a part of fun personal projects made by other people like a [machine learning classifier](https://github.com/casassg/meme_puller), a [CQRS and event sourcing lab](https://github.com/NebulaEngineering/lab-cqrs-es), a [discord bot](https://github.com/Sam-Ryan/Scylla), an [aggregator website](https://github.com/danylkaaa/ReservoirCodeInt20Test) and others.

Scraping enables automation, giving more freedom to power users. Some platforms do not like this because we won't see their ads, however most power users block ads anyway as they usually cripple web apps. Other reasons to scrape may not be very ethical. For example, there are many bots scraping the web in search for emails to send spam. Any website is free to block scraping in any way they want, after all they are the ones that choose how to reply to requests. Linkedin sued a company for scraping their public content, but the court did not give them reason. A website term of services page hidden somewhere with a lot of text that no one reads is not really legally enforceable.

If websites provide a reasonable API, web scraping can be reduced. This way they can enforce limits on API clients, like the ammount of requests allowed per day, track how it is used and even make money by charging for additional API calls. Most web applications are already using internal APIs, they could be made public. Frequently, scrapers actually consume these internal APIs instead of parsing the whole website, which is more efficient for both sides.

## How web scraping works and evade detection

A web application can be rendered on the server side or client side. When the content rendered on the server and sent to the browser, scraping is easier because only HTML has to be parsed. When rendering happens on client side, a javascript interpreter is required to render the content, so a simple HTTP client is not enough to scrape content, a browser emulator like [Puppeteer](https://github.com/puppeteer/puppeteer) must be used, which is more resource intensive. I recently had to update my old library to use Puppeteer to avoid a CAPTCHA check.

It is ironic that Google created their empire on web scraping and are the ones to create solutions like CAPTCHA to prevent others from crawling. Some websites even include metadata to help Google scrap their content more accurately and easily, using [JSON-LD](https://json-ld.org) for SEO purposes. There are many other crawlers that behave nicely and have good intentions.

So how is web scraping detected? Common methods include checking HTTP headers like the user agent, device fingerprinting with javascript and inspecting connection or behaviour patterns. The easiest solution to avoid being detected is to use Puppeteer with a [stealth extension](https://www.npmjs.com/package/puppeteer-extra-plugin-stealth), which already takes some steps to avoid detection. In addition, navigation should look like a real user, with random time between requests and do not access URLs not visible to a user. A pool of servers with different IP addresses may be needed for mass scale crawling, however many companies can detect if the IP is from major cloud providers, so residential IPs are better.

## Conclusion

Web scraping detection will always be a cat and mouse game and some ways of detecting this were presented. However, in the end, if information is publicly available, even a real human can act as a web scraper with or without web extensions to help automate the process.
