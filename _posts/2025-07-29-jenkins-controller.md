---
layout: post
title: Jenkins -- Controller
date: 2025-07-29 16:41 -0700
categories: [devops, cicd]
tags: [devops, jenkins]
---

```
| Annotation  | Purpose                        | Resulting Endpoint                           |
| ----------- | ------------------------------ | -------------------------------------------- |
| `@Exported` | Included in `/api/json` output | `/.../api/json?tree=fieldName`               |
| `doXYZ()`   | Bound to URL endpoint          | `/.../xyz`                                   |
| `getXYZ()`  | Usually bound as a property    | `/.../xyz/` (sometimes), or `api/json` field |

```

[Jenkins sample build](/assets/raw/jenkins_build.xml.txt)
