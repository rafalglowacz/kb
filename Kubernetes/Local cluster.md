# Local Kubernetes cluster setup

This document describes how to create a local K8s cluster for learning and testing new things before using them in remote clusters.

## Network prerequisites

Aside from the pod network which will live entirely inside the cluster, we need to consider networking that involves the outside world:

- the cluster needs to be able to access the internet
- you need to be able to communicate with the nodes in the cluster from your host

As far as I was able to figure out, networking setup depends on the virtualization software used:

- VirtManager: you need just one network. There's a NAT network created by default in which the host gets the first IP (192.168.122.0/16 on my machine) and the guests can use IPs in the same network. The host is the default gateway, so this setup handles both internet access and host-guest communication
- VirtualBox: you need a bridged network for access to the internet and a host-only network for host-guest communication.
    - you could also use the bridged network for both purposes, but I prefer it when my VMs don't depend on my router

From now on the instructions will assume the following set of 2 outside-world networks, you can simplify to 1 if that's your choice:

- 192.168.122.0/24 - bridged network, specific IPs assigned by DHCP
- 172.10.0.0/16 - host-only network, host has IP 1, Kubernetes master has IP 2, other Kubernetes nodes have IP 3 and up

And for cluster internal networking we'll use:

- 172.30.0.0/16

## Installing OS in VMs

These instructions were written while using Ubuntu Server 20.04. Initial steps will be done on all the nodes we create, and then later there will be some steps depending on whether it's a master or a slave. As mentioned before, I'm assuming there's no VM cloning involved, I run into some issues when using it (maybe caused by something else completely, I might revisit the topic in the future).

After creating the VM don't forget to:

- add one more CPU core
- set up networking

Other resource considerations:

- I initially gave the guests 4G of RAM, but that turned out to be insufficient for Kubernetes runners building Docker images inside the cluster. So I upped it to 8G, although 6G would probably have been fine.
- for the same reason, I started with 20 or 30G of disk space, but ended up increasing it to 50G. Now I have 100G as I had the space but 50G should be fine.

When installing:

- use manual settings for IP v4, e.g.:
    - Subnet: 192.168.122.0/24
    - Address: 192.168.122.2
    - Gateway: 192.168.122.1
    - Name servers: 8.8.8.8,8.8.4.4
- **don't** select Docker on the screen that allows to automatically install extra stuff. It didn't work for me.
- install OpenSSH server and import keys from GitHub - optional, but a great time-saver

## Preparing node

If using VirtualBox, after the installation run `shutdown now` in the guest and then on the host run:

```shell
vboxmanage modifyvm kubernetes-NODE_HOSTNAME --defaultfrontend headless
```

This will configure the guest to not show its screen when it's launched in VirtualBox. In VirtManager it works like this by default.

Start the machine and from now on use SSH to control the guest.

```shell
sudo swapoff -a
sudo sed -i 's/\(\/swap.img\)/#\1/' /etc/fstab
```

Install Docker:

- https://docs.docker.com/engine/install/ubuntu/
- https://docs.docker.com/engine/install/linux-postinstall/ (no sudo, optional)

Edit /etc/docker/daemon.json

```json
{
  "exec-opts": ["native.cgroupdriver=systemd"]
}
```

If you're going to use a local Docker registry, you might want to also add:

```json
"insecure-registries": ["172.10.0.1:50000"]
```

I'm assuming here that the registry will run on the host, not inside the cluster.

```shell
sudo systemctl restart docker
```

Install kubeadm:

- before clicking through to instructions, note that:
    - "Verify the MAC address and product_uuid..." - this will be necessary if you want to save some time by creating a base VM that you'll clone to create more nodes. In this guide I'm assuming that each node will be created manually.
    - "Check required ports" - you can skip that
    - "Configuring a cgroup driver" - we already did that, so you can skip it now
- instructions: https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

## Initialize cluster

### Master

Command template:

> sudo kubeadm init --apiserver-advertise-address=MASTER_IP --pod-network-cidr=POD_NETWORK

For instance:

```shell
sudo kubeadm init --apiserver-advertise-address=172.10.0.2 --pod-network-cidr=172.30.0.0/16
```

It will output a command you'll need to join the cluster later, I recommend saving it to a file, e.g. `join.txt`

Then follow the rest of the commands from the link below:

- before going to the instructions, note that:
    - instead of `kubectl create -f https://docs.projectcalico.org/manifests/custom-resources.yaml`, you'll have to do:
        - `wget https://docs.projectcalico.org/manifests/custom-resources.yaml`
        - replace the network it specifies with `172.30.0.0`
        - use the local file: `kubectl create -f custom-resources.yaml`
- instructions: https://docs.projectcalico.org/getting-started/kubernetes/quickstart

### Node

Run join command shown by kubeadm init from master

### Errors

If `kubeadm` shows an error and something needs to be corrected, then when you re-run it, it'll probably complain about something and exit. In that case you can run it again with the `--ignore-preflight-errors=all` parameter, and it should complete successfully.

## Controlling the cluster from the host

Copy /etc/kubernetes/admin.conf from the master and put it in ~/.kube/config, install `kubectl`.

- e.g. `pamac install kubectl` or `dnf install kubernetes-client`, but it's good to first check if it has the same version as in the cluster, as significantly different versions might not work together
- chmod 600 ~/.kube.config (some commands will complain if the permissions are visible to others)