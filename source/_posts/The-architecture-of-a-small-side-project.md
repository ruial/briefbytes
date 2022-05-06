---
title: The architecture of a small side project
date: 2022-01-05 21:05:00
tags: [architecture, cloud]
---

There is a huge amount of people that like to develop side projects. I mostly do it because I want to learn new things, but gaining some passive income is also an attractive idea. In the last months I launched [Ad Pruner](https://www.producthunt.com/posts/ad-pruner), an OpenVPN and Pi-hole as a service, and I will describe different architecture and infrastructure alternatives.

## Domain registration and DNS

This is the only part of my setup that I had to pay. To have an online business, be able to receive/send emails and maintain complete ownership of our content, a domain name is required, preferably .com. It's one of the reasons why I publish this blog on my own domain instead of Medium. Regarding registration, my favorite providers are Namesilo and Namecheap. I prefer to keep registration decoupled of DNS and hosting providers (there are the occasional horror stories of accounts closed due to bad fraud detection and bad support). However, Cloudflare is the cheapest registrar and they also offer free CDN, DNS and DDoS protection, so I always tend to use their free tier and never had any issues.

## Hosting

For small projects it doesn't make sense to spend a lot on hosting infrastructure. As engineers we should try to balance costs and quality. Most of the time, a cheap VM with 512MB of RAM and a throttled vCPU will work fine for a couple of low traffic websites, a small database, cache and batch jobs.

There are plenty of cheap VM/VPS under the 5$ range, like Amazon Lightsail, Vultr or Hetzner. However, we can go even cheaper with Oracle's free cloud offer which includes 2 AMD VMs with 1GB RAM, up to 4 ARM VMs, 10GB object storage, a load balancer, multiple IPv4 addresses, among other things. It does sound too good to be true, but I've been using them for months without issues. GCP always free tier also includes 1 small VM in the US and other useful things.

To receive emails on your domain, the best free solution is Zoho Mail. ImprovMX is an alternative if you are only interested in forwarding. For sending bulk email I would use Sendgrid free tier.

## Programming language and frameworks

This is mostly personal opinion, people either tend to use what they are most familiar with or what they want to learn. For rapid development, Python and Django beat most alternatives out there. Developing a separate API and a JavaScript front end will always take longer. The downside of Python is that it will consume more computational resources than alternatives like Go or Node.js. I would recommend writing tests and using types, as they help maintenance of bigger projects. CI is easy and can be free nowadays with Github Actions. Deployments should also be automated with Ansible and Docker or similar.

## Analytics, logging and monitoring

It is useful to keep track of the application behaviour without having to access the server. For analytics, the most common approach is to use a solution like Google Analytics or the open-source privacy oriented Plausible Analytics. You can also complement these with web server logs.

For metrics and log analytics, my favorite stack is Elasticsearch+Kibana and Prometheus+Grafana. However, it does not make sense to host these yourself until you grow to a certain size. Elasticsearch is particularly hungry for RAM. I had a good experience with Sematext free tier, which uses the Elastic stack, but opting for Grafana Cloud and Logz.io are also valid approaches. At scale, it is common to have agents like Filebeat to send application logs to Kafka as a buffer, to prevent overloading Elastic and doing additional processing in Logstash. To build status pages I'm using Freshping for HTTP and TCP checks, StatusCake is popular too.

## Scaling when traction is gained

Your project is growing, your data is valuable and downtime is no longer acceptable, now what? At a small scale, configuring individual VMs with Ansible and Docker works well, you can also scale vertically to a certain point. The current state of the art for horizontal scaling and deployment of applications is Kubernetes. The managed cloud offers like AKS from Azure help reduce the cost of ownership by taking care of some operational tasks. It is also possible to host stateful databases in Kubernetes, but managed services are easier. Here's a diagram of a common N-tier architecture:

{% asset_img "web-architecture.png" "Web architecture" %}

With a setup like this you can scale well, provided that your application is stateless. The decision of rolling your own or using external services should be considered carefully, to reduce operational burden and keep costs low. On a small scale, 1 or 2 cheap VMs and the free services I suggested will work fine for most projects, but having the knowledge of how to scale is always useful.
