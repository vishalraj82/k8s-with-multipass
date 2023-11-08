## Kubernetes setup with Canonical Multipass

### Multipass

[Multipass](https://multipass.run/) is an operating system virtualization tool from [Canonical](https://canonical.com/), the orgnization behind the most popular linux based operating system [Ubuntu](https://ubuntu.com/). 

### Kubernetes

[Kubernetes](https://kubrnetes.io/) is a container orchestration tool, originally developed by [Google](https://www.google.com) and later donated to [Cloud Native Computing Foundation](ihttps://www.cncf.io/). It is one of the most popular tool. Some of the other orchestration tools are [Docker Swarm](https://docs.docker.com/engine/swarm/) and [Apache Mesos](https://mesos.apache.org/). Docker Swarm is a simple tool with easy setup but limited functionality. On the other hand, Apache Mesos is hard to setup, but provides an array of functionality. Kubernetes sits between these two, providing the right amount of functionality with appropriate configuration.

### What's in this tutorial

The idea of this repository is to provide a birds' eye view of how a Kubrnetes cluster is setup. We use the Multipass virtualization tool to create ubuntu nodes. Also, utlized is [Cloud init](https://cloud-init.io/) to create initial configuration for the nodes in the cluster. While there are many solutions out there, which let you create a local kubernetes cluster, but they can hide a lot of nitty gritty behind the scenes, providing you with a readymade playground. In order to understand, the basics of a K8s cluster setup, I have done this.

### Before getting started

Before we begin, we assume that you have a brief idea of the Kubrnetes cluster architecture. In the cluster, there must be at least one master node and one or more worker node. We shall here provision one master node and two worker nodes.

For the master node, the following hardware is recommended
    - 2 vCPU
    - 4 GB RAM
    - 10 GB disk

For the worker node, the following hardware is recommended
    - 2 vCPU
    - 2 GB RAM
    - 10 GB disk

The host machine runs Ubutu Jammy. In order to install `multipass`, we'll use the snap store

```bash
sudo snap install multipass lxd
```

For Linux based host machine, multipass can use LXD or QEMU as the driver. For Windows based host machine, its possible to use LXD, Docker or VirtualBox as the drivers. For this project, we need to ensure that Multipass is using LXD as its local driver.

```
multipass set local.driver=lxd
multipass get local.drier
```

For our purpose, we're going to use __Ubuntu Jammy__ (LTS). In order to see the list of OS images supported by multipass

```bash
multipass find
```

### Launching the nodes

Lets get started with launching the cluster nodes. First we'll begin with the `master` node and then 2 worker nodes. We need to wait a couple of minute after the prompt returns from the launch as initialization can take sometime. The multipass prompt waits for a defautl time of __300__ seconds. This can be changed by specifying the parameter `--timeout <timeout>` during the launch operation.

```bash
multipass launch --name master --cpus 2 --memory 4G --disk 10G --cloud-init cloud-init.yml jammy

multipass launch --name node1 --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init.yml  jammy

multipass launch --name node2 --cpus 2 --memory 2G --disk 10G --cloud-init cloud-init.yml  jammy
```

Once all the nodes are launched, we'll restart the nodes. The command `multipass restart` can take a list of nodes to restart of the flag `--all` to restart all running nodes

```bash
multipass restart --all

multipass restart master node1 node2

```

Once all the nodes are ready, we'll first proceed to have the master node ready. In order to gain shell access to mater node, one of the below command would work. In the home directory of the user `ubuntu`, a script with name `kubeadm-init` is available.

```bash
multipass shell master

multipass exec master /bin/bash
```

In order to find the IP of the master node, run the command

```bash
ifconfig
```

From the abve commandy it can be found out what network interface the master node has and its corresponding assigned IP. Finally, we'll use the calico networking plugin.

```bash
MASTER_PRIVATE_IP=$(ip addr show <network-interface> | awk '/inet / {print $2}' | cut -d/ -f1)
NODENAME=$(hostname -s)
POD_CIDR="172.16.0.0/16"
sudo kubeadm init \
    --apiserver-advertise-address="$MASTER_PRIVATE_IP" \
    --apiserver-cert-extra-sans="$MASTER_PRIVATE_IP" \
    --pod-network-cidr="$POD_CIDR" \
    --node-name "$NODENAME" \
    --kubernetes-version 1.28.3

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

The above command will initialize all the necessary kubernetes components on the master node. On the master node, the command `kubectl` can either be used as root user or as any non-root user. For both the cases, the instructions are printed at the end of the aforementioned command. Towards the last of the output, a `kubeadm` command is printed with token, which can then be used by the worker nodes to join the master node.

```bash
multipass exec node1 -- sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha256-token>

multipass exec node2 -- sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<sha256-token>
```

Once the worker nodes join the cluster, this can again be verifed on the master node

```bash
multipass exec master -- kubectl get nodes
```

This would list all the nodes in the cluster, including the master node.

### Local kubernetes cluster in action

Login to the shell for master node and create two files

```bash
# nginx-deployment.yml
apiVersion: apps/v1
kind: Deployment
metatata:
  name: nginx-deployment
  namespace: default
template:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  spec:
    containers:
      - name: nginx-pod
        image: nginx:alpine
        ports:
          - containerPort: 80
            name: nginx-pod-port
```

```bash
# nginx-service.yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
    - protocol: TCP
      port: 80
      targetPort: nginx-pod-port
      nodePort: 30000
```

First create the deployment followed by the service

```bash
kubectl apply -f nginx-deployment.yml
kubectl apply -f nginx-service.yml
```

Now exit from the master shell, open the browser and hit - `http://<master-node-ip>:3000` and you should see the nginx welcome screen. For more information, refer to the [official kubernetes documentation](https://kubernetes.io/docs/home).
