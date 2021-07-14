---
title: Automation with Node-RED
date: 2020-08-04 22:00:00
tags: [automation, javascript, node.js]
hidden: true
---

Inspired by [iOS Shortcuts](https://apps.apple.com/us/app/shortcuts/id915249334) and [Zapier](https://zapier.com), I searched for an open source solution to easily create automations through a web interface and found two projects, [Node-RED](https://nodered.org) and [n8n](https://n8n.io).

## Node-RED

As Node-RED is more low-level and mature than n8n, I picked it to create some integrations. It is a flow-based programming tool developed in Node.js and is part of the [JS foundation](https://js.foundation). From the web browser, applications/flows are created by adding nodes and linking them together to send messages. Custom JavaScript code can be added in function nodes. The flows created are saved as JSON, which allows for easy sharing. To extend Node-RED, new nodes are developed by the community and published [here](https://flows.nodered.org). I really appreciated their [documentation](https://cookbook.nodered.org) with common use cases. The fastest way to get started is using Docker Compose:

```yaml
version: '3'
services:
  nodered:
    image: nodered/node-red:1.1.2
    container_name: nodered
    ports:
      - "1880:1880"
    environment:
      - TZ=Europe/Lisbon
    volumes:
      - ./nodered-data:/data
```

Node-RED can even be deployed on a Raspberry Pi. If exposing it to the internet make sure to enable authentication and adequate security mechanisms such as a firewall or a reverse proxy. I will briefly describe 2 useful flows created for testing purposes.

## Example flows

The first flow is triggered daily to get the title and description of the [Packt](https://www.packtpub.com/packt/offers/free-learning) daily free e-book through their API. Using the excellent Telegram [Bot API](https://core.telegram.org/bots), a message is sent to a channel so I never forget to download a nice book.

{% asset_img "packt-flow.png" "Packt flow" %}

The second flow exposes an API with the latest projects in [Product Hunt](https://www.producthunt.com). Node-RED already has nodes to scrape HTML using selectors and a template engine to render responses, although some data transformations are required to produce the final JSON.

{% asset_img "producthunt-flow.png" "Producthunt flow" %}

The flows were easy to create and topics like logging, using environment variables, web APIs and scraping were straightforward. To test them, import the code available in this [gist](https://gist.github.com/ruial/69dec3c00b991c3b75da16c16907f87d).

## Summary

Node-RED is a great tool to create small prototypes and personal automations, however I am still not convinced that low-code visual programming tools are fit for production environments due to edge cases and maintainability concerns.
