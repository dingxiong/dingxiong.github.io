---
layout: post
title: Kubernetes -- Kubelet
date: 2025-01-12 12:47 -0800
categories: [kubernetes]
tags: [kubernetes, kubelet]
---

After ssh-ing into a node, below is the processes that are relevant to kubelet.

```
[ec2-user@ip-172-31-74-55 ~]$ ps -ef | grep kubelet
root      3125     1  2 01:28 ?        00:00:08 /usr/bin/kubelet --cloud-provider aws --image-credential-provider-config /etc/eks/ecr-credential-provider/ecr-credential-provider-config --image-credential-provider-bin-dir /etc/eks/ecr-credential-provider --config /etc/kubernetes/kubelet/kubelet-config.json --kubeconfig /var/lib/kubelet/kubeconfig --container-runtime remote --container-runtime-endpoint unix:///run/containerd/containerd.sock --node-ip=172.31.74.55 --pod-infra-container-image=602401143452.dkr.ecr.us-east-2.amazonaws.com/eks/pause:3.5 --v=2 --node-labels=eks.amazonaws.com/nodegroup-image=ami-0af5eb518f7616978,eks.amazonaws.com/capacityType=ON_DEMAND,eks.amazonaws.com/nodegroup=xiong-test,role=xiong-test
root      4136  3380  0 01:28 ?        00:00:00 /csi-node-driver-registrar --csi-address=/csi/csi.sock --kubelet-registration-path=/var/lib/kubelet/plugins/efs.csi.aws.com/csi.sock --v=5
root      4591  4381  0 01:28 ?        00:00:00 /csi-node-driver-registrar --csi-address=/csi/csi.sock --kubelet-registration-path=/var/lib/kubelet/plugins/ebs.csi.aws.com/csi.sock --v=2
```

The content of the config files

```
[ec2-user@ip-172-31-74-55 ~]$ cat /var/lib/kubelet/kubeconfig
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://369A1EB0F8100069D5C96281F5BD706D.gr7.us-east-2.eks.amazonaws.com
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubelet
  name: kubelet
current-context: kubelet
users:
- name: kubelet
  user:
    exec:
      apiVersion: client.authentication.k8s.io/v1beta1
      command: /usr/bin/aws-iam-authenticator
      args:
        - "token"
        - "-i"
        - "evergreen"
        - --region
        - "us-east-2"
```

```
[ec2-user@ip-172-31-74-55 ~]$ cat /etc/kubernetes/kubelet/kubelet-config.json
{
  "kind": "KubeletConfiguration",
  "apiVersion": "kubelet.config.k8s.io/v1beta1",
  "address": "0.0.0.0",
  "authentication": {
    "anonymous": {
      "enabled": false
    },
    "webhook": {
      "cacheTTL": "2m0s",
      "enabled": true
    },
    "x509": {
      "clientCAFile": "/etc/kubernetes/pki/ca.crt"
    }
  },
  "authorization": {
    "mode": "Webhook",
    "webhook": {
      "cacheAuthorizedTTL": "5m0s",
      "cacheUnauthorizedTTL": "30s"
    }
  },
  "clusterDomain": "cluster.local",
  "hairpinMode": "hairpin-veth",
  "readOnlyPort": 0,
  "cgroupDriver": "systemd",
  "cgroupRoot": "/",
  "featureGates": {
    "RotateKubeletServerCertificate": true,
    "KubeletCredentialProviders": true
  },
  "protectKernelDefaults": true,
  "serializeImagePulls": false,
  "serverTLSBootstrap": true,
  "tlsCipherSuites": [
    "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256",
    "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256",
    "TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305",
    "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384",
    "TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305",
    "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384",
    "TLS_RSA_WITH_AES_256_GCM_SHA384",
    "TLS_RSA_WITH_AES_128_GCM_SHA256"
  ],
  "clusterDNS": [
    "10.100.0.10"
  ],
  "kubeAPIQPS": 10,
  "kubeAPIBurst": 20,
  "evictionHard": {
    "memory.available": "100Mi",
    "nodefs.available": "10%",
    "nodefs.inodesFree": "5%"
  },
  "kubeReserved": {
    "cpu": "70m",
    "ephemeral-storage": "1Gi",
    "memory": "442Mi"
  },
  "maxPods": 17
}
```

Notice that the DNS address is the same as kube-dns service.

```
$ k get svc -n kube-system kube-dns
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   2y156d
```
