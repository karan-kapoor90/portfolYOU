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

## Backup and restore

### Resource Configuration

#### Resource Configurations

You must back up resources created using both declarative and imperative methods.

```bash
$ kubectl get all --all-namespaces -o yaml > all-resources.yaml
```

> Vellero can help taking a backup as well.

#### Etcd resource

Etcd has details of all the configurations on the server. Etcd is deployed on the master node. The data directory of the Etcd service has the path for all the congurations in the server and is hence what should be backed up.

``` etcd.service --data-dir

Etcd also has a built in utility for taking a snapshot of the etcd database. 

**Backup**

```bash
$ ETCDCTL_API=3 etcdctl snapshot save snapshot.db
```

**Restore**

1. Stop the apiserver

```bash
$ service kube-apiserver stop
```

2. Restore using the etcd server restore utility

```bash
$ ETCDCTL_API=3 etcd snapshot restore snapshot.db \
--data-dir /var/lib/etcd-from-backup \ # path to the new data directory to be created. This is where the new data will be for this restored database
--initial-cluster master1:https://<master1-ip-address>:2380,master2=https://<master1-ip-address>:2380 \ # master node communication addresses
--initial-cluster-token <target-cluster-token> 
```

> When etcd triggers a restore it initialises a new cluster configuration and configures the members of etcd as new members to a new cluster. This is to prevent a new member from accidentally joining an existing cluster. For example when you're setting up a new test cluster with the production cluster configration, this will ensure that the new members don't accidentally join the production cluster.

3. Pointing the running etcd service to the new restored configuration

Update the etcd.service --data-dir and the --initial-cluster-token to point to the new configuration. Then restart the etcd service as follows:

```bash
$ systemctl daemon-reload
$ service etcd restart

```

4. Restart the API service

```bash
$ service kube-api-service restart
```

> With all etcdctl commands, you must specify the --endpoint=<etcd-service-address> --cacert=/etc/etcd/ca.crt --cert=/etc/etcd/etcd-server.crt --key=/etc/etcd/etcd-server.key