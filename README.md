# kubernetes-at-home
Summary of my experience building and running Kubernetes cluster at home

# Introduction

In this article, I share my experience building a Kubernetes cluster at home.

I started hosting applications on Linux computers more than 10 years ago. Initially, these were installed directly on the OS. Later, for convenience, I switched to their containerized versions using Docker Compose. Enabling port forwarding on my home router allowed me to access these applications from anywhere, but providing HTTPS access to multiple applications in this setup would require a much more complex network configuration.

As I began using these applications more frequently, ensuring their resilience became increasingly important. While preparing for my CKA and CKAD exams, I decided to build a Kubernetes cluster at home to address these limitations.

To achieve a minimum level of storage resiliency, at least three storage nodes are required. Of course, application and storage clusters must share the same hardware; otherwise, you would need at least seven hosts. To keep costs low, all I needed to do was purchase two inexpensive mini PCs in addition to my two existing Linux hosts. This allowed me to build a cluster with one master node and three worker nodes.

<img width="766" height="212" alt="nodes" src="https://github.com/user-attachments/assets/0684b719-a6ec-4b55-bd52-1cad0ed85c09" />

# Learnings summary

Investing time and some money in setting up a minimalist Kubernetes cluster on 4 Linux hosts was worth it. The applications I run at home have gained resilience, and I now spend less time managing them. In addition, I can now host more applications exposed to the internet at home, thanks to the reverse proxy running in the Kubernetes cluster.

# Installation with ‘kubeadm’

Overall, the Kubernetes documentation is excellent, especially the ‘kubeadm’-related sections. I simply followed the steps outlined on the Bootstrapping clusters with kubeadm page. For adding worker nodes, I referred to the Adding Linux worker nodes page.
I tried several CNI plugins with varying degrees of success and ultimately chose Cilium. One additional point worth mentioning—since the documentation is not very clear on this step—is that when you add a worker node, you must not forget to approve the Certificate Signing Requests (CSRs) generated for the newly added worker nodes.
Because I chose to proceed with a single Control Plane node, I immediately tested the cluster backup and restore process in case that node’s hardware fails completely. All Kubernetes files on the Master node are backed up daily, and the backup files are stored in multiple locations, both locally and in the cloud. For more details, see the backup/restore section.

# Next steps

I would like to see if I can move one of the three worker nodes to my friend’s house and make it accessible via a site-to-site VPN from my home network. Even though we both have fiber internet, I wonder if the network latency will be low enough in this case. It would be equivalent to having one of the worker nodes in another “availability zone,” using public cloud terminology.

# List of installed components and applications

## Core components

Cilium CNI plugin, Rook Ceph, Redis, cert-manager, MetalLB, CloudNativePG, Nginx Ingress Controller, Prometheus/Grafana

## Applicatons

### NextCloud

I have been using NextCloud for more than 8 years now. Synchronization clients are available for all platforms and are extremely efficient. To run it efficiently on Kubernetes, S3-compatible storage, external Redis, and PostgreSQL are essential.

<img width="535" height="447" alt="my_nextcloud" src="https://github.com/user-attachments/assets/edb82579-3814-4e5f-80b9-ded2df2a4b1d" />

### GitLab Community Edition

This was the most challenging part of the installation, as finding the right values for the custom Helm chart is incredibly difficult. Just like with NextCloud, using external S3, Redis, and PostgreSQL options ensures resilience and high performance.
Both NextCloud and GitLab are accessible from anywhere, thanks to the ‘cert-manager’ and Nginx ingress controller.

<img width="711" height="486" alt="my_gitlab" src="https://github.com/user-attachments/assets/c768539f-93ec-42b8-9725-aa4b09acaebf" />

### Custom Jump Server

I have also built a custom jump server. It is based on ‘ssh’ on the server side and ‘autossh’ on the client side. I use it to remotely manage several Linux machines belonging to my family members.

# Storage

I chose Rook Ceph, although I first tried Longhorn. Rook Ceph is rock-solid and reliable. I already tried replacing an SSD drive on one of the three worker nodes and was surprised by how smoothly the data recovery process went.

<img width="1034" height="604" alt="ceph_pools" src="https://github.com/user-attachments/assets/4af584ab-9738-47fc-8dc3-a8884541e865" />

# Network

I started using wi-fi network on all nodes in the beginning, as I hated the idea of having to manage cables. I suspected that I would have to use LAN at the end, but I wanted to give it a try. I saw very quickly that high and variable latency of the wi-fi network leads to poor performance of Ceph storage. So I had to get a LAN switch and once all worker nodes were connected via 1Gb Lan, the performance issues of Ceph object and file storage were gone.

Getting MetalLB and NGINX Ingress Controller in reverse proxy mode up and running was pretty straigh forward.

# Security

Single ‘cert-manager’ installation is used to manage let’s encrypt certificates for all publicly exposed applications hosted in the Kubernetes cluster. 

# CNPG

Single PostgresQL cluster serves NextCloud and GitLab applications. I have setup automatic backups for the CNPG cluster in addition to WAL archiving via Barman Cloud Plugin.

# Upgrading Kubernetes

I have performed already the upgrade to version 1.33.1 and again the Kubernetes documentation was clear and easy to follow. For now it is done manually, but it takes only 10 minutes for all four nodes. I guess I can automate it, but I will look at it later.

# Backup and Restore

## Backup script

The script below should be ran on the Master node in crontab as root

```
#!/bin/bash
DTS=`date "+%Y%m%d_%H%M%S"`
mkdir -p /temp/clbackup/etcd/
mkdir -p /temp/clbackup/certs/
mkdir -p /temp/clbackup/kubeadm-config/
mkdir -p /temp/clbackup/etc-hosts/

etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt --key=/etc/kubernetes/pki/etcd/healthcheck-client.key snapshot save /temp/clbackup/etcd/etcd-snapshot-latest.db

cp -r /etc/kubernetes/pki /temp/clbackup/certs/
cp kubeadm_config.yaml /temp/clbackup/kubeadm-config/
cp /etc/hosts /temp/clbackup/etc-hosts/

tar czvf /home/{user}/$DTS-clbackup.tar.gz /temp/clbackup/

rm -rf /temp/clbackup
```

## Restore run book

### PREPARATION

1. Make the new server aquire the IP address of the original master node server. make modifications on dhcp server and renew IP using ‘dhcpclient’
2. Make the new server get the hostname of the original master node server.

```
sudo cp /etc/hostname /etc/hostname.saved
sudo hostnamectl set-hostname masternode
```

exit and login again, the prompt should have the name of the master node, like this:

```
user@masternode:~$ 
```

### Restore kubernetes files

if needed, bootstrap kubernetes, install etcdctl (‘sudo apt-get install -y etcd-client’), install cilium CLI (https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)

```
tar xzvf backup.tar.gz -C .

sudo cp -r backup/certs/* /etc/kubernetes/pki
sudo cp backup/kubeadm-config/kubeadm_config.yaml kubeadm_config.yaml
sudo cp backup/etc-hosts/hosts /etc/hosts
sudo ETCDCTL_API=3 etcdctl snapshot restore backup/etcd/etcd-snapshot-latest.db --data-dir=/var/lib/etcd --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key
```
