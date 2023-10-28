### Kubernetes setup with Canonical Multipass

#### Multipass

[Multipass](https://multipass.run/) is an operating system virtualization tool from [Canonical](https://canonical.com/), the orgnization behind the most popular linux based operating system [Ubuntu](https://ubuntu.com/). 

#### Kubernetes

[Kubernetes](https://kubrnetes.io/) is a container orchestration tool, originally developed by [Google](https://www.google.com) and later donated to [Cloud Native Computing Foundation](ihttps://www.cncf.io/). It is one of the most popular tool. Some of the other orchestration tools are [Docker Swarm](https://docs.docker.com/engine/swarm/) and [Apache Mesos](https://mesos.apache.org/). Docker Swarm is a simple tool with easy setup but limited functionality. On the other hand, Apache Mesos is hard to setup, but provides an array of functionality. Kubernetes sits between these two, providing the right amount of functionality with appropriate configuration.

##### What's in this tutorial

The idea of this repository is to provide a birds' eye view of how a Kubrnetes cluster is setup. We use the Multipass virtualization tool to create ubuntu nodes. Also, utlized is [Cloud init](https://cloud-init.io/) to create initial configuration for the nodes in the cluster.

##### Lets get started

Before we begin, we assume that you have a brief idea of the Kubrnetes cluster architecture. In the cluster, there must be master node and one or more worker node. We shall here provision one master node and two worker nodes.
For the master node, the following hardware is recommended
    - 2 vCPU
    - 2 GB RAM
    - 10 GB disk

For the worker node, the following hardware is recommended
    - 1 vCPU
    - 1 GB RAM
    - 10 GB disk

The host machine runs Ubutu Jammy. In order to install `multipass`, we'll snap store

```bash
sudo snap install multipass lxd
```

For Linux based host machine, multipass can use LXD or QEMU as the driver. For Windows based host machine, its possible to use LXD, Docker or VirtualBox as the drivers. For this project, we need to ensure that Multipass is using LXD as its local driver.

```
multipass set local.driver=lxd
multipass get local.drier
```