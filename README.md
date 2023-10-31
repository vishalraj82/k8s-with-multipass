### Kubernetes setup with Canonical Multipass

#### Multipass

[Multipass](https://multipass.run/) is an operating system virtualization tool from [Canonical](https://canonical.com/), the orgnization behind the most popular linux based operating system [Ubuntu](https://ubuntu.com/). 

#### Kubernetes

[Kubernetes](https://kubrnetes.io/) is a container orchestration tool, originally developed by [Google](https://www.google.com) and later donated to [Cloud Native Computing Foundation](ihttps://www.cncf.io/). It is one of the most popular tool. Some of the other orchestration tools are [Docker Swarm](https://docs.docker.com/engine/swarm/) and [Apache Mesos](https://mesos.apache.org/). Docker Swarm is a simple tool with easy setup but limited functionality. On the other hand, Apache Mesos is hard to setup, but provides an array of functionality. Kubernetes sits between these two, providing the right amount of functionality with appropriate configuration.

##### What's in this tutorial

The idea of this repository is to provide a birds' eye view of how a Kubrnetes cluster is setup. We use the Multipass virtualization tool to create ubuntu nodes. Also, utlized is [Cloud init](https://cloud-init.io/) to create initial configuration for the nodes in the cluster.

##### Before getting started

Before we begin, we assume that you have a brief idea of the Kubrnetes cluster architecture. In the cluster, there must be master node and one or more worker node. We shall here provision one master node and two worker nodes.

For the master node, the following hardware is recommended
    - 2 vCPU
    - 2 GB RAM
    - 10 GB disk

For the worker node, the following hardware is recommended
    - 1 vCPU
    - 1 GB RAM
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

##### Launching the nodes

Lets get started with launching the cluster nodes. First we'll begin with the `master` node and then 1 worker node. We need to wait a couple of minute after the prompt returns from the launch as initialization can take sometime. The multipass prompt waits for a defautl time of __300__ seconds. This can be changed by specifying the parameter `--timeout <timeout>` during the launch operation.

```bash
multipass launch --name master --cpus 2 --memory 2G --disk 10G jammy

multipass launch --name worker1 --cpus2 --memory 2G --disk 10G jammy
```

Once both the nodes are ready, we'll first login to the `master` node to initialize it as master node. Post the initialization, we will also configure the non-root user (`ubuntu` in our case) to use the cluster, followed by the netwoking setup for the cluster.

```bash
mp shell master
sudo kubeadm config images pull
sudo kubeadm init --config /etc/kubernetes/kubeadm-config.yml
```

Once the initializeation is done, the networking must be configured for the cluster

```bash
mkdir $HOME/.kube

sudo cp /etc/kubernetes/admin.conf

sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://raw.githubusercontent.com/thxCode/coreos-flannel/master/Documentation/kube-flannel.yml

sudo kubeadm token create --print-join-command
```

The last comand woud print a token which can then be used by all the worker nodes to join the cluster. Now let move to the worker nodes.

```bash
multipass shell worker1
```

Copy the token command generated on the master node and paste it into the worker node terminal

```bash
sudo kubeadm join <master-node-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<discovery-token-ca-cert-hash>
```

Once all the worker nodes joins the cluster, switch back to the master node terminal to confirm

```bash
kubectl get nodes
```

Next we will set up the networking configuration. For our case, we'll use the Weave networking. On the master node

```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

kubectl get nodes
```
