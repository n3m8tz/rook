---
title: Multi-Node Test Environment
weight: 91
indent: true
---

# Multi-Node Test Environment

## Setup expectation

There are a bunch of pre-requisites to be able to deploy the following environment. Such as:

* A Linux workstation (CentOS or Fedora)
* KVM/QEMU installation

For other Linux distribution, there is no guarantee the following will work.
However adapting commands (apt/yum/dnf) could just work.

## Prerequisites installation

On your host machine, execute `tests/scripts/multi-node/rpm-system-prerequisites.sh`.

## Deploy Kubernetes with Kubespray

Clone it:

```bash
git clone https://github.com/kubernetes-incubator/kubespray
cd kubespray
```

In order to successfully deploy Kubernetes with Kubespray, you must have this code: https://github.com/kubernetes-incubator/kubespray/pull/2153 and https://github.com/kubernetes-incubator/kubespray/pull/2271.

Edit `inventory/group_vars/k8s-cluster.yml` with:

```bash
docker_options: "--insecure-registry=172.17.8.1:5000 --insecure-registry={{ kube_service_addresses }} --graph={{ docker_daemon_graph }}  {{ docker_log_opts }}"
```

FYI: `172.17.8.1` is the libvirt bridge IP, so it's reachable from all your virtual machines.
This means a registry running on the host machine is reachable from the virtual machines running the Kubernetes cluster.

Create Vagrant's variable directory:

```bash
mkdir vagrant/
```

Put `tests/scripts/multi-node/config.rb` in `vagrant/`. You can adapt it at will.
Feel free to adapt `num_instances`.

Deploy!

```bash
vagrant up --no-provision ; vagrant provision
```

Go grab a coffee:

```
PLAY RECAP *********************************************************************
k8s-01                     : ok=351  changed=111  unreachable=0    failed=0
k8s-02                     : ok=230  changed=65   unreachable=0    failed=0
k8s-03                     : ok=230  changed=65   unreachable=0    failed=0
k8s-04                     : ok=229  changed=65   unreachable=0    failed=0
k8s-05                     : ok=229  changed=65   unreachable=0    failed=0
k8s-06                     : ok=229  changed=65   unreachable=0    failed=0
k8s-07                     : ok=229  changed=65   unreachable=0    failed=0
k8s-08                     : ok=229  changed=65   unreachable=0    failed=0
k8s-09                     : ok=229  changed=65   unreachable=0    failed=0

Friday 12 January 2018  10:25:45 +0100 (0:00:00.017)       0:17:24.413 ********
===============================================================================
download : container_download | Download containers if pull is required or told to always pull (all nodes) - 192.44s
kubernetes/preinstall : Update package management cache (YUM) --------- 178.26s
download : container_download | Download containers if pull is required or told to always pull (all nodes) - 102.24s
docker : ensure docker packages are installed -------------------------- 57.20s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 52.33s
kubernetes/preinstall : Install packages requirements ------------------ 25.18s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 23.74s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 18.90s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 15.39s
kubernetes/master : Master | wait for the apiserver to be running ------ 12.44s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 11.83s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 11.66s
kubernetes/node : install | Copy kubelet from hyperkube container ------ 11.44s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 11.41s
download : container_download | Download containers if pull is required or told to always pull (all nodes) -- 11.00s
docker : Docker | pause while Docker restarts -------------------------- 10.22s
kubernetes/secrets : Check certs | check if a cert already exists on node --- 6.05s
kubernetes-apps/network_plugin/flannel : Flannel | Wait for flannel subnet.env file presence --- 5.33s
kubernetes/master : Master | wait for kube-scheduler -------------------- 5.30s
kubernetes/master : Copy kubectl from hyperkube container --------------- 4.77s
[leseb@tarox kubespray]$
[leseb@tarox kubespray]$
[leseb@tarox kubespray]$ vagrant ssh k8s-01
Last login: Fri Jan 12 09:22:18 2018 from 192.168.121.1

[vagrant@k8s-01 ~]$ kubectl get nodes
NAME      STATUS    ROLES         AGE       VERSION
k8s-01    Ready     master,node   2m        v1.9.0+coreos.0
k8s-02    Ready     node          2m        v1.9.0+coreos.0
k8s-03    Ready     node          2m        v1.9.0+coreos.0
k8s-04    Ready     node          2m        v1.9.0+coreos.0
k8s-05    Ready     node          2m        v1.9.0+coreos.0
k8s-06    Ready     node          2m        v1.9.0+coreos.0
k8s-07    Ready     node          2m        v1.9.0+coreos.0
k8s-08    Ready     node          2m        v1.9.0+coreos.0
k8s-09    Ready     node          2m        v1.9.0+coreos.0
```

## Development workflow on the host

Everything should happen on the host, your development environment will reside on the host machine NOT inside the virtual machines running the Kubernetes cluster.

Now, please refer to [https://rook.io/docs/rook/master/development-flow.html](https://rook.io/docs/rook/master/development-flow.html) to setup your development environment (go, git etc).

At this stage, Rook should be cloned on your host.

From your Rook repository (should be $GOPATH/src/github.com/rook) location execute `bash tests/scripts/multi-node/build-rook.sh`.
During its execution, `build-rook.sh` will purge all running Rook pods from the cluster, so that your latest container image can be deployed.
Furthermore, **all Ceph data and config will be purged** as well.
Ensure that you are done with all existing state on your test cluster before executing `build-rook.sh` as it will clear everything.

Each time you build and deploy with `build-rook.sh`, the virtual machines (k8s-0X) will pull the new container image and run your new Rook code.
You can run `bash tests/scripts/multi-node/build-rook.sh` as many times as you want to rebuild your new rook image and redeploy a cluster that is running your new code.

From here, resume your dev, change your code and test it by running `bash tests/scripts/multi-node/build-rook.sh`.


## Teardown

Typically, to flush your environment you will run the following from within kubespray's git repository.
This action will be performed on the host:

```bash
[user@host-machine kubespray]$ vagrant destroy -f
```

Also, if you were using `kubectl` on that host machine, you can resurrect your old configuration by renaming `$HOME/.kube/config.before.rook.$TIMESTAMP` with `$HOME/.kube/config`.

If you were not using `kubectl`, feel free to simply remove `$HOME/.kube/config.rook`.
