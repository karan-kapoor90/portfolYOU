---
title: Kubernetes Primer
tags: [kubernetes, cloud native]
style: 
color: 
description: A primer guide with commands and syntax for kubernetes
---

## Kubernetes Master Plane Components


1. etcd - key value store that holds all the data about the cluster, the pods, containers, versions, deployment times etc. Default port 2379
2. kube-scheduler - the component that manages deployment of containers on a node, identifies the right node as per the policies etc.
3. Controllers - monitors the state of resources and works to bring them to the desired state
   - Node Controller - Node status - monitors nodes every 5 seconds; waits for 40 seconds before markingit unreachable. 5 mins wait after that for machine to come back. Then it'll remove pods from node
   - Replication Controller 


## Resources


### Pods

Pod definition default yaml vars:

```yaml
apiVersion: v1
type: Pod
    metadata:
        name: <pod_name>
        label:
            app: something
    spec:
        containers:
            - name: <somename_for_container>
              image: karankapoor/container_name
```

> One of the major differences b/w replication controller and the replicaSet is the selector label. Selector labels help the replicaset control pods that weren't created using the template inside the RS definition. The Selector label is an optional component in the RC. 
rc are available in apiVerision v1, while rs is available in apps/v1.

Som 

#### Useful Commands

- Getting the details of a pod as a yaml
  `$ kubectl get po <pod_name> -o yaml`
- Getting more details about a deployed pod
  `$ kubectl get po <pod_name> -o wide`
- Watch the get pods command for changes to pods
  `$ kubectl get po <pod_name> -w`
- Imperatively create a pod directly from the command line without using a yaml file
  ```bash
  $ kubectl run --generator=run-pod/v1 <pod_name> --image=<image_name> --labels="<key>=<value>" --dry-run -o yaml > newfile.yaml
  # --dry-run : Shows the outcome of the command but doesn't execute it at all
  # --labels : adds the labels to the pod that will be created
  # -o yaml : Outputs the resource definition that is going to be created in a yaml format
  # > newfile.yaml : writes the output (the yaml contents in this case) to newfile.yaml
  ```
- Get pods by specific labels
  `$ kubectl get po --selector <key>=<value>  # instead of usign --selector, you can also use -l`

- Edit a pod
  `$ kubectl edit pod <podname>     # you cannot however edit some of the properites for a pod at runtime such as the resource utilization, env vars, service accounts etc.`
  * export the pod definition to a yaml file, make changes and then delete the pod and recreate using the modified yaml file. A better option is to put the pod into a deployment and editing the deployment file since the pod spec is a child of the deployment. 

### ReplicaSet


```yaml
apiVersion: apps/v1
type: ReplicaSet
    metadata:
        name: <Replicaset_name>
        label:
            app: something
    spec:
        replicas: <number_of_base_replicas>
        selector: 
            matchLabels:
                app: something
        template:
            # Copy the pod definition here
            metadata:
                name: <pod_name>
                label:
                    app: something
            spec:
                containers:
                    - name: <somename_for_container>
                    image: karankapoor/container_name
```


the labels under the matchlabels section must match the pods labels and not the rs labels because these are used by the rs to determine which pods to control.


### Deployment



### Namespaces

Namespace service Address: <namespace>.svc.cluster.local
kube-system: namespace for k8s services
kube-public: services to be accessed by all namespaces
default: a default namespace


Switch your current context to a different namespace
$ kubectl config set-context $(kubectl config current-context) --namespace=<namespace_name>


### Services

Nodeport: makes container port available on the node's physical port
ClusterIp: Service creates a virtual IP inside the cluster to enable inter container communication.
LoadBalancer: use a cloud provider's nodeport




NodePort: 
TargetPort: the port on which the pod is exposing what's running inside it.
Port: this is the service port. think of the service as a virtual server. Inside the cluster it has its own cluster address called the service's cluster Ip
NodePort: A valid port in the 30000-32767 port range on the host machine.


Uses a "Random" Algorithm. It makes the same port on all nodes available as the outbound port for the service.

```yaml
apiVersion: v1
kind: Service
metadata:
    name:
    label:
spec:
    type: NodePort
    ports:
    - targetPort: 80  # taken as the same value as the port if not specified
      port: 80  # mandatory
      nodePort: 30008  # will be selected from the valid range if value is not provided
    selector: # selector labels from the pod definition
      app: something
      type: something
```



ClusterIp: the right method for establishing internal communication between pods. This basically gives a single interface/ internal IP address for other pods to access the service. Uses " Random" algorithm.



```yaml
apiVersion: v1
kind: Service
metadata:
    name:
    label:
spec:
    type: ClusterIP  # every service even when the type is not defined, defaults to clusterIP
    ports:
    - targetPort: 80 # taken as the same value as the port if not specified
      port: 80  # mandatory
      nodePort: 30008  # will be selected from the valid range if value is not provided
    selector: # selector labels from the pod definition
      app: something
      type: something
```

Important: to create a dummy template yaml file than can be used to fill up with more specific details before actually using out the file, use the following:


kubectl create deployment --image=nignx nginx --dry-run -o yaml > myDescriptor.yaml


To create a pod, one has to use the generator to create it directly from the CLI. Use the following command


kubectl run --generator=run-pod/v1 nginx --image=nginx --labels="app=test" --dry-run -o yaml


Expose pod as a service


kubectl expose pod <podname> --port=<portnumber> --name=<service-name>  //automatically uses the pod's selectors for the service.


Create a deployment 


$ kubectl create deployment --image=<imagename> <deployment-name>    // Note that this command doesn't allows you to define the number of replicas. You will either need to create the file or use the scale command
$ kubectl scale deployment/<deployment_name> --replicas=4    //note that the scale command doesn't update the underlying yaml with the latest number of replicas in the deployment.






### Kubernetes Scheduler

Scheduler decides which pod should go on which node. It does so by setting a property called nodeName in the pod definition file automatically if it hasn't already been set. Without the scheduler, the pod stays in a pending state and is not deployed to any particular node.

You can manually set the `nodeName` field in the Pod `spec`. 

> You can only specify the node name property at creation time. It cannot be modified later. 

To be able to change the nodeName spec for a pod, you should configure a binding object that binds the pod to a node. This is essentially the same way how the scheduler does it.

So if you wanted to specify the nodeName in the Pod spec itself, it would look like this:

```yaml
apiVersion: v1
kind: Pod
metadata:
    name: <pod-name>
    label:
        app: frontend
spec:
    containers:
    - name: nginx
      image: nginx
      ports:
        - containerPort: 8080
    nodeName: node02
```

Converseley if you'd want to create a binding to do the pod to node allocation, a *representative* Binding yaml would look like:

```yaml
apiVersion: v1
kind: Binding
metadata:
    name: <my_binding_name>
target:
    apiVersion: v1
    kind: Node
    name: node02
```

> The above is *only for representation purposes*, in order to actually do this, you have to send a REST API call with the input as json formatted string.


### Labels and Selectors

Labels are the key value labels that you put in your cluster. Selectors are what are used to select specific components basis their types and labels.
> Annotations are another section that can be given in a resource's metadata definition. This can be used to specify things like the author, version or something else that  you may want to specify.

### Taint and Tolerations

Taint is marking a node such that the scheduler doesn't schedule the pod on the node. There could however be pods that can be tolerant to the taint and can still be placed on a particular node.

Use Case: Imagine you've got 2 services, A and B that need to be deployed on the cluster. Node 1 is a dedicated resource in your 3 node cluster for Service A. In order to achieve this isolation, here's what you'd do:
- Taint node 1 so no pods can be placed on that node.
- To enable pod A to be able to be deployed to this node, add a toleration to Service A pod that can tolerate the taint. 

> Taints are set on nodes, tolerations are set on pods.

> Taints and tolerations do not guarantee that a pod with a certain toleration will not be placed on any other un-tainted node. Meaning that a node tainted red will only accept pods which have a toleration defined for red, but it doesn't imply that the pod with a toleration for red will not be placed on a node that doesn't have any taint.

In order to do this, create a taint on a node:

```bash
$ kubectl taint node <node_name> <key>=<value>:<taint_effect>

# <key>=<value> is how you create a taint. For any pod to be deployed on this node, it must have this <key>=<value> in its tolerations.

```

3 types of taint_effects
1. NoSchedule - no new pods will be scheduled on the node; existing pods will remain.
2. PreferNoSchedule - it will attempt to not schedule any pods on the node, but not guaranteed. Existing pods will remain
3. NoExecute - No new pods will be scheduled; existing pods if they don't have a toleration towards the taint, will be gracefull evicted.

Example command:
`$ kubectl taint node node1 app=blue :NoSchedule`

To add tolerations to a pod, add tolerations to the spec section in the pods definition as follows:

> This is how one would create a toleration for the node1 as per the example command above. All values MUST BE in double quotes.

```yaml
apiVersion: v1
kind: Pod
    metadata:
        name: <pod_name>
        label:
            app: something
    spec:
        containers:
            - name: <container_name>
              image: <image_name>
        tolerations:
            - key: "app"
              operator: "Equal"
              value: "blue"
              effect: "NoSchedule"
```
NoExecute: As soon as the taint with NoExecute is defined on a node, it will automatically kill the pod that doesn't have the toleration for that specific taint.

> Point to remember: Taints and tolerations do not limit the pods from going to a specific node, instead limit the kind of pods that a node can accept. This means that if Node1 has a taint and only Service A has a toleration for that taint, the taints and tolerations stop pods other than Pod A from being scheduled on Node1 but does not stop Pod A from being scheduled on any other node. This can be done using Node Affinity.

> By default in a k8s cluster, there is a taint set on the master node that prevents pods from being scheduled on the master node. See it using the command `kubectl describe node master | grep Taints`

To remove a taint from a node, get the value of taint from the node description and run the following command:
`$ kubectl taint node <node-name> <taint-value-from-node-description>-    # Note the hyphen at the end of the command`


### Node Selectors

You may have different hardware boxes as a part of the same cluster that you may want to dedicate to different pods etc. The simple way to do it is to define the `nodeSelector` in the pod description with the key value pair that you also mark on the specific node where you want these pods to go. 

Labelling a node:

```bash
$ kubectl label nodes <node-name> <key>=<value>
```

In the pod definition yaml, you can do the following: 

```yaml
apiVersion: v1
kind: Pod
    metadata:
        name: <pod_name>
        label:
            app: something
    spec:
        containers:
            - name: <container_name>
              image: <image_name>
        nodeSelector:
            <key>: <value>           
```

> This can help you put specific pods on specific nodes, but what if you want a slightly different scenario where you want a pod to be deployed to a node with the labels size=small or size=medium? or a limitation like any node except size=small? - This can be done using Node affinity.


### Node Affinity

You cand provide advanced/ complex selector mechanisms for scheduling pods on nodes using node affinity. This is where you can use operators as well to for example specify a set of nodes where the values for the key may be x, y or z...

> Node affinity and anti-affinity can also be used to create hard or soft affinity on the basis of pods such as a hard anti-affinity for pods A and pods B being placed on the same node together.

- Hard Affinity: requiredDuringSchedulingIgnoreDuringExecution - these rules MUST be satisfied for a pod to be scheduled to a node.
- Soft Affinity: preferredDuringSchedulingIgnoreDuringExecution - the scheduler must try and meet these rules but continue to schedule the pod even if they're not met.

**Roadmap Affinity Types**

- RequiredDuringSchedulingRequiredDuringExecution - this will evict the pods from any of the nodes where the conditions aren't met.

> IgnoreDuringExecution means that if the labels on a node change at runtime, the pods already running on that node that do not have their rules met, will continue to keep running and not shut down.

Supported Operators:
- In
- NotIn
- Exists
- DoesnotExist
- Gt
- Lt

> One can use NotIn and DoesNotExist along with Taints to achieve node anti-affinity.

```yaml
apiVersion: v1
kind: Pod
    metadata:
        name: <pod_name>
        label:
            app: something
    spec:
        containers:
            - name: <container_name>
              image: <image_name>
        affinity:
            nodeAffinity:
                requiredDuringSchedulingIgnoreDuringExecution:
                    nodeSelectorTerms: # This will check if the node matches ANY of the matchExpressions. It does not need all the match expressions below it to be met.
                    - matchExpressions:  # This will ensure that the pod is scheduled on the node that matches ALL the key value pair conditions in the matchExpressions
                        - key: <somekey>
                          operator: In # In case you use Exists, you do not neceseily need to give a value because it'll check if the key itself exists.
                          value: 
                          - somevalue1
                          - soevalue2           
```
> NOTE: yaml doesn't accept tabs and typing out this yaml can be quite difficult typographically. Try and type this on a yaml editor if required.


```yaml
# Representation for setting node affinity by required during scheduling rule
affinity:
    nodeAffinity:
        requiredDuringSchedulingIgnoreDuringExecution:
            nodeSelectorTerms: # Non Array type
            - matchExpressions:     # Array of match expressions, ALL match expressions in the array must be satisfied
                - key: <somekey>
                  operator: <some_operator>
                  values:
                  - <value1>
                  - <value2>
```

Weight is needed when setting up a preferredDuringScheduling rule because the scheduler will try and schedule the pod with the highest score:

```yaml
# Representation for setting the node affinity by preferrence rule
affinity:
    nodeAffinity:  
        preferredDuringSchedulingIgnoreDuringExecution:
        - weight: 100     # Array starts from here, not from matchExpressions. Weight is calculated for every individual node and the highest scoring node gets the pod.
          preference:
            matchExpressions:
            - key: <somekey>
              operator: <some_operator>
              values:
              - <value1>
              - <value2>
```

> Node anti-affinity doesn't have a seperate clause for itself. You control it using the operators in the 

> Node affinity is defined on the pod spec, meaning that that pod will be placed on a node with a matching label, it doesn't however restrict other pods from being scheduled on that node. 

### Pod Affinity

We may want to put for example the redis pod on the same node where the frontend pods are. Hence, similar to node affinity, one can define pod affinity and anti affinity to place pods in proximity/ away from each other. 

```yaml
affinity:
    podAffinity:
        requiredDuringSchedulingIgnoreDuringExecution:
        - labelSelector:
            matchExpressions:
            - key: <some-key>
              operator: <some-operator>
              value: <some-value>
          topologyKey: failure-domain.beta.kubernetes.io/zone
```

```yaml
affinity:
    podAntiAffinity:
        preferredDuringSchedulingIgnoreDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
                matchExpressions:
                - key: <some-key>
                  operator: <some-operator>
                  value: <some-value>
            topologyKey: zone/ hostname/ region
```

> GOTCHA: Let's assume a scenario where pods S1 and S2 need to be placed together, hence you're looking to setup pod affinity for the same. In such a scenario where the podAffinity is set to a particular label for say a pod S1 and the affinity rule for S1 is written in pod S2's podSpec, the first time you try to create a pod for S2 and if there are no running S1 pods, S2 will also not get scheduled if the affinity rule is set to requiredDuringScheduling. For this corner case, see https://github.com/kubernetes/community/blob/master/contributors/design-proposals/scheduling/podaffinity.md#affinity 

`topologyKey`: these are pre-populated labels allocated to nodes and persistentVolumes (more on that later) and you can use these labels to define nodeAffinity and pod affinity. For more information and a detailed description of these labels, see : https://kubernetes.io/docs/reference/kubernetes-api/labels-annotations-taints/#topologykubernetesioregion 


### Dedicating nodes for specific pods

To dedicate nodes for specific pods, we must:
Step 1. - use taints and toleration to paint the individual nodes
Step 2. - use node affinity to get specific pods to specific nodes



### Resource requirements and Limits

> If during scheduling, the scheduler finds that no capacity is available on any of the nodes, it delays the scheduling of pod on any of the nodes and the pod shows in pending state.

> By defualt k8s assumes that every pod requires 0.5 CPU's and 256 MB RAM. - this is the minimum requested by every pod. 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: <somename>
spec:
  containers:
  - name: <container_name>
    image: <image_name>
    ports:
    - containerPort: 8080
    resources:      # resource configurations are defined per container.
        requests:       # container in the pod is initiated with this set of resources. Basically the minimum
          memory: "1Gi"     # Ki (Kibibyte) = 1024 bytes while K is 1000 bytes
          cpu: 1
        limits:     # container in the pod can go upto this maximum limit on the node.
          memory: "2Gi"
          cpu: 2 

```

Docker containers can typically take up all the resources on a node. K8s however sets a 1 vcpu limit. 
If a pod is throttled to stay in the cpu limit however k8s allows the pod to go over the memory limit. If the pod stays beyond the memory limit for longer durations, it is terminated.


### Daemon Set

Daemon sets allow you to run a particular pod on every node of the cluster. Services such as kube proxy, some per node networking agent, a monitoring and logging agen tetc are the perfect examples of Daemon sets. The creation and deletion of a doaemon set is linked with the creation and deletion of the node. It will be created when a new node is attached and deleted from that node when the node is removed.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: <some-name>
spec:
  selector:
    matchLabels:
        app: monitoring-agent
  template:
    metadata:
        name: <somename>
        labels:
          app: monitoring-agent
    spec:
      containers:
      - name: <container-name>
        image: <monitoring-agent-image-name>
```

Daemon sets internally use the nodeAffinity rules to ensure one Daemon set per node. 

### Static Pods

If one looks at the individual worker nodes, every worker has a kubelet process running on it that knows how to run pods on the machine. In order for it to run a pod however, it must receive the yaml file from the api-server running on a master node. We can configure a path on the worker machine from where the kubelet can pick up yaml files and create pods wihtout needing the api-server. The kubelet periodically checks the folder for changes and runs any new resources. If the pod fails, the kubelet can recreate the pod on the node. If you make changes, the kubelet applies the changes. If you delete a file from the directory the pod will be removed as well. Such pods are called static pods.

> kubelet can only run a pod this way, replica sets, deployments, services etc can not be run this way. 

1. Configuring using a config file:
The kubelet service has a path for the configuration file that it is using. `--config` in the kubelet.service. The config yaml file that this flag points to should have an entry called the `staticPodPath` for the folder from where the kubelet should pick up files for creating static pods. Cluster configured using the kubeadm tool use this method. 
To check static pod path:
```bash
$ ps -aux| grep kubelet
# read the location of the --config flag
$ cat <file_location>/<filename> | grep staticPodPath
```


2. Configure using the kubelet service:
The kubelet.service file can have a variable called the `--pod-manifest-path` which defines the folder path for picking up yaml files for static pods

> In the worker node, we don't have the kube-api server, the pods run using the kubelet will still be seen using the `docker ps` command. Kubectl commands will not be available here for the management.


> The api server on the master talks to the api exposed by the kubelet. Hence one can create static pods using the API for the kubelet as well. 

Static pods create using the kubelet directly show up in `kubectl get pods` as well but are in read only mode and appended with the name of the node. Management will have to be done from the individual files on the node itself.


> Use case: this can be used to deploy pods on a node such that they're not dependent on the kubernetes control plane. We can use it to deploy the various control plane components as well. That's how kubeadm does the setup. You install kubelet, create pod defiition file for docker and then put into the static pods folder. 

Static Pods(SP) vs Daemon Sets(DS):

SP: Created by Kubelet
DS: Created by Kube API server using the DaemonSet Controller

SP: Used to deploy contorl plane components as static pods.
DS: Deploy Monitoring, Alerting etc. per node agents

Both are ignored by kube-scheduler. the kube scheduler has no effect on these resources.


### Kube schedulers

We can use a custom scheduler that can be used to schedule pods as per our own custom logic. We can give schedulerName in the podSpec to specify which scheduler we want to run for the specific pod. 

If the scheduler is misconfigured, the pod will remain in a pending stage and will not be scheduled onto a node. To find the scheduler, on the individual kubernetes worker node, look at the kubelet service details by looking at the kubelet service startup command, and look for the --config flag for the location of the kubelet configuration file. Here look for the staticPodPath. The kubernetes scheduler yaml file should be here. To create your own scheduler, make a copy of this file, and make changes to it.


1. Get the kubernetes config file path (if you're logged into the master. If not, you might need to ssh into the master node)
```bash 
$ ps -aux | grep kubelet | grep config
# Example Response
root      3602  2.7  2.2 764928 91184 ?        Ssl  08:31   0:07 /usr/bin/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --kubeconfig=/etc/kubernetes/kubelet.conf--config=/var/lib/kubelet/config.yaml --cgroup-driver=systemd --network-plugin=cni --pod-infra-container-image=k8s.gcr.io/pause:3.1
```

3. Open the file at the --config path and get the staticPodPath 

```bash
$ cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Example response
staticPodPath: /etc/kubernetes/manifests
```

4. Copy the kubernetes scheduler yaml from the staticPodPath location

```bash
$ cp /etc/kubernetes/manifests/kube-scheduler.yaml my-scheduler.yaml
```

5. Edit the my-scheduler.yaml spec

```yaml
spec:
  containers:
  - containers:
    - kube-scheduler
    - --scheduler-name=my-scheduler  # Additional Parameter
    - --port=10282  # Additional Parameter
    livenessProbe:
      failureThreshold: 8
      httpGet:
        port: 10282  # Updated port number
```

6. Deploy the new scheduler and check running status

```bash
$ kubectl create -f my-scheduler.yaml
$ kubectl get po -n kube-system | grep my-scheduler
```

#### Using the scheduler in a pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  schedulerName: my-scheduler
  containers:
  - name: nginx
    image: nginx
```