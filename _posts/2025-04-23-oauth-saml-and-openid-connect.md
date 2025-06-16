---
layout: post
title: OAuth, SAML and OpenID Connect
date: 2025-04-23 12:16 -0700
categories: [security]
tags: [oauth, saml, oidc]
---

> OAuth 2.0 is an authorization framework, not an authentication protocol

```
+-------------+                               +------------------+
|             |--(1) Auth Request------------>|  Authorization   |
|             |   (openid scope)              |     Server       |
|             |                               |  (e.g., Google)  |
|  User/Browser|<-(2) Login & Consent----------|                  |
|             |                               +------------------+
|             |--(3) Authorization Code------->|                  |
|             |                               |                  |
|             |<-(4) ID Token + Access Token--|                  |
+-------------+                               +------------------+
       |
       | Use ID Token to identify user
       |
       v
+-------------+
|   Your App  |
| (verifies   |
|  ID Token)  |
+-------------+
```

> At the risk of over-simplification, OpenID Connect is a rewrite of SAML using
> OAuth 2.0

1. OIDC is essentially an extension of OAUTH 2.0.

## OKTA

### Integration Network

### Experiment.

1. Register an account. Must use a company email.
2. Add a user with personal email.
3. Then two urls:
   - https://dev-06484947.okta.com/
   - https://dev-06484947-admin.okta.com

## How to make okta authenticate less often.

First, change okta session in Chrome from session level to permanent and set a
long expiration time. This step is make sure the same session could be used
after Chrome is closed and session token will be lost because it lives only
inside memory. ![okta session](/assets/images/okta_session_token.png) Btw, the
max age of a session is 400 days in Chrome. When I tried to set a expiration
date beyond that, chrome auto corrected me.

Second, write a cron job to auto refresh the token on the server side.

```
$ curl -L 'https://ziphq.okta.com/api/v1/sessions/me/lifecycle/refresh' -H 'Cookie: idx=<...>'

{
    ...
    "createdAt": "2025-06-16T02:48:21.000Z",
    "expiresAt": "2025-06-19T02:48:10.000Z",
    "status": "ACTIVE",
    ...
}

```

UPDATE!!!

This approach does not work for idx token. See
[source](https://developer.okta.com/docs/guides/oie-upgrade-sessions-api/main/).
Man, I need to be OKTA admin to configure the whole crap.

In principle, I can write a script that read the `idx` out from Chrome session
cookie stored in disk, then use okta `sessions/me` api to check the expiration
status, and use pupeeteer to generate a new token and saves back to Chrome
session cookie sqlite database on disk. However, session cookie is encrypted
and to encrypt it I need Apple’s Keychain. See
[this](https://www.cyberark.com/resources/threat-research-blog/the-current-state-of-browser-cookies)
and
[this](https://gist.github.com/creachadair/937179894a24571ce9860e2475a2d2ec). I
definitely do not want to save it in plain text in my cron script because if
someone cracks my laptop, then he/she may steal my bank account session cookie.

A funny story is that 30 min later after I ran
`security find-generic-password -w -a Chrome -s 'Chrome Safe Storage'`, a
security guy reached out to me asking what I am doing. :(
