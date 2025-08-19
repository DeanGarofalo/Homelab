# My Homelab Kubernetes Cluster

This repository contains the **GitOps-managed configuration** for my Kubernetes homelab cluster.  
All resources in this cluster are deployed, updated, and managed by FluxCD, and run on a virtualized instance of Talos, on a server I built.

---

## ⚙️ Cluster Overview

- **Cluster Type**: Single Node Kubernetes (VM-based inside of Truenas Scale)

- **Infrastructure**: OS is Talos running in Truenas Scale VM. Underlying VM storage is ZVOL on ZFS. 

- **GitOps Tooling**: FluxCD for management, Renovate for automatic updates.

- **Networking**: NGINX as ingress controller, MetalLB for load balancer IP allocation, Cilium for CNI.

- **Storage**: Longhorn as the default CSI.


---

## 📦 Core Components

### GitOps (FluxCD)

- Manages the lifecycle of all Kubernetes resources.

- Watches this GitHub repo and reconciles cluster state automatically.

- Handles Helm releases, Kustomizations, and secrets via SOPS.

- Configured with Discord push notifications and a Webhook that initializes reconciliation of GitHub push events for instant manual updates.


### Networking

- **NGINX Ingress Controller**: Provides ingress routing and TLS termination with **cert-manager**. 2 different instances are used to isolate ingress traffic from internal local traffic and external traffic.

- **MetalLB**: Allocates static load balancer IP's within the local network.

- **Cilium**: used as the default CNI.

- **Cloudflare Tunnel**: used to allow external access for FluxCD webhook without punching a hole in my firewall. Also use Cloudflare WAF to filter the noise and only allow whitelisted GitHub IP's.

- **Blocky**: is a DNS server / adblocker which also exposes all of my Kubernetes ingress resources for resolution in my local network. 

### Storage

- **Longhorn**: supports Volume Snapshots which are controlled and managed by **Volsync** and backed up to Cloudflare-R2 as S3 storage. 

- **Truenas Scale**: Used as a NFS server for longterm large data.


### Secrets Management

- **SOPS & Age**: All Kubernetes secrets are encrypted in transit and in Git.

- Decrypted at runtime inside the cluster by FluxCD.

## 📊 Monitoring & Observability 

- **Prometheus & Grafana**: for metrics and dashboards.

- **Loki**: for log aggregation.

- **Alloy**: for scraping logs for ingestion into Loki.

---

## 📁 Repository Structure
```
├── clusters
│  └── main
│     ├── clusterenv.yaml     # File of secret variables to be post build subsituted by Flux
│     ├── kubernetes
│     │  ├── apps             # HelmReleases for my applications
│     │  ├── core             # Core cluster components Ex: Blocky, ClusterIssuer
│     │  ├── flux-entry.yaml  # Entry point for Flux to bootstrap
│     │  ├── flux-system      # Flux core components and addons
│     │  ├── kube-system      # Extra Kube componments, CNI, CSR-approver, metrics server
│     │  ├── kustomization.yaml # Surface level Kustomize file
│     │  ├── monitoring       # Loki & Alloy folder
│     │  ├── networking       # NGINX controllers + cloudflared and Multus-CNI
│     │  └── system           # CSI, MetalLB, Volsync, Postgres operator
│     └── talos               # Talos configuration
├── README.md
└── repositories
```
---


## 📝 Notes


**Truenas Scale** is the host OS and serves as general storage hub using ZFS for the underlying filesystem and acts as a NFS server for some of the Pods the access data from. Truenas also runs the VM that this K8s cluster runs on.

For the virtualized OS I am using **Talos**, a secure, immutable and minimal OS. Talos is a unique take on what a operating system should look like and since this OS's sole purpose is to run a Kubernetes Cluster, Talos ends up being set it and forget it configuration and less like an OS I have to manage. My only complaint is visibility and troubleshooting can be difficult unless you have an extensive observability stack, since traditional methods are incompatible with Talos by design. 

CSI is **Longhorn**. Since this is a single node cluster, I didn't have any solid requirements out of my CSI other than I wanted Volume Snapshots for offsite S3 backups, and Longhorn fit the bill. I don't have a ton to say about Longhorn other than I dislike how it treats storage on the block level and not on the filesystem level so I need to have trim CRON jobs run otherwise it will exhaust disk space. 

CNI is **Cilium**. It's so overkill and I don't have any good reason to use it. I figured I'd grow into it and use it as a learning exercise but I haven't had to touch it since it was first deployed.

Since this is a *baremetal* Kubernetes cluster I use **MetalLB** as my load balancer. I define a range of IP's for MetalLB to use and MetalLB distributes those to Pod's requesting them, and advertises them across Layer 2 traffic. I only use load balancers for a few Pods since the majority of my deployments use Ingress.

<br>
<br>

<small>Disclaimer: This is a fork of my actual homelab, the only difference is I have removed a couple of helm charts here that I didn't want to display (they revolve around multimedia), and on the off chance I accidentally expose a non encrypted secret. I do have a pre-commit script that runs sops on the folder structure, but I'd rather be safe than sorry.</small>