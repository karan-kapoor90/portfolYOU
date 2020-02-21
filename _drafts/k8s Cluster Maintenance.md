# Worker Node OS upgrades

If you create and deploy pods, and the node on which they're running goes down, k8s waits for a default timeout of 5 mins for the nodes to come back up. If it doesn't come back up, the pods are terminated and k8s considers them dead. If the pods are a part of a replica set thne they're recreated on other nodes. The time that k8s waits for a pod to come back online is called `pod eviction timeout` and it's set on the `kubernetes controller manager`. This is 5 mind by default. If the node comes back up after 5 mins, it comes back online as empty and the pods aren't recreated. 

Hence do a quick update and reboot if you cannot be sure if the node is going to be back online in 5 mins.

Way to do it is by draining the nodes `kubectl drain <node-name>`. When you drain a node the pods are gracefully terminated and recreated on another node and the node is marked as un-schedulable. When it comes back online, it is still un-schedulable. You have to un-cordone it `kubectl uncordon <node-name>`. The cordon command simply marks the node unschedulable but does not move the exiting pods to another node, only marks it so that no new pods are scheduled on it `kubectl cordon <node-name>`.

> Drain may need --force or --ignore-daemonsets if the node has any pods running on it that aree not controlled by a replicaset or/ and if there are daemonset pods running on the node respectively.

# Kubernetes version upgrade

ETCD and CoreDNS components are componets that are external dependencies used outside of the core components of k8s. The core components are:
1. kube-apiserver --> version x
2. controller-manager --> version x-1
3. kube-scheduler --> version x-1
4. kubelet --> version x-2
5. kube-proxy --> version x-2
6. kubectl --> x+1>x-1

An upgrade can be upgraded component by component. At any time k8s supports only 3 minor versions. Meaning that ver 1.13 supports 1.13, 1.12 and 1.11. The recommended upgrade path is from one minor version to the next and not a big leap to a newer version. Upgrade steps are as follows:

1. Upgrade kubeadm.

```bash
$ apt-get upgrade -y kubeadm=1.13.0
```

2. Upgrade master node. This doesn't have an application downtime but neither the kube-apiserver (hence kubectl) nor the controller managers are active. Since the controller managers aren't active it means that if a replica set goes down, they will not be recreated. 

```bash
$ kubeadm upgrade apply v1.13.0

# if the master node also runs containers, and hence a kubelet process, upgrade the kubelet on the master as follows:

$ apt-get upgrade -y kubelet=1.13.0
$ systemctl restart kubelet
```

3. Update worker nodes. This is basically upgrading the kubelets. Worker update strategies:

    - Strategy 1 - all at once
    - Strategy 2 - Cyclically upgrade nodes
    - Strategy 3 - Add new nodes

```bash
# drain the worker node
$ kubectl drain <node-name>

# upgrade kubeadm and kubelet
$ apt-get upgrade -y kubeadm=1.13.0
$ apt-get upgrade -y kubelet=1.13.0

# run kubeadm to upgrade the node configuration to a new kubelet version
$ kubeadm upgrade node config --kubelet-version 1.13.0

# restart the kubelet service
$ systemctl restart kubelet

# make the node schedulable again. New pods or pods being recreated will come on this node
$ kubectl uncordon <node-name>
```

