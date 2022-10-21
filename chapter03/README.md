- [What is Kubernetes?](#what-is-kubernetes)
- [Kubernetes Achitecture](#kubernetes-achitecture)
- [Main Kubernetes Componentes](#main-kubernetes-componentes)
  - [Node & Pod](#node--pod)
  - [Service & Ingress](#service--ingress)
  - [Configmap & Secret](#configmap--secret)
  - [Volume](#volume)
  - [Deployment & StatefulSet](#deployment--statefulset)
- [Kubernetes Configuration](#kubernetes-configuration)
- [Kubectl](#kubectl)
- [Deploy Kubernetes](#deploy-kubernetes)
- [Demo](#demo)
- [Reference](#reference)


## What is Kubernetes?
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221020211252.png)


## Kubernetes Achitecture

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221020212810.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221020214413.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221020214603.png)

## Main Kubernetes Componentes

### Node & Pod

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221020220030.png)

### Service & Ingress

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021172848.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021175300.png)


### Configmap & Secret

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021173458.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021191730.png)



### Volume

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021174037.png)


### Deployment & StatefulSet

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021174737.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021174946.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021175014.png)



## Kubernetes Configuration

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021175757.png)



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021180009.png)

## Kubectl

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20221021181126.png)

## Deploy Kubernetes

- [Deploy Kubernetes Using Kubeadm in Ubuntu Stey by Stey](https://github.com/cr7258/golang-learning/tree/master/%E6%9E%81%E5%AE%A2%E6%97%B6%E9%97%B4%E4%BA%91%E5%8E%9F%E7%94%9F%E8%AE%AD%E7%BB%83%E8%90%A5/homework/module4)
- [Managed Kubernetes service in cloud vendors(AWS, GCP, Azure, DigitalOcean)](https://cloud.digitalocean.com/)
- Run Kubernetes in your local machine quickly
  - [Minikube](https://minikube.sigs.k8s.io/docs/)
  - [Kind](https://kind.sigs.k8s.io/)
  - [K3d](https://k3d.io/v5.4.6/)

## Demo

Deploy a cluster using minikube. Useful parameter:

- `-n`: The number of nodes to spin up. Defaults to 1.
- `--image-repository`:  Alternative image repository to pull docker images from. This can be used when you have limited access to gcr.io. Set it to "auto" to let minikube decide one for you. For Chinese mainland users, you may use local gcr.io mirrors such as registry.cn-hangzhou.aliyuncs.com/google_containers.
- `--kubernetes-version`:  The Kubernetes version that the minikube VM will use (ex: v1.2.3, 'stable' for v1.25.0, 'latest' for v1.25.0). Defaults to 'stable'.
- `--driver`: Driver is one of: qemu2 (experimental), docker, podman (experimental), ssh (defaults to auto-detect)
- `--cpus`: Number of CPUs allocated to Kubernetes. Use "max" to use the maximum number of CPUs.
- `--memory`: Amount of RAM to allocate to Kubernetes (format: <number>[<unit>],where unit = b, k, m or g). Use "max" to use the maximum amount of memory.
- `--disk-size`: Disk size allocated to the minikube VM (format: <number>[<unit>], where unit = b, k, m or g).
-  `-p, --profile`: The name of the minikube VM being used. This can be set to allow having multiple instances of minikube independently. (default "minikube")

```
minikube start \
-n 3 \
-p demo-cluster \
--image-repository="registry.cn-hangzhou.aliyuncs.com/google_containers"
```

Deploy a cluster using kind.(Recommend)

```yaml
# three node (two workers) cluster config
# demo-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

```bash
# create cluster
kind create cluster --name demo-cluster --config demo-cluster.yaml

# delete cluster
kind delete clusters demo-cluster
```

nginx Deployment with 3 replicas.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
```

Service expose nginx Deployment.

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  ports:
  - port: 10000 # Service port
    protocol: TCP
    targetPort: 80 # nginx pod port
  selector: # match nginx pod label
    app: nginx
```



## Reference

- [Kubernetes Architecture: 11 Core Components Explained](https://spot.io/resources/kubernetes-architecture-11-core-components-explained)
- [Kubernetes Crash Course for Absolute Beginners](https://www.youtube.com/watch?v=s_o8dwzRlu4)
- [minikube quick start](https://minikube.sigs.k8s.io/docs/start/)
