# Myfactory network issue
## Management summary
At [nivits](https://nivits.com), we discovered that subsequent HTTP requests to [Myfactory](https://myfactory.ch) cloud endpoints are failing if the client chooses a TCP source port that was recently used. When using APIs, a lot of automated requests are made and it is likely and legitimate that a client re-uses the same source port for a new connection. However, Myfactory kind of blocks these requests, which result in timeouts.

This document provides proof of a misconfiguration and requires no custom code, but simple, plain HTTP requests without any prerequisites.

## Proof
We've gathered several PCAPs clearly identifying the issue. To reproduce the issue in the most simple way, we only need a Linux host with `netfilter` loaded and the `iptables` userspace tools installed.

Typically, the OS's TCP/IP stack is responsible for choosing an ephemeral port, so it gets randomly assigned and typically is out of control of the user or software. However, with `iptables` we can "force" to use a specific outgoing port.

**Step 1:**
Set the ephemeral port for HTTPS to 61000:

```
dmanser@tinos ~ sudo iptables -t nat -A POSTROUTING -p tcp --dport 443 -j SNAT --to :61000
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

The server fails to accept the request (does not send a `SYN-ACK`).

**Step 4:**
Try with other websites, and they are working fine:

```
dmanser@tinos ~ curl -I https://nivits.com

HTTP/1.1 200 OK
Date: Sun, 07 Apr 2024 09:52:42 GMT
Server: Apache/2.4.25 (Debian)
Strict-Transport-Security: max-age=31536000; includeSubdomains
X-Powered-By: PHP/7.3.29
Content-Type: text/html; charset=UTF-8
Access-Control-Allow-Origin: *
```

and again:

```
dmanser@tinos ~ curl -I https://nivits.com

HTTP/1.1 200 OK
Date: Sun, 07 Apr 2024 09:52:45 GMT
Server: Apache/2.4.25 (Debian)
Strict-Transport-Security: max-age=31536000; includeSubdomains
X-Powered-By: PHP/7.3.29
Content-Type: text/html; charset=UTF-8
Access-Control-Allow-Origin: *
```

Or, try with Youtube:

```
dmanser@tinos ~ curl -I https://www.youtube.com

HTTP/2 200
content-type: text/html; charset=utf-8
x-content-type-options: nosniff
cache-control: no-cache, no-store, max-age=0, must-revalidate
pragma: no-cache
expires: Mon, 01 Jan 1990 00:00:00 GMT
date: Sun, 07 Apr 2024 09:57:31 GMT
content-length: 492345
strict-transport-security: max-age=31536000
x-frame-options: SAMEORIGIN
report-to: {"group":"youtube_main","max_age":2592000,"endpoints":[{"url":"https://csp.withgoogle.com/csp/report-to/youtube_main"}]}
origin-trial: AvC9UlR6RDk2crliDsFl66RWLnTbHrDbp+DiY6AYz/PNQ4G4tdUTjrHYr2sghbkhGQAVxb7jaPTHpEVBz0uzQwkAAAB4eyJvcmlnaW4iOiJodHRwczovL3lvdXR1YmUuY29tOjQ0MyIsImZlYXR1cmUiOiJXZWJWaWV3WFJlcXVlc3RlZFdpdGhEZXByZWNhdGlvbiIsImV4cGlyeSI6MTcxOTUzMjc5OSwiaXNTdWJkb21haW4iOnRydWV9
cross-origin-opener-policy: same-origin-allow-popups; report-to="youtube_main"
permissions-policy: ch-ua-arch=*, ch-ua-bitness=*, ch-ua-full-version=*, ch-ua-full-version-list=*, ch-ua-model=*, ch-ua-wow64=*, ch-ua-form-factor=*, ch-ua-platform=*, ch-ua-platform-version=*
p3p: CP="This is not a P3P policy! See http://support.google.com/accounts/answer/151657?hl=de for more info."
server: ESF
x-xss-protection: 0
set-cookie: YSC=P1vk-BPs2g4; Domain=.youtube.com; Path=/; Secure; HttpOnly; SameSite=none
set-cookie: __Secure-YEC=Cgt0WUYxWDBjNjZCcyiL3MmwBjIKCgJDSBIEGgAgMg%3D%3D; Domain=.youtube.com; Expires=Wed, 07-May-2025 09:57:30 GMT; Path=/; Secure; HttpOnly; SameSite=lax
set-cookie: VISITOR_PRIVACY_METADATA=CgJDSBIEGgAgMg%3D%3D; Domain=.youtube.com; Expires=Wed, 07-May-2025 09:57:31 GMT; Path=/; Secure; HttpOnly; SameSite=none
set-cookie: VISITOR_INFO1_LIVE=; Domain=.youtube.com; Expires=Mon, 12-Jul-2021 09:57:31 GMT; Path=/; Secure; HttpOnly; SameSite=none
alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
```

and again, it works:

```
dmanser@tinos ~ curl -I https://www.youtube.com

HTTP/2 200
content-type: text/html; charset=utf-8
x-content-type-options: nosniff
cache-control: no-cache, no-store, max-age=0, must-revalidate
pragma: no-cache
expires: Mon, 01 Jan 1990 00:00:00 GMT
date: Sun, 07 Apr 2024 09:57:35 GMT
content-length: 481978
x-frame-options: SAMEORIGIN
strict-transport-security: max-age=31536000
report-to: {"group":"youtube_main","max_age":2592000,"endpoints":[{"url":"https://csp.withgoogle.com/csp/report-to/youtube_main"}]}
origin-trial: AvC9UlR6RDk2crliDsFl66RWLnTbHrDbp+DiY6AYz/PNQ4G4tdUTjrHYr2sghbkhGQAVxb7jaPTHpEVBz0uzQwkAAAB4eyJvcmlnaW4iOiJodHRwczovL3lvdXR1YmUuY29tOjQ0MyIsImZlYXR1cmUiOiJXZWJWaWV3WFJlcXVlc3RlZFdpdGhEZXByZWNhdGlvbiIsImV4cGlyeSI6MTcxOTUzMjc5OSwiaXNTdWJkb21haW4iOnRydWV9
cross-origin-opener-policy: same-origin-allow-popups; report-to="youtube_main"
permissions-policy: ch-ua-arch=*, ch-ua-bitness=*, ch-ua-full-version=*, ch-ua-full-version-list=*, ch-ua-model=*, ch-ua-wow64=*, ch-ua-form-factor=*, ch-ua-platform=*, ch-ua-platform-version=*
p3p: CP="This is not a P3P policy! See http://support.google.com/accounts/answer/151657?hl=de for more info."
server: ESF
x-xss-protection: 0
set-cookie: YSC=gqpjbJVBR6c; Domain=.youtube.com; Path=/; Secure; HttpOnly; SameSite=none
set-cookie: __Secure-YEC=Cgs3S1pVbUNzUFdWRSiP3MmwBjIKCgJDSBIEGgAgIg%3D%3D; Domain=.youtube.com; Expires=Wed, 07-May-2025 09:57:34 GMT; Path=/; Secure; HttpOnly; SameSite=lax
set-cookie: VISITOR_PRIVACY_METADATA=CgJDSBIEGgAgIg%3D%3D; Domain=.youtube.com; Expires=Wed, 07-May-2025 09:57:35 GMT; Path=/; Secure; HttpOnly; SameSite=none
set-cookie: VISITOR_INFO1_LIVE=; Domain=.youtube.com; Expires=Mon, 12-Jul-2021 09:57:35 GMT; Path=/; Secure; HttpOnly; SameSite=none
alt-svc: h3=":443"; ma=2592000,h3-29=":443"; ma=2592000
```