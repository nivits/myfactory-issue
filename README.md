# Myfactory network issue
## Management summary
At [nivits](https://nivits.com), we discovered that subsequent HTTP requests to [Myfactory](https://myfactory.ch) cloud endpoints are failing if the client chooses a TCP source port that was recently used. When using APIs, a lot of automated requests are made and it is likely and legitimate that a client re-uses the same source port for a new connection. However, Myfactory kind of blocks these requests, which result in timeouts.

This document provides proof of a misconfiguration and requires no custom code, but simple, plain HTTP requests without any prerequisites.

## Proof
We've gathered several PCAPs clearly identifying the issue. To reproduce the issue in the most simple way, we only need a Linux host with `netfilter` loaded and the `iptables` userspace tools installed.

Typically, the OS's TCP/IP stack is responsible for choosing an ephemeral port, so it gets randomly assigned and typically is out of control of the user or software. However, with `iptables` we can "force" to use a specific outgoing port.

**Step 1:**
Set the ephemeral port for HTTPS to 61000:

```iptables -t nat -A POSTROUTING -p tcp --dport 443 -j SNAT --to :61000
```

**Step 2:**
Fetch a Myfactory URL:

```
dmanser@tinos ~ curl -I https://public02.myfactory.cloud/saas/

HTTP/2 200
cache-control: private
content-length: 3962
content-type: text/html; charset=utf-8
expires: Sun, 07 Apr 2024 09:20:45 GMT
server: Microsoft-IIS/10.0
x-aspnet-version: 4.0.30319
date: Sun, 07 Apr 2024 09:21:44 GMT
```

We can see the web server responded with a `200` status code. Everything is fine.

**Step 3:**
After some seconds, issue the same command again:

```
dmanser@tinos ~ curl -I https://public02.myfactory.cloud/saas/

curl: (28) Failed to connect to public02.myfactory.cloud port 443 after 130969 ms: Connection timed out
```
