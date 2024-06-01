---
layout: post
title: Kubernetes Network
date: 2024-04-13 22:28 -0700
categories: [kubernetes]
tags: [kubernetes, cgroup, network]
---

Network itself is a complicated subject. It becomes even tougher inside
kubernetes. This doc is meant to introduce a lot of network concepts and
components inside kubernetes.

I highly recommend Mark Betz's
[3-series blogs](https://medium.com/google-cloud/understanding-kubernetes-networking-pods-7117dd28727)
on k8s network.

Basically, in the same host, pod to pod networking is realized through virtual
network interface bridge and veth. For pod to pod communication that across
host, k8s has a service type that configure iptables NAT table. For traffic
from outside of the cluster, then k8s has PortNode, LoadBalancer and Ingresss
types.

I am shocked when finding out the kubernetes uses SPDY protocol today. SPDY is
a abandoned protocol and is superseded by HTTP2. There is proposal to use
websocket to replace SPDY in kubernetes. See
[this ticket](https://github.com/kubernetes/kubernetes/issues/89163). But It
hasn't been done yet!

## Kube proxy

Mayank Shah wrote a clear explanation of
[how kube-proxy works](https://mayankshah.dev/blog/demystifying-kube-proxy/).
In a high level, kube-proxy is a kubernetes component responsible for traffic
routing. It is a pod installed in each node that listens for service, Endpoint,
EndpointSlice and node updates and then rewrites iptable rules of the node that
it is running.

Let's go deeper into the code!

Below is what is running inside a kube-proxy pod. No need to argue, this must a
cobra program!

```
# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 00:02 ?        00:00:01 kube-proxy --v=2 --config=/var/lib/kube-proxy-config/config
```

The entry point is
[here](https://github.com/kubernetes/kubernetes/blob/567d1a2e226f75a6c2bc2b4a0df514d80e049637/cmd/kube-proxy/app/server.go#L659).
It defines a `ServiceConfig`, `EndpointConfig`, `EndpointSliceConfig` and
`NodeConfig` that listens for changes for these types of resources. Continuing
reading the handlers registered for these configs, you will find that the main
function that updates iptable rules is
[here](https://github.com/kubernetes/kubernetes/blob/dbb6c77de41d947782cc97ca9a4c415c42e1234d/pkg/proxy/iptables/proxier.go#L806).
This is long function and I don't want to explain all details inside. Let's use
an example to illustrate what this function does.

Namesapce `test-jimmy` is namespace in one of ziphq's system cluster. Below is
what is running inside it: a service and two pods belonging to this service.

```
$ k get service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
hello-world   ClusterIP   10.100.73.189   <none>        80/TCP    670d

$ k get pods -o wide
NAME                           READY   STATUS    RESTARTS   AGE   IP              NODE                                          NOMINATED NODE   READINESS GATES
hello-world-0                  1/1     Running   0          67d   172.31.39.245   ip-172-31-33-184.us-east-2.compute.internal   <none>           <none>
hello-world-78c6ff5585-54px7   1/1     Running   0          67d   172.31.80.113   ip-172-31-81-18.us-east-2.compute.internal    <none>           <none>
```

And below is part of the iptables I grabbed from an ec2 instance inside this
cluster. Irrelevant parts are removed.

```
# iptables -t nat -L

Chain KUBE-SERVICES (2 references)
target     prot opt source               destination
...
KUBE-SVC-DV4GL2IIJHQSYSNI  tcp  --  anywhere             ip-10-100-73-189.us-east-2.compute.internal  /* test-jimmy/hello-world: cluster IP */ tcp dpt:http
...

Chain KUBE-SEP-4SENZW6K2YNAYEVT (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  ip-172-31-39-245.us-east-2.compute.internal  anywhere             /* test-jimmy/hello-world: */
DNAT       tcp  --  anywhere             anywhere             /* test-jimmy/hello-world: */ tcp to:172.31.39.245:80

Chain KUBE-SEP-3UHZLYXV5R5LGKHL (1 references)
target     prot opt source               destination
KUBE-MARK-MASQ  all  --  ip-172-31-80-113.us-east-2.compute.internal  anywhere             /* test-jimmy/hello-world: */
DNAT       tcp  --  anywhere             anywhere             /* test-jimmy/hello-world: */ tcp to:172.31.80.113:80

Chain KUBE-SVC-DV4GL2IIJHQSYSNI (1 references)
target     prot opt source               destination
KUBE-SEP-4SENZW6K2YNAYEVT  all  --  anywhere             anywhere             /* test-jimmy/hello-world: */ statistic mode random probability 0.50000000000
KUBE-SEP-3UHZLYXV5R5LGKHL  all  --  anywhere             anywhere             /* test-jimmy/hello-world: */
```

First, the definitions of `KUBE-SEP` and `KUBE-SVC` are
[here](https://github.com/kubernetes/kubernetes/blob/dbb6c77de41d947782cc97ca9a4c415c42e1234d/pkg/proxy/iptables/proxier.go#L689-L693).
`SVC` stands for service. `SEP` stands for service endpoint.

Chain `KUBE-SVC-DV4GL2IIJHQSYSNI` has two targets, both targets are a service
endpoint rule. It basically says if there is traffic for hello-world service,
please route it the one of these two targets. You may wonder
`KUBE-SVC-hello-world` is a much better chain name, why use some kind of random
char sequence instead, then read
[this](https://github.com/kubernetes/kubernetes/blob/dbb6c77de41d947782cc97ca9a4c415c42e1234d/pkg/proxy/iptables/proxier.go#L677-L681)
please. Also, notice at the top there is a chain `KUBE-SERVICES` that point
this hello-world service to the correct ip address `10.100.73.189`. The
relevant code for writing service chain rule is
[here](https://github.com/kubernetes/kubernetes/blob/dbb6c77de41d947782cc97ca9a4c415c42e1234d/pkg/proxy/iptables/proxier.go#L1159-L1186).

For each service endpoint, they have a `DNAT` rule. Using the first one as
example, it says for traffic to this chain, please redirect it to ip address
`172.31.39.245:80` which is the ip address of the corresponding pod. The
relevant code for writing this iptable rule is
[here](https://github.com/kubernetes/kubernetes/blob/dbb6c77de41d947782cc97ca9a4c415c42e1234d/pkg/proxy/iptables/proxier.go#L1022-L1053).

What will happen if we delete iptable rules? For example, for below service
chain.

```
[ec2-user@ip-172-31-95-245 ~]$ sudo iptables -t nat -L KUBE-SVC-K33YM7QL3UKVXTHK --line-numbers
Chain KUBE-SVC-K33YM7QL3UKVXTHK (1 references)
num  target     prot opt source               destination
1    KUBE-SEP-XWNWQKCBWXXPVOLP  all  --  anywhere             anywhere             /* sample/helloworld:http -> 172.31.55.43:5000 */ statistic mode random probability 0.50000000000
2    KUBE-SEP-TXHPX2YG4URKEKRQ  all  --  anywhere             anywhere             /* sample/helloworld:http -> 172.31.83.230:5000 */
```

If I run `sudo iptables -t nat -D KUBE-SVC-K33YM7QL3UKVXTHK 1`. It shows

```
[ec2-user@ip-172-31-95-245 ~]$ sudo iptables -t nat -L KUBE-SVC-K33YM7QL3UKVXTHK --line-numbers
Chain KUBE-SVC-K33YM7QL3UKVXTHK (1 references)
num  target     prot opt source               destination
1    KUBE-SEP-TXHPX2YG4URKEKRQ  all  --  anywhere             anywhere             /* sample/helloworld:http -> 172.31.83.230:5000 */
```

After about a few seconds, I list the rules again

```
[ec2-user@ip-172-31-95-245 ~]$ sudo iptables -t nat -L KUBE-SVC-K33YM7QL3UKVXTHK --line-numbers
Chain KUBE-SVC-K33YM7QL3UKVXTHK (1 references)
num  target     prot opt source               destination
1    KUBE-SEP-XWNWQKCBWXXPVOLP  all  --  anywhere             anywhere             /* sample/helloworld:http -> 172.31.55.43:5000 */ statistic mode random probability 0.50000000000
2    KUBE-SEP-TXHPX2YG4URKEKRQ  all  --  anywhere             anywhere             /* sample/helloworld:http -> 172.31.83.230:5000 */
```

It shows up again! Kube-proxy is very resilient.

To sum up, kube-proxy is a component that keeps iptable rules in sync with the
topology of the cluster. If you are critical reader, you may realize that in
the iptables above, the destination of `DNAT` rule is an IP address, and the
some places use `ip-xxx.us-east-2.compute.internal` DNS names. Then the
questions is how kube-proxy get these ip addresses and how dns is resolved. The
answer lies on CNI and coreDns.

## CNI

In the `kube-proxy` section, we discussed how kube-proxy maintains iptable
rules inside nodes. Iptable rules tells which ip address to route the traffic
to. But wait a second, where are these ip addresses from? For pod-to-pod
communication, each pod should have an IP address assigned. Then who assigns
these ip addresses?

This attributes to CNI, i.e., Container Network Interface. CNI is a standard,
and CNI plugin implements it. In a nutshell, CNI plugin is called by kubelet
when a pod is created and destroyed. When a pod is created, kubelet sends a
`ADD` request to CNI plugin, and CNI plugin returns a valid IP address. When a
pod is destroyed, kubelet sends a `DELETE` request to CNI plugin to reclaim the
IP address. CNI is simple, besides `ADD` and `DELETE`, there are only two other
endpoints: `CHECK` and `VERSION`.

Sorry, I want to dive a little bit more into how CNI works. You can skip this
paragraph if you are not interested. Requests are sent to CNI plugin by `stdin`
together with some environment variables. Response is collected by `stdout`. It
sounds incredibly simple, right? This reminds me of the old days of `cgi`
protocol in web development. On one hand, it provides extreme flexibility for
implementation. On the other hand, it makes logging a little bit inconvenient.
Be careful of not sending any log to `stdout`! This also reminds me about
goreplay middleware interface.

In AWS EKS, the CNI implementation is
[amazon-vpc-cni-k8s](https://github.com/aws/amazon-vpc-cni-k8s).
[Its proposal](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/cni-proposal.md)
is a good starting point to understand the purpose of CNI.

> 1. All containers can communicate with all other containers without NAT
> 2. All nodes can communicate with all containers (and vice-versa) without NAT
> 3. The IP address that a container sees itself as is the same IP address that
>    others see it as

`NAT` stands for Network Address Translation. Another
[document](https://github.com/aws/amazon-vpc-cni-k8s/blob/master/docs/eni-and-ip-target.md)
summarizes how this plugin works.

> The AWS VPC CNI has two components. The CNI binary, /opt/cni/bin/aws-cni, and
> ipamd running as a Kubernetes daemonset called aws-node, adding a pod on
> every node that keeps track of all ENIs and IPs attached to the instance. The
> CNI binary is invoked by the kubelet when a new pod gets added to, or an
> existing pod removed from the node.

Let's dive into these two components.

### aws-cni

We need to ssh into the host to see where aws-cni is.

```
[ec2-user@ip-172-31-74-55 ~]$ ll /opt/cni/bin/
total 103440
-rwxr-xr-x 1 root root 33715560 Feb  3 01:28 aws-cni
-rwxr-xr-x 1 root root    16318 Feb  3 01:28 aws-cni-support.sh
-rwxr-xr-x 1 root root  4159518 May 13  2020 bandwidth
-rwxr-xr-x 1 root root  4671647 May 13  2020 bridge
-rwxr-xr-x 1 root root 12124326 May 13  2020 dhcp
-rwxr-xr-x 1 root root  5945760 May 13  2020 firewall
-rwxr-xr-x 1 root root  3069556 May 13  2020 flannel
-rwxr-xr-x 1 root root  4174394 May 13  2020 host-device
-rwxr-xr-x 1 root root  3614480 May 13  2020 host-local
-rwxr-xr-x 1 root root  4314598 May 13  2020 ipvlan
-rwxr-xr-x 1 root root  3209463 May 13  2020 loopback
-rwxr-xr-x 1 root root  4389622 May 13  2020 macvlan
-rwxr-xr-x 1 root root  3939867 Feb  3 01:28 portmap
-rwxr-xr-x 1 root root  4590277 May 13  2020 ptp
-rwxr-xr-x 1 root root  3392826 May 13  2020 sbr
-rwxr-xr-x 1 root root  2885430 May 13  2020 static
-rwxr-xr-x 1 root root  3356587 May 13  2020 tuning
-rwxr-xr-x 1 root root  4314446 May 13  2020 vlan
```

```
[ec2-user@ip-172-31-74-55 ~]$ cat /etc/cni/net.d/10-aws.conflist
{
  "cniVersion": "0.3.1",
  "name": "aws-cni",
  "plugins": [
    {
      "name": "aws-cni",
      "type": "aws-cni",
      "vethPrefix": "eni",
      "mtu": "9001",
      "pluginLogFile": "/var/log/aws-routed-eni/plugin.log",
      "pluginLogLevel": "Debug"
    },
    {
      "type": "portmap",
      "capabilities": {"portMappings": true},
      "snat": true
    }
  ]
}
```

You see that there are two plugins registered. `aws-cni` is the main one. The
other one `protmap` is used to support `hostPort`. It is an official plugin
published by CNI committee. More details can be found here
https://www.cni.dev/plugins/current/meta/portmap/

The code for `aws-cni` is
[here](https://github.com/aws/amazon-vpc-cni-k8s/blob/d2f240df492fb567508b0154e8bd167b6346445a/cmd/routed-eni-cni-plugin/cni.go#L516).
It is quite simple. It implements `ADD` and `DELTETE` endpoints of CNI by
calling a gRPC service `ipamd`.

### ipamd

`ipamd` stands for IP address manager daemon. It runs in the `kube-system`
namespace in `aws-node-xxx` pod. Below lists the processes running inside a
`aws-note-xx` pod.

```
bash-4.2# ps -ef
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  0 06:04 ?        00:00:00 bash /app/entrypoint.sh
root         9     1  0 06:04 ?        00:00:02 ./aws-k8s-agent
root        10     1  0 06:04 ?        00:00:00 tee -i aws-k8s-agent.log
root      3118     0  0 06:22 pts/0    00:00:00 bash
root      3470  3118  0 06:23 pts/0    00:00:00 ps -ef
```

The entry point starts from
[here](https://github.com/aws/amazon-vpc-cni-k8s/blob/d2f240df492fb567508b0154e8bd167b6346445a/cmd/aws-k8s-agent/main.go#L80).
Most details are in
[this file](https://github.com/aws/amazon-vpc-cni-k8s/blob/d2f240df492fb567508b0154e8bd167b6346445a/pkg/ipamd/rpc_handler.go#L63)
and are boring. Basically, each node reserves some ENI (AWS Elastic Network
Interface) and a pool of private IP addresses when a node starts. One ENI will
be assigned to the `eth0` network interface. Usually, a node has two private ip
addresses, i.e., `eth0` and `eth1`, and one more ENI will be assigned to
`eth1`. When a pod is attached to this node, `ipamd` creates a `veth` pair and
assigns an private ip address to the `veth` inside this pod. We use an example
to illustrate what happens under the hood.

Below I ssh into a node and list all network interfaces in this node.

```
[ec2-user@ip-172-31-74-55 ~]$ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:2b:8b:69:f9:54 brd ff:ff:ff:ff:ff:ff
    inet 172.31.74.55/20 brd 172.31.79.255 scope global dynamic eth0
       valid_lft 3328sec preferred_lft 3328sec
    inet6 fe80::42b:8bff:fe69:f954/64 scope link
       valid_lft forever preferred_lft forever
3: eni38fb9ccdc92@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 52:1f:ed:c6:db:64 brd ff:ff:ff:ff:ff:ff link-netns cni-10808ca2-1399-80a4-aeeb-af7af8fe5b0e
    inet6 fe80::501f:edff:fec6:db64/64 scope link
       valid_lft forever preferred_lft forever
4: enic2f0e5537df@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether c2:36:fd:79:84:42 brd ff:ff:ff:ff:ff:ff link-netns cni-a603b98f-1cbb-5d7b-3598-6d0534386140
    inet6 fe80::c036:fdff:fe79:8442/64 scope link
       valid_lft forever preferred_lft forever
5: eni4c16b78ecbc@if3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 02:07:e8:b6:3d:45 brd ff:ff:ff:ff:ff:ff link-netns cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04
    inet6 fe80::7:e8ff:feb6:3d45/64 scope link
       valid_lft forever preferred_lft forever
6: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc mq state UP group default qlen 1000
    link/ether 06:91:75:77:fc:3e brd ff:ff:ff:ff:ff:ff
    inet 172.31.76.93/20 brd 172.31.79.255 scope global eth1
       valid_lft forever preferred_lft forever
    inet6 fe80::491:75ff:fe77:fc3e/64 scope link
       valid_lft forever preferred_lft forever
```

Let's explain the meaning of some fields in the above output first.

- `1:`, `2:`, ... : index of the network interface. Yes, network interface is
  referenced by their index not their name!
- `lo`, `eth0`, `enixxx`: the name of the interface.
- `mtu 9001`: maximum transmission unit = 9001 bytes. A wired number???
- `link/ether 06:91:75:77:fc:3e`: MAC address of this interface.
- `inet 172.31.76.93/20`: ipv4 address.
- `link-netns cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04`: network namespace
- `if` stands for network interface. `@if3` means network interface index 3.

Other fields can be looked up online and are not important for the discussion
follows.

You see that it has two private ip addresses: 172.31.74.55 and 172.31.76.93
which correspond to interface 2 and 6. Interface 1 is the loopback interface.
Interface 2, 3, and 4 are `veth` interfaces. Each `veth` has two ends. One end
is shown above. The other end is inside the network namespace `cni-\*`. Let's
take an example.

```
[ec2-user@ip-172-31-74-55 ~]$ sudo ip netns exec cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04 ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
3: eth0@if5: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 9001 qdisc noqueue state UP group default
    link/ether 16:24:66:46:26:75 brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 172.31.74.233/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::1424:66ff:fe46:2675/64 scope link
       valid_lft forever preferred_lft forever
```

You see that interface 5 in the default namespace and interface 3 in the
`cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04` namespace are the two ends of a
`veth` pair. That is what `@if3` and `@if5` mean in the output. Also, notice
this interface has ip address `172.31.74.233`. This is the same internal IP of
the corresponding pod. See blow `datadog` pod.

```
$ k get po -A -o wide | grep ip-172-31-74-55
kube-system      aws-node-9dn9j                1/1     Running  0    4h43m   172.31.74.55    ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
kube-system      ebs-csi-node-8d88m            3/3     Running  0    4h43m   172.31.72.89    ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
kube-system      efs-csi-node-52j82            3/3     Running  0    4h43m   172.31.74.55    ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
kube-system      kube-proxy-sn9bb              1/1     Running  0    4h43m   172.31.74.55    ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
monitoring       datadog-xkxjd                 3/3     Running  0    4h42m   172.31.74.233   ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
monitoring       filebeat-ops-filebeat-55kx8   1/1     Running  0    4h42m   172.31.77.62    ip-172-31-74-55.us-east-2.compute.internal    <none>    <none>
```

We can do some fun things using `ip` command. First, let's see what processes
are running inside these network namespaces:

```
[ec2-user@ip-172-31-74-55 ~]$ f() { sudo ip netns pids $1 | xargs -I {} ps -p {} --no-headers ; }
[ec2-user@ip-172-31-74-55 ~]$ f cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04
 4915 ?        00:00:00 pause
 5565 ?        00:08:16 agent
 5648 ?        00:00:28 trace-agent
 5714 ?        00:01:59 process-agent
[ec2-user@ip-172-31-74-55 ~]$ f cni-a603b98f-1cbb-5d7b-3598-6d0534386140
 4806 ?        00:00:00 pause
 5124 ?        00:00:20 filebeat
[ec2-user@ip-172-31-74-55 ~]$ f cni-10808ca2-1399-80a4-aeeb-af7af8fe5b0e
 4419 ?        00:00:00 pause
 4514 ?        00:00:01 aws-ebs-csi-dri
 4591 ?        00:00:00 csi-node-driver
 4681 ?        00:00:02 livenessprobe
```

Ah! `datadog-agent`, `filebeat` and `aws related drivers` are running inside
these namespaces.

Second, let's run a http server in one namespace.

```
[ec2-user@ip-172-31-74-55 ~]$ sudo ip netns exec cni-c94a5e10-0a59-4cb8-4ef5-534dfcce3c04 python -m SimpleHTTPServer
Serving HTTP on 0.0.0.0 port 8000 ...
```

This is equivalent to `k exec pod -- python -m SimpleHTTPServer`. Then we curl
it

```
[ec2-user@ip-172-31-74-55 ~]$ curl 172.31.74.233:8000
<!DOCTYPE html PUBLIC "-//W3C//DTD HTML 3.2 Final//EN"><html>
<title>Directory listing for /</title>
<body>
<h2>Directory listing for /</h2>
<hr>
<ul>
<li><a href=".bash_history">.bash_history</a>
<li><a href=".bash_logout">.bash_logout</a>
<li><a href=".bash_profile">.bash_profile</a>
<li><a href=".bashrc">.bashrc</a>
<li><a href=".cache/">.cache/</a>
<li><a href=".lesshst">.lesshst</a>
<li><a href=".ssh/">.ssh/</a>
</ul>
<hr>
</body>
</html>
```

Bingo! The IP assigned to this namespace works. It must work!!. But then the
question is how this host know which namespace or network interface the above
request should be sent to. Why does the node know it should not send the
request to the other two `veth` interfaces? The answer also lies on CNI plugin.
It not only assigns an IP address to a pod, but it also sets up the routing
rule. See below output. Read more detail
[here](https://github.com/aws/amazon-vpc-cni-k8s/blob/d2f240df492fb567508b0154e8bd167b6346445a/docs/cni-proposal.md#cni-plugin-sequence).

```
[ec2-user@ip-172-31-74-55 ~]$ ip route show
default via 172.31.64.1 dev eth0
169.254.169.254 dev eth0
172.31.64.0/20 dev eth0 proto kernel scope link src 172.31.74.55
172.31.72.89 dev eni38fb9ccdc92 scope link
172.31.74.233 dev eni4c16b78ecbc scope link
172.31.77.62 dev enic2f0e5537df scope link
```

```
[ec2-user@ip-172-31-74-55 ~]$ ip rule list
0:      from all lookup local
512:    from all to 172.31.72.89 lookup main
512:    from all to 172.31.77.62 lookup main
512:    from all to 172.31.74.233 lookup main
1024:   from all fwmark 0x80/0x80 lookup main
32766:  from all lookup main
32767:  from all lookup default
```

At this point, I hope you have a good understanding about how CNI works inside
AWS EKS :).

## CoreDNS

In `kube-proxy` section, we discussed how kube-proxy manages iptables and we
see domain name like `ip-172-31-80-113.us-east-2.compute.internal` shows up in
the iptables rules. For packets to transmit, there is step of translating this
name to an IP address. Who does this work for us then? The answer is CoreDNS.

See below `resolv.conf` file I grabbed from a random pod. You see the name
server points to `10.100.0.10` which is the IP address of the `kube-dns`
service.

```
root@server-684c765d78-6z4wv:/app# cat /etc/resolv.conf
nameserver 10.100.0.10
search qa.svc.cluster.local svc.cluster.local cluster.local us-east-2.compute.internal
options ndots:5
```

```
$ k get svc kube-dns -n kube-system
NAME       TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)         AGE
kube-dns   ClusterIP   10.100.0.10   <none>        53/UDP,53/TCP   2y73d
```

Note that the service name is `kube-dns` not `coredns`. This is because before
CoreDNS was invented, kubernetes uses `kube-dns`. CoreDNS is a replacement of
`kube-dns`, and it decided to reuse the same service name `kube-dns` to make it
easy to upgrade kubernetes clusters. Check out
[this page](https://kubernetes.io/docs/tasks/administer-cluster/dns-custom-nameservers/)
for more details.

The `dig` command below also shows `SERVER: 10.100.0.10#53`.

```
root@server-684c765d78-6z4wv:/app# dig redis.production +search

; <<>> DiG 9.16.37-Debian <<>> redis.production +search
;; global options: +cmd
;; Got answer:
;; WARNING: .local is reserved for Multicast DNS
;; You are currently testing what happens when an mDNS query is leaked to DNS
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 51089
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1
;; WARNING: recursion requested but not available

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; COOKIE: efb9d7acd5ce18a1 (echoed)
;; QUESTION SECTION:
;redis.production.svc.cluster.local. IN A

;; ANSWER SECTION:
redis.production.svc.cluster.local. 5 IN A      10.100.215.197

;; Query time: 0 msec
;; SERVER: 10.100.0.10#53(10.100.0.10)
;; WHEN: Sun Feb 05 04:38:21 UTC 2023
;; MSG SIZE  rcvd: 125
```

### How does it work

CoreDNS loads its config file from a configmap as below.

```
$ k describe configmap coredns
Name:         coredns
Namespace:    kube-system
Labels:       eks.amazonaws.com/component=coredns
              k8s-app=kube-dns
Annotations:  <none>

Data
====
Corefile:
----
.:53 {
    errors
    health
    kubernetes cluster.local in-addr.arpa ip6.arpa {
      pods insecure
      fallthrough in-addr.arpa ip6.arpa
    }
    prometheus :9153
    forward . /etc/resolv.conf
    cache 30
    loop
    reload
    loadbalance
}


BinaryData
====

Events:  <none>
```

Each row in the curly bracket is a plugin. Here we only care about the
`kubernetes` plugin. It has a few parameters.

- `cluster.local`, `in-addr.arpa` and `ip6.arpa` are zone names, among which,
  `cluster.local` is the primary zone.
- `pods insecure` means when resolving a pod's address, we do not verify the
  existence of this pod. The other option is `verified`.
- `fallthrough in-addr.arpa ip6.arpa`: [TODO: read this part]

Checkout the
[initialization code](https://github.com/coredns/coredns/blob/169da444ff5ca96104db73742636cc4c9ad15157/plugin/kubernetes/setup.go#L87).

OK, we said enough about configuration. Let's see how kubernetes plugin works.
Let's use the above dig example `dig redis.production +search`. The entry point
is
[here](https://github.com/coredns/coredns/blob/169da444ff5ca96104db73742636cc4c9ad15157/plugin/kubernetes/handler.go#L13).
First, it will find out the matching zone for this request, which matches
`cluster.local`. Then after a few function jumps, it comes to
[here](https://github.com/coredns/coredns/blob/169da444ff5ca96104db73742636cc4c9ad15157/plugin/kubernetes/kubernetes.go#L377).
The interesting part is this line

```
	r, e := parseRequest(state.Name(), state.Zone)
```

This function translates a DNS request to a kubernetes lookup request. What I
mean is, using our example, it translate the DNS request for host name
`redis.production.svc.cluster.local.` to a request of finding out the service
`redis` in namespace `production`. Once this service is retrieved from
kubernetes api server, then the internal IP of this service is returned to the
DNS client. Here, `state.Name()` is the DNS query question name, which is
`redis.production.svc.cluster.local.` (copied below).

```
;; QUESTION SECTION:
;redis.production.svc.cluster.local. IN A
```

`state.Zone` is `cluster.local.`. If you read this function carefully, you will
see that it only support 3 different types of DNS host name. Here we show some
examples.

**Query by service name**

```
# dig redis.production.svc.cluster.local +noall +answer
redis.production.svc.cluster.local. 5 IN A      10.100.215.197
```

**Query by endpoints id**

```
# dig 172-31-89-91.redis-master.production.svc.cluster.local +noall +answer
172-31-89-91.redis-master.production.svc.cluster.local. 5 IN A 172.31.89.91
```

Here the ip address is the ip of endpoints `redis-master`.

**Query by pod id**

```
# dig 172-31-89-91.production.pod.cluster.local +noall +answer
172-31-89-91.production.pod.cluster.local. 5 IN A 172.31.89.91
```

Here the ip address is the ip of a pod `redis-master-0`.
