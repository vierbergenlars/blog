---
layout: post
title: Breaking DNS with consul and dnsmasq forwarding
---

Last month, I broke DNS on a production cluster by attempting to resolve a hostname.

## Discovery

On my laptop, I run Debian 10, with systemd-resolved for DNS.
My VPN client hooks into systemd-resolved to add a private DNS server and local domains (including `*.service.consul` and `*.node.consul`) to the configuration. That makes it very easy to connect to internal services by name. Multiple instances of these services are deployed to achieve high availability, consul DNS will always direct me to a working instance.

Last week I upgraded my laptop from Debian 9 to 10. This also came with a change in defaults for `systemd-resolved`: DNSSEC is now set to `allow-downgrade` by default, which will send additional DNS queries.

After connecting to the VPN, I try to connect to a consul service. This fails because of a DNS resolution failure.

Manually trying to resolve the name with `dig` results in connection timeouts as well.

Then trying with `systemd-resolve` results in an other error message.

```
lars@lars-debian:~$ systemd-resolve xyz.service.consul
xyz.service.consul: resolve call failed: DNSSEC validation failed: no-signature
```

The logs of `systemd-resolved` are filled with these DNSSEC validation failures too.
After a lot of retried attempts, `systemd-resolved` marks the DNS server as non-DNSSEC capable: `Jul 18 14:31:22 lars-debian systemd-resolved[635]: Server 10.88.30.3 does not support DNSSEC, downgrading to non-DNSSEC mode.`

Unfortunately, at this time the damage is already done.

## Walking in circles

What happened? To check for DNSSEC support, `systemd-resolved` sends a DNS query for the `DS` record type of `consul.` to its configured recursor.

In this case, that is the dnsmasq server running at `10.88.10.3`. dnsmasq receives the query, and sees that a specific resolver has been configured for `consul.`. It forwards the DNS query to the consul server at `10.88.10.2` to resolve it.

Consul is configured with a recursor, so DNS queries for other (not `*.consul`) domains would be forwarded to the same dnsmasq server.

However, [the underlying DNS server implementation](https://github.com/miekg/dns/blob/b13675009d59c97f3721247d9efa8914e1866a5b/serve_mux.go#L66-L81) of consul delegates `DS` queries to the parent handler, which are the configured recursors.

The result is that a `DS consul.` query is sent from consul `10.88.10.3` to the dnsmasq server running at `10.88.10.2`.
Upon seeing a query for `consul.`, dnsmasq forwards it back to consul, and round and round in circles it goes.

## Amplification

Dnsmasq has a maximum number of concurrent pending queries, by default 150.
When this limit is reached, it starts answering `SERVFAIL` to all new requests.
This response starts making its way back up the stack of recursive DNS queries.

Upon receiving a `SERVFAIL` answer, dnsmasq will retry the DNS query once more.
There is a stack of 150 pending queries that are waiting for a response of the previous query.
Every time a slot becomes available, it is immediately occupied by a retry.

The result is that very little legitimate DNS queries are able to get through.
The only way to resolve this loop is to break it by restarting either dnsmasq or consul.

## Conclusion

Don't create a loop in your DNS recursors' configuration.
In our case, nothing was querying consul directly anyways, since the dnsmasq server was configured the system resolver.

I [reported an issue](https://github.com/hashicorp/consul/issues/6183) to consul, because either the documentation or the code is incorrect, which lead us to think that our configuration was okay.
