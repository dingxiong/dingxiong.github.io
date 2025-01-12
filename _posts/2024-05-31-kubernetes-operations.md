---
layout: post
title: Kubernetes -- Operations
date: 2024-05-31 21:06 -0700
categories: [kubernetes]
tags: [kubernetes]
---

### Increase the disk size of StatefulSet PVC

```bash
kubectl edit pvc <pvc-name> -n <namespace>
kubectl delete statefulset --cascade=orphan <statefulset_name> -n <namespace>
# Then you can reapply the helm chart
```

## Namespace Stuck in Terminating State

1. `kubectl get namespace <ns> -o json > ~/tmp/tmp.json`
2. Edit above json and delete the items in the finalizer list.
3. Run `kubectl proxy` in a new tab.
4. `curl -k -H "Content-Type: application/json" -X PUT --data-binary @tmp.json http://127.0.0.1:8001/api/v1/namespaces/<ns>/finalize`
