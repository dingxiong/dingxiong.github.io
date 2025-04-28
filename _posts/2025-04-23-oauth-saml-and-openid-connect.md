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
