---
title: A not so brief DNS introduction
date: 2021-03-26 19:00:00
tags: [cloud, dns, infrastructure]
---

The Domain Name System (DNS) is one of the most important pieces of the global internet infrastructure. DNS is an hierarchical and decentralized database system to translate host names into IP addresses. In this post I will share some of my knowledge about DNS.

## Domain names and zones

To manage a zone in the public internet, we have to buy a domain name like *briefbytes.com* through a domain registrar. Then we can set the name servers we desire, to manage our DNS records. In my personal case, I've chosen Cloudflare for this.

After having a domain, we can manage the entire zone for that domain. In some companies it is also common to delegate zones. For example, one business unit can manage the root domain and another one could manage all records in the subdomain *it.briefbytes.com*, using different name servers.

Most companies will also create their private zones, which are used for internal DNS resolution on [private computer networks](https://tools.ietf.org/html/rfc1918), with private DNS servers. Two popular DNS servers are Bind9 for Linux and Windows DNS Server for Windows Server. Records can be saved in zone files or any other type of database. Cloud providers, such as Azure, also offer managed DNS solutions like [Azure Private DNS](https://docs.microsoft.com/en-us/azure/dns/private-dns-overview), which is capable of creating DNS records for machines as they are provisioned, without having configure [Dynamic DNS updates](https://tools.ietf.org/html/rfc2136) using a tool like *nsupdate*, as is usually done with self-hosted alternatives.

## DNS records

In the DNS protocol, each resource record must have a name, a type, a time to live (TTL) for caching purposes and some data with the format dependant on the type. More details can be found in [RFC-1035](https://tools.ietf.org/html/rfc1035). The main record types are:

- A (IPv4 host address)

- AAAA (IPv6 host address)

- CNAME (ALIAS for host address)

- SOA (Start of Authority)

- NS (Name Servers)

- MX (Mail Servers)

- SRV (Service records for [service discovery](https://tools.ietf.org/html/rfc2782), however check out [Consul](https://www.consul.io/docs/discovery/dns) and [consul-template](https://github.com/hashicorp/consul-template) they are really awesome for service discovery and work with DNS)

- TXT (Arbitrary text strings, useful for proving domain ownership)

- PTR (Reverse lookup, translates IP to name)

There are some peculiarities about different record types. For example, a CNAME record cannot coexist with other records with the same name, which is why it cannot be created in the apex zone (which is often represented by the name '@'). TXT records are the only ones that can contain multiple values in a single resource record, although it is more common to create multiple records with a single value. PTR records are part of the reverse lookup zone, not the forward lookup zone like the rest. As such, they cannot be changed with your DNS provider because they are managed by the owners of the IP addresses, so they can only be changed by the respective internet service provider (ISP) or cloud provider, unless it's your private zone. There's also wildcard records and glue records.

When two resource records are created with the same name and type, but different data, they are referred to as resource record sets ([RRSet](https://tools.ietf.org/html/rfc2181)). CNAMEs and SOA are exceptions and cannot exist in sets. RRSets enable round-robin load balancing through DNS by rotating the order in which the list of record values is returned, as the first one is often used by clients. However, this is a poor man's load balancer because there is no guarantee that the load will be evenly distributed and if a server goes down, the IP will still be cached until the TTL expires. Round-robin DNS is not an alternative to actual L4 or L7 load balancers, which perform health checks and rotation.

The alternative to RRSets for load balancing at a global level is Anycast. With Anycast, ISPs and cloud providers, using BGP, can publish different routes in different parts of the world to reach certain IPs, making it possible to have multiple machines or network appliances, like load balancers, around the world with the same IP. With Anycast, it is possible to use a solution like [Azure Traffic Manager](https://docs.microsoft.com/en-us/azure/traffic-manager/traffic-manager-routing-methods) to return different DNS responses based on user locations (actually, based on their DNS servers locations, which are also spread across the world using Anycast, so even the single Google DNS IP 8.8.8.8 maps to many servers). If you use *dig* or *nslookup* on the Google domain you will get different answers on different countries.

## How DNS queries are made

When trying to resolve a domain like *briefbytes.com*, the operating system does the following operations to retrieve the associated IP:

1. Check the hosts file (*/etc/hosts* on Linux and Mac OS, inside System32 folder on Windows)

2. Check the system DNS cache (both Windows and Linux have a local DNS cache, however Linux does not, although a local caching DNS server like dnsmasq or systemd-resolved can be installed)

3. Send the query to the configured name servers, which by default is set through DHCP (even on Azure virtual networks), but can be changed (*/etc/resolv.conf* on Linux and Mac OS, network settings on Windows)

When a query is received by a name server, in the case of the Windows DNS server, a widely used DNS server in many enterprises, the following happens:

1. Check the local cache

2. If the server is authoritative for the zone, return authoritative answer

3. Do a recursive query through a conditional forwarder if a name server is set for a certain domain

4. Do a recursive query through a forwarder if a name server is configured (like 8.8.8.8 for Google DNS or 168.63.129.16 in Azure networks)

5. Do iterative queries on the root name servers and follow referrals, if a forwarder is not set or fails (configurable)

A fully qualified domain names (FQDN) always has a dot at the end to avoid issues with relative names (e.g.: `www.briefbytes.com.`). Here's one image from the [Microsoft docs](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/plan/reviewing-dns-concepts), which explains name resolution very nicely:

{% asset_img "dns.gif" "DNS resolution" %}

A recursive query from the root name servers, with the objective of getting an authoritative answer for this blog domain can be done like this:

```sh
nslookup -type=ns com a.root-servers.net
nslookup -type=ns briefbytes.com a.gtld-servers.net
# authoritative answer
nslookup briefbytes.com ned.ns.cloudflare.com

# non authoritative
nslookup -debug briefbytes.com 8.8.8.8
  Non-authoritative answer:
  Name:	briefbytes.com
  Address: 172.67.165.65
  Name:	briefbytes.com
  Address: 104.21.57.175
  Name:	briefbytes.com
  Address: 2606:4700:3033::6815:39af
  Name:	briefbytes.com
  Address: 2606:4700:3034::ac43:a541
```

There are actually 2 name servers configured for my domain in Cloudflare, with some friendly names. Ned is the primary name server and Tani is a secondary name server. Secondary zones are read only copies of the primary (in some cases like Windows DNS server, all domain controllers are actually primary servers, since writes can happen in any server and data is saved to Active Directory and replicated). By doing a SOA query on each name server we can see a serial number. This value is incremented on the primary server when updates are made. The secondary server periodically checks the serial number and does an incremental [zone transfer](https://tools.ietf.org/html/rfc5936) when needed, to sync the records. DNS protocol supports TCP, although UDP is preferred for small requests (up to 512 bytes).

## Security and privacy

DNS traffic is not encrypted, so even if you change your default name servers provided by your ISP, they are able to know which sites you visit based on your DNS queries. In my country, ISPs are forced by the government censorship to block certain domain names like LibGen, so changing them is essential for many people. Let's hope they never decide to block IP address ranges and take down entire regions of the internet.

To prevent ISPs from snooping our traffic to sell our data, DNS over HTTPS ([DoH](https://tools.ietf.org/html/rfc8484)) seems to be the future. However, keep in mind that the actual DNS servers will still have the data. Browser vendors like [Firefox](https://blog.mozilla.org/blog/2020/02/25/firefox-continues-push-to-bring-dns-over-https-by-default-for-us-users) pushing DoH, however DNS should be a global operating system setting and not set by individual browsers or other applications. I want to set [Pi-Hole](https://pi-hole.net) as a DNS server to block ads on the network and propagate it to all machines via router's DHCP without additional configuration, this is what standards are for.

Regarding DNS security there's also [DNSSEC](https://tools.ietf.org/html/rfc4033) to verify the integrity of DNS records but it is complex and not widely used. To authenticate updates and zone transfers on a DNS database there's [TSIG](https://tools.ietf.org/html/rfc2845), which also guarantees message integrity using shared secrets.

With DNS management comes public key infrastructure ([PKI](https://tools.ietf.org/html/rfc2510)) certificate management, as we want to issue certificates for domain names. On public DNS zones in the internet, we have to pay for the certificates, unless we use [Let's Encrypt](https://letsencrypt.org), as they must be signed by a trusted certificate authority (CA). For private DNS zones on our network, we can create a new internal CA and add the root certificate on all the machines we want the issued certificates to be trusted. To avoid managing internal CAs, some companies buy domains for private DNS resolution, instead of using [reserved domains](https://tools.ietf.org/html/rfc8244) like '.local'. Then they usually buy wildcard certificates and install them with configuration management tools on the web servers and any other computer should trust these certificates without additional work.

The DNS protocol is a critical part of any infrastructure. Even the machines that have to be locked down for compliance reasons will often have DNS traffic enabled, which can be used for [data exfiltration](https://blogs.akamai.com/2017/09/introduction-to-dns-data-exfiltration.html). Using DNS for this is far from ideal, as each outbound lookup query will have a maximum of 255 bytes and order of arrival in unpredictable. From a security perspective, it makes sense to monitor DNS traffic. When exposing a DNS server on the internet, recursive queries should be disabled, so that only authoritative answers are returned.

## Conclusion

This was a not so brief introduction to DNS. One last advice is if you ever need to develop software related with DNS, check the following Go libraries:

- https://github.com/miekg/dns - to create DNS clients or servers

- https://github.com/bodgit/tsig - perform dynamic updates with [GSS-TSIG](https://www.ietf.org/rfc/rfc3645.txt) on Windows Server with 'secure only' updates
