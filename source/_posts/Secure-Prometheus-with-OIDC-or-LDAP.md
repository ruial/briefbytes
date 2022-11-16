---
title: Secure Prometheus with OIDC or LDAP
date: 2020-11-28 22:42:50
tags: [prometheus, ldap, oidc]
---

When hosting applications for internal use, it is common to only allow connections from private networks. However, in addition to using a VPN, being able to authenticate and authorize users with a centralized identity system is often required for security reasons. I found two good alternatives that I will describe next, along with a monitoring use case.

## Prometheus and Grafana

[Prometheus](https://prometheus.io) is one of the most widely used systems for monitoring and alerting, while [Grafana](https://grafana.com) is very popular to create observability dashboards like this one:

{% asset_img "grafana.png" "Grafana dashboard" %}

Although some applications like Grafana already have support for OAuth or LDAP authentication, others like Prometheus delegate this responsibility to other systems.

## Pomerium

[Pomerium](https://www.pomerium.com/docs) is an identity-aware proxy developed in Go. It can be placed in front of internal services and integrate with identity providers using [OpenID Connect](https://en.wikipedia.org/wiki/OpenID_Connect). Out of the box, it supports identity providers like Google, Github and standard compliant OIDC providers like [node-oidc-provider](https://github.com/panva/node-oidc-provider) or [IdentityServer](https://identityserver4.readthedocs.io/en/latest). Here's a comprehensive [guide](https://www.scottbrady91.com/OpenID-Connect/Getting-Started-with-oidc-provider) to setup a provider.

## Authelia

[Authelia](https://www.authelia.com/docs) is an authentication and authorization server written in Go. It requires integration with a reverse proxy, such as Nginx. Instead of OIDC, LDAP is supported so it can be integrated with Active Directory (AD). On my tests I've used a local users file and Nginx, as it was the simplest approach. I also had success with AD and HAProxy (with LUA plugins), following the official documentation.

## Architecture

Both auth technologies work quite well. It boils down to what you need to integrate with. Some additional features can be added on the reverse proxy or load balancer, like rate limiting or modifying request headers.

I've setup a [demo project](https://github.com/ruial/monitoring-auth-demo) to demonstrate their capabilities that uses the following architecture:

{% asset_img "auth.png" "Auth flow" %}

Besides proxy mode, which passes all traffic through, Pomerium also supports Forward authentication, which causes the reverse proxy to call an endpoint to verify if the user is authorized. User information is then placed on request headers. Both technologies make use of cookie sessions and redirect the user to the login page when not authenticated. They also support [Redis](https://redis.io), which is important to preserve sessions during restarts or if using multiple instances, as you don't usually want to depend on sticky sessions at the load balancer level.

If you are interested in authenticating requests from public internet Github actions runners with internal services inside a private subnet, check out this [OIDC proxy](https://github.com/ruial/actions-oidc-proxy) approach.
