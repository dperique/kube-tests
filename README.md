# A Collection of Kubernetes tests

These are tests we can use to confirm that Kubernetes is working well when we do various things
like upgrading various components, replacing, adding, removing Kubernetes nodes, etc.
Typically, Kubernetes operators will have to do things
like upgrade Kubernetes, network plugins, applications, etc.  We want to write some tests
we can execute so that we know such operations result with behaviors we expect.  For
example, I want to know that a Kubernetes upgrade or a Calico upgrade causes no downtime
or disruption to workloads.

## The full mesh connectivity test

Summary: confirm there is connectivity between Pods running on each Kubernetes node.
Every Pod on every node should be able to reach every Pod on every node.  There are two
types of tests: simple ping and iperf.  Ping is simple and works well but due to the spacing
between the icmp packets, it will miss very brief networking "blips".  Iperf is more complex
but gives a more continuos stream of traffic.

* Create a daemonset using a Pod image that contains ping and iperf.  This will give
  you a Pod on every Kubernetes node.
* Run iperf servers on every Pod image
* Run iperf clients on every Pod image; use maybe 1Kbps of bandwidth
* Confirm you have no packet drops and the bandwidth you specified
* Confirm you have full-mesh: Pod-to-Pod traffic from every Kubernetes node to
  every Kubernetes node.
  * If there are too many, you can run iperf server on all N
    nodes and one iperf client on each node but each iperf client is sending traffic
    to a different iperf server on a different node.  This will at least give you a
    sampling of the full-mesh traffic paths.
  * Repeat with a different set of iperf clients

You can perform the same test above except using ping with a small interval.  For example
`ping -i .01 10.233.67.80`.

Here are some iperf commands:

```
# Start iperf udp server, set report interval of 1 second.
$ iperf -sui 1

# Start iperf udp client sending traffic to 10.233.67.80 with report interval of 1.
$ iperf -c -u -i 1 10.233.67.80
```

## Test to confirm /etc/hosts

After adding a new Kubernetes node using kubespray, the /etc/hosts file should list every node.
This is especially true with the apiserver, controller, and scheduler.  There have been times
when I add a Kubernetes node only to find out that kubelet doesn't start.

After adding a new Kubernetes node, run these commands (I assume 3 Kubernetes masters named
"kube-test-k8s-node-{1..3}").  View
the results and you should see all of your Kubernetes nodes listed in /etc/hosts.

```
for i in 1 2 3 ; do kubectl -n kube-system exec -t kube-apiserver-kube-test-k8s-node-$i cat /etc/hosts ; done
for i in 1 2 3 ; do kubectl -n kube-system exec -t kube-controller-manager-kube-test-k8s-node-$i cat /etc/hosts ; done
for i in 1 2 3 ; do kubectl -n kube-system exec -t kube-scheduler-kube-test-k8s-node-$i cat /etc/hosts ; done
```

## Confirm Kubernetes events look good

See [this article](https://www.bluematador.com/blog/kubernetes-events-explained) for ideas which
I'll expand upon later.

## Test Calico upgrade

Calico upgrades from v2.x to v2.x+1 are quick and deployed in a rolling fashion becase
Calico pods deploy as a daemonset.  We believe that once networking and IPs are set up
by Calico, Calico pods can restart with no loss of traffic and no disruption to Kubernetes
workloads.

The test:

* Setup and run a full-mesh ping or full-mesh iperf test
* Upgrade the calico daemonset
  * the calico pods should restart in a rolling fashion
  * You should see no loss in traffic
* Repeat with more traffic as needed
* Restart several Pods (this will make calico give more IP addresses)
* Setup and run a full-mesh ping or full-mesh iperf test
* Confirm `/etc/hosts` is correct
* Confirm events look ok

Calico upgrades from 2.x to 3.x require a database migration.  We use kubespray to run
this upgrade.  Repeat the same test above except upgrade from Calico 2.x to 3.x.
