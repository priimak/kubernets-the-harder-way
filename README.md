Kubernets The Harder Way
========================

"_When you come to fork in the road take it_" - Yogi Berra

Here we describe set of steps necessary to install Kubernetes from scratch. These
steps are solely for educational purposes.

## Basic Prerequisites

We are assuming that you have two nodes `k1` and `k2` on which you will be installing
Kubernetes and the installation will proceed from some other machine. These are the
_kubernetes nodes_. You should have
password-less access to these nodes (`k1` and `k2`) set up using either password-less
ssh key or with key added to ssh agent. Thus you should be able to login into these nodes
using simply commands `ssh k1` and `ssh k2`. The account which you will be using to
login into kubernetes nodes should also have password-less sudo access set up. One way
of doing it is to make sure that your user is member of `wheel` group and file
`/etc/sudoers` has following entry `%wheel  ALL=(ALL)       NOPASSWD: ALL`.

## Download all Kubernetes binaries and install Control Plane

On the machine from which you will be installing kubernetes from create directory `~/kubernetest-the-harder-way`
and then go into that directory. From that point on all actions performed below should happen
from that directory.

    mkdir ~/kubernetest-the-harder-way
    cd ~/kubernetest-the-harder-way

Download all necessary Kubernetes binaries

    export K8V=$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)
    for i in kubectl kube-apiserver kube-controller-manager kube-scheduler kubelet kube-proxy; do
        curl -LO https://storage.googleapis.com/kubernetes-release/release/$K8V/bin/linux/amd64/$i
    done
    curl -LO https://github.com/opencontainers/runc/releases/download/v1.0.0-rc10/runc.amd64
    curl -LO https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.18.0/crictl-v1.18.0-linux-amd64.tar.gz
    curl -LO https://github.com/containernetworking/plugins/releases/download/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tgz
    curl -LO https://github.com/containerd/containerd/releases/download/v1.3.3/containerd-1.3.3.linux-amd64.tar.gz

    mv runc.amd64 runc
    tar zxvf crictl-v1.18.0-linux-amd64.tar.gz

Copy control plane software onto node `k1`

    for i in kubectl kube-apiserver kube-controller-manager kube-scheduler; do
        scp $i k1:
        ssh k1 sudo install $i /usr/local/bin
        ssh k1 rm $i
    done


## Create SSL Certificates

Record host names and ip addresses for the Kubernetes hosts

    K1IP=$(host k1 | grep has | sed 's/.*has address //')
    K2IP=$(host k2 | grep has | sed 's/.*has address //')

All Kubernetes components talk to each other over ssl and use ssl client
auth. Thus we need to create bunch of ssl certificates. To create them we
will use `ssl-cert-maker.sh` which is just a wrapper over openssl command
line tool.

    curl -LO https://raw.githubusercontent.com/priimak/ssltk/master/ssl-cert-maker.sh
    chmod 755 ssl-cert-maker.sh
    touch ~/.rnd

Generate new CA signing certificate

    ./ssl-cert-maker.sh new_ca_cert KubernetesCA 1024

Create certificate for `admin` user

    ./ssl-cert-maker.sh new_signed_cert KubernetesCA admin 1024


Create certificates `system:kube-controller-manager`, `system:kube-proxy`, `system:kube-scheduler`
and `service-account` all signed by `KubernetesCA`

    ./ssl-cert-maker.sh new_signed_cert KubernetesCA system:kube-controller-manager 1024
    ./ssl-cert-maker.sh new_signed_cert KubernetesCA system:kube-proxy 1024
    ./ssl-cert-maker.sh new_signed_cert KubernetesCA system:kube-scheduler 1024
    ./ssl-cert-maker.sh new_signed_cert KubernetesCA service-account 1024

For each of the worker node make certificate with CN `system:node:$node_name`
and alt names that point to DNS name and an ip address

    ./ssl-cert-maker.sh new_signed_cert KubernetesCA system:node:k1 1024 --subjectAltName=$K1IP --subjectAltName=k1
    ./ssl-cert-maker.sh new_signed_cert KubernetesCA system:node:k2 1024 --subjectAltName=$K2IP --subjectAltName=k2

For api server that will run on node `k1` make cert with CN `kubernetes` and related alt names

    ./ssl-cert-maker.sh new_signed_cert KubernetesCA kubernetes 1024 --subjectAltName=$K1IP --subjectAltName=k1

At the end you should have following certificated created in `~/.ssl/certs`

    admin.crt
    kubernetes.crt
    service-account.crt
    system:kube-controller-manager.crt
    system:kube-proxy.crt
    system:kube-scheduler.crt
    system:node:k1.crt
    system:node:k2.crt

and signing CA cert, key and revocation list in `~/.ssl/ca`

    ca.KubernetesCA.crt
    ca.KubernetesCA.key
    ca.KubernetesCA.srl

Distribute all certificates onto all Kubernetes nodes

    scp -r ~/.ssl k1:
    scp -r ~/.ssl k2:

## Create all config files
