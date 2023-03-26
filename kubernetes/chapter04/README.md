- [Pod](#pod)
- [Pods and controllers](#pods-and-controllers)
- [Pod Phase](#pod-phase)
- [Pod conditions](#pod-conditions)
- [Container states](#container-states)
  - [Waiting](#waiting)
  - [Running](#running)
  - [Terminated](#terminated)
- [Container probes](#container-probes)
  - [Readiness Probes](#readiness-probes)
  - [Liveness Probes](#liveness-probes)
  - [Startup Probes](#startup-probes)
- [Init Containers](#init-containers)
- [Ephemeral Containers](#ephemeral-containers)
- [Pod Quality of Service Classes（QoS）](#pod-quality-of-service-classesqos)
  - [Guaranteed](#guaranteed)
  - [Burstable](#burstable)
  - [BestEffort](#besteffort)

## Pod 

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20230319171527.png)

Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20230319171618.png)

The shared context of a Pod is a set of **Linux namespaces, cgroups**, and potentially other facets of isolation - the same things that isolate a container. A Pod is similar to a set of containers with shared namespaces and shared filesystem volumes.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```


## Pods and controllers

You can use workload resources to create and manage multiple Pods for you. A controller for the resource handles replication and rollout and automatic healing in case of Pod failure. 

Here are some examples of workload resources that manage one or more Pods:
- [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)


## Pod Phase

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20230319112619.png)

Note: When a Pod is being deleted, it is shown as **Terminating** by some kubectl commands. This Terminating status is not one of the Pod phases. A Pod is granted a term to terminate gracefully, which defaults to 30 seconds. You can use the flag --force to terminate a Pod by force.


## Pod conditions

A Pod has a PodStatus, which has an array of PodConditions through which the Pod has or has not passed. Kubelet manages the following PodConditions:

- PodScheduled: the Pod has been scheduled to a node.
- PodHasNetwork: (alpha feature; must be enabled explicitly) the Pod sandbox has been successfully created and networking configured.
- ContainersReady: all containers in the Pod are ready.
- Initialized: all init containers have completed successfully.
- Ready: the Pod is able to serve requests and should be added to the load balancing pools of all matching Services.

```bash
# create a pod
kubectl run nginx --image=nginx

# display pod information
kubectl get pod nginx -o yaml

# output
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: nginx
  name: nginx
  namespace: default
  resourceVersion: "536324"
  uid: 2fa0f043-601e-4f65-b25e-b4b7d2efad2b
spec:
  containers:
    image: nginx
    imagePullPolicy: Always
    name: nginx
......
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2023-03-19T03:03:05Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2023-03-19T03:03:10Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2023-03-19T03:03:10Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2023-03-19T03:03:05Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: containerd://ff2e7936e67eb10cebfa3426c50318b9215157cfe090422d26c99a34da856c75
    image: docker.io/library/nginx:latest
    imageID: docker.io/library/nginx@sha256:aa0afebbb3cfa473099a62c4b32e9b3fb73ed23f2a75a65ce1d4b4f55a5c2ef2
    lastState: {}
    name: nginx
    ready: true
    restartCount: 0
    started: true
    state:
      running:
        startedAt: "2023-03-19T03:03:10Z"
  hostIP: 10.180.25.81
  phase: Running
  podIP: 100.64.0.27
  podIPs:
  - ip: 100.64.0.27
  qosClass: BestEffort
  startTime: "2023-03-19T03:03:05Z"
```

## Container states

### Waiting
If a container is not in either the Running or Terminated state, it is Waiting. A container in the Waiting state is still running the operations it requires in order to complete start up: for example, **pulling the container image from a container image registry, or applying Secret data.** When you use kubectl to query a Pod with a container that is Waiting, you also see a Reason field to summarize why the container is in that state.

### Running
The Running status indicates that a container is executing without issues. If there was a **postStart** hook configured, it has already executed and finished. When you use kubectl to query a Pod with a container that is Running, you also see information about when the container entered the Running state.

### Terminated
A container in the Terminated state began execution and then either ran to completion or failed for some reason. When you use kubectl to query a Pod with a container that is Terminated, you see a reason, an exit code, and the start and finish time for that container's period of execution.If a container has a **preStop** hook configured, this hook runs before the container enters the Terminated state.


[Lab: Attach Handlers to Container Lifecycle Events](https://kubernetes.io/docs/tasks/configure-pod-container/attach-handler-lifecycle-event/)

## Container probes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sample-web-app
  labels:
    app: web
spec:
  containers:
  - name: web-container
    image: example/sample-web-app:latest
    ports:
    - containerPort: 8080
    readinessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 3
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
      timeoutSeconds: 2
      successThreshold: 1
      failureThreshold: 3
    startupProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 5
      timeoutSeconds: 1
      successThreshold: 1
      failureThreshold: 30
```

This example defines a pod with a single container running a web application. The container exposes port 8080 and uses the /health endpoint for all three probes.

- **Readiness Probe**: This probe checks if the container is ready to serve traffic. It starts 5 seconds after the container is created, with a period of 5 seconds between each check. It fails if it takes longer than 1 second to get a response, and the probe is considered failed after 3 consecutive failures.
- **Liveness Probe**: This probe checks if the container is still running. It starts 15 seconds after the container is created, with a period of 10 seconds between each check. It fails if it takes longer than 2 seconds to get a response, and the container will be restarted after 3 consecutive failures.
- **Startup Probe**: This probe checks if the application has started. It starts 10 seconds after the container is created, with a period of 5 seconds between each check. It fails if it takes longer than 1 second to get a response, and the container will be restarted after 30 consecutive failures.


### Readiness Probes

**Readiness probes are used to let kube-proxy know when the application is ready to accept new traffic.** A primary use case for readiness probes is directing traffic to deployments behind a service.

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/0_AvaYbgMkeHJ0Pis8.gif)


### Liveness Probes

**Liveness probes are used to restart unhealthy containers.** The kubelet periodically pings the liveness probe, determines the health, and kills the pod if it fails the liveness check.

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/0_yicsIyLNZJlDlIsf.gif)

### Startup Probes

Startup probes are similar to readiness probes but only executed at startup. They are optimized for slow starting containers or applications with unpredictable initialization processes. 

For newer (≥ 1.16) Kubernetes clusters, use a startup probe for applications with unpredictable or variable startup times. The startup probe may share the same endpoint (e.g. /healthz ) as the readiness and liveness probes, **but set the failureThreshold higher than the other probes to account for longer start times**, but more reasonable time to failure for liveness and readiness checks.


## Init Containers

 Init containers is specialized containers that run before app containers in a Pod. 

If you specify multiple init containers for a Pod, kubelet runs each init container **sequentially**. Each init container must succeed before the next can run. When all of the init containers have run to completion, kubelet initializes the application containers for the Pod and runs them as usual.

Here are some common use cases for init-containers:
- **Initializing data**: An init-container can be used to download data or configuration files that the main container needs to operate. For example, an init-container might download database schemas or configuration files and make them available to the main container.
- **Waiting for dependencies**: Sometimes, an application may depend on other services or resources to be available before it can start running. An init-container can be used to wait for these dependencies to become available before starting the main container.
- **Running pre-flight checks**: An init-container can run pre-flight checks to ensure that the main container is able to run successfully. For example, it might check that required directories or ports are available, or that certain dependencies are installed.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: workdir
      mountPath: /usr/share/nginx/html
  # These containers are run during pod initialization
  initContainers:
  - name: install
    image: busybox:1.28
    command:
    - wget
    - "-O"
    - "/work-dir/index.html"
    - http://info.cern.ch
    volumeMounts:
    - name: workdir
      mountPath: "/work-dir"
  dnsPolicy: Default
  volumes:
  - name: workdir
    emptyDir: {}
```

[Lab: Init containers in use](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/#init-containers-in-use)


## Ephemeral Containers

Ephemeral containers is a special type of container that runs temporarily in an existing Pod to accomplish user-initiated actions such as troubleshooting. You use ephemeral containers to inspect services rather than to build applications.

Ephemeral containers are useful for interactive troubleshooting when kubectl exec is insufficient because a container has crashed or a container image doesn't include debugging utilities.

```bash
# create a normal pod
kubectl run nginx --image=nginx

# debug specific pod
# The --target parameter targets the process namespace of another container. It's necessary here because kubectl run does not enable process namespace sharing in the pod it creates.
kubectl debug -it nginx --image=busybox:1.28 --target=nginx
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20230319162048.png)

[Lab: Debugging with an ephemeral debug container](https://kubernetes.io/docs/tasks/debug/debug-application/debug-running-pod/#ephemeral-container)

## Pod Quality of Service Classes（QoS）

Kubernetes assigns every Pod a QoS class based on the resource **requests** and **limits** of its component Containers. QoS classes are used by Kubernetes to decide which Pods to evict from a Node experiencing Node Pressure. The possible QoS classes are **Guaranteed**, **Burstable**, and **BestEffort**. 

When a Node runs out of resources, Kubernetes will first evict BestEffort Pods running on that Node, followed by Burstable and finally Guaranteed Pods. When this eviction is due to resource pressure, **only Pods exceeding resource requests are candidates for eviction.**


### Guaranteed

For a Pod to be given a QoS class of Guaranteed:
- Every Container in the Pod must have a memory limit and a memory request.
- For every Container in the Pod, the memory limit must equal the memory request.
- Every Container in the Pod must have a CPU limit and a CPU request.
- For every Container in the Pod, the CPU limit must equal the CPU request.


```yaml
apiVersion: v1
kind: Pod
metadata:
  name: example-pod
spec:
  containers:
  - name: nginx-container
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 256Mi
      limits:
        cpu: 100m
        memory: 256Mi
```

### Burstable

A Pod is given a QoS class of Burstable if:
- The Pod does not meet the criteria for QoS class Guaranteed.
- At least one Container in the Pod has a memory or CPU request or limit.


### BestEffort

A Pod is BestEffort only **if none of the Containers in the Pod have a memory limit or a memory request, and none of the Containers in the Pod have a CPU limit or a CPU request.** Containers in a Pod can request other resources (not CPU or memory) and still be classified as BestEffort.


