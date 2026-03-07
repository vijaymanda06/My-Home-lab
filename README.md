# 🏠 Homelab — Production Kubernetes on a Laptop

![Homelab Dashboard and Terminal](assets/cover.png)

I built a fully functional 3-node Kubernetes cluster that runs 24/7 
on my personal laptop — accessible from anywhere in the world, 
including from my phone, with zero cloud costs and zero open ports.

This isn't a tutorial. It's a working setup I use to learn and 
experiment with the same tools used in real production environments.

## Why I Built This

I kept running into a problem at work — I wanted to test real 
infrastructure setups, but spinning up EKS or GKE every time 
costs money and takes time. I also wanted a place to break things 
without consequences.

So I turned my ASUS laptop into a Kubernetes cluster. The goal 
was simple: run real production-grade tooling — ArgoCD, Prometheus, 
Longhorn, Ingress — on hardware I already owned, and make it 
accessible from anywhere so I could actually use it day to day.

What surprised me was how far you can push a consumer laptop 
when you set it up properly.

## What I Built

![What I Built Overview](assets/what-i-built.png)
![Longhorn Storage Dashboard](assets/cluster-state.png)

## Hardware

Nothing fancy — just a laptop I already had sitting around.

| Component       | Details                                          |
|-----------------|--------------------------------------------------|
| Machine         | ASUS VivoBook X513UA                             |
| CPU             | AMD Ryzen 5 5500U — 6 cores, 12 threads, 4 GHz  |
| RAM             | 15.5 GB                                          |
| Storage         | 512 GB SSD                                       |
| Virtualization  | AMD-V — hardware-assisted KVM                    |
| OS              | Ubuntu 24.04 LTS — Kernel 6.17.0                 |
| Worker VMs      | 2× KVM VMs — 2 vCPU, 4 GB RAM, 20 GB each       |
| Workstation     | MacBook Air M4                                   |
| Mobile          | Samsung S23 Ultra (Android 16)                   |

The two worker nodes are KVM virtual machines running on the same 
laptop. Yes, it's a bit unusual. But it works, and it taught me 
more about virtualization than any course ever did.

## Tailscale — The Piece That Made Everything Possible

![Tailscale Web Client](assets/tailscale-web-client.png)

Before Tailscale, remote access to a home server meant one of these:
- Open ports on your router (bad idea)
- Set up a VPN server yourself (another thing to maintain)
- Pay for a cloud VM to use as a jump host (defeats the purpose)

Tailscale solved all of that in one command. Every device gets a 
permanent private IP. WireGuard handles the encryption. NAT traversal 
is automatic. I didn't touch my router at all.

The thing I found most satisfying — if I disconnect Tailscale on 
my phone, SSH drops within a second. Nothing is reachable from the 
public internet. The VPN is literally the only way in.

| Device             | Role                          |
|--------------------|-------------------------------|
| vijay-ubuntu       | Cluster host — always on      |
| MacBook Air M4     | Primary workstation           |
| Samsung S23 Ultra  | Mobile management via Termux  |

## The Cluster — Why RKE2

I picked RKE2 over kubeadm after reading about it for about 
ten minutes. kubeadm gives you a cluster. RKE2 gives you a 
CIS-hardened cluster with etcd, containerd, and Canal CNI 
all bundled in — and it's what Rancher uses in production.

For a homelab where I'm trying to learn real patterns, 
that matters more than simplicity.

The control plane runs on the host machine itself. 
The two worker VMs handle all application workloads — 
the control plane is tainted so nothing schedules on it 
by accident.

Currently running 69 pods across the two workers.

## Networking — MetalLB + Ingress + nip.io

Getting LoadBalancer services to work outside of a cloud 
provider is the first real challenge in bare-metal Kubernetes.
MetalLB solves this by responding to ARP requests on your network,
making Kubernetes think it has a cloud load balancer available.

I pointed MetalLB's IP pool at my Tailscale IP — which means 
every LoadBalancer service is automatically reachable from any 
device on my Tailscale network.

For DNS, I use nip.io — a free wildcard DNS service where 
anything.IP.nip.io resolves to that IP automatically. No domain 
purchase. No DNS record management. It just works.

So grafana.100.x.x.x.nip.io opens Grafana on any device, 
as long as Tailscale is connected.

## Storage — Longhorn

The first time I deployed Prometheus without persistent storage, 
I restarted a pod and lost two days of metrics. That was the moment 
I actually understood why distributed storage matters.

Longhorn replicates every volume across both worker nodes. 
If a worker goes down, the other has a full copy. When it 
comes back, Longhorn rebuilds the replica automatically.

Currently backing:
- Prometheus: 10 GB PVC
- Grafana: 2 GB PVC (dashboards survive restarts)

## Monitoring — Prometheus, Grafana, Headlamp

![Grafana Dashboard](assets/grafana.png)
![Headlamp Cluster Map](assets/headlamp.png)
![Headlamp Pods View](assets/headlamp-pods.png)

Installed via kube-prometheus-stack. One thing worth noting — 
I tried to deploy it through ArgoCD first. Hit a known bug 
where the Prometheus CRDs exceed ArgoCD's annotation size limit.
The fix was to install it directly with Helm and let ArgoCD 
manage everything else.

Current cluster state:
- 3 nodes, 69 pods, all healthy
- CPU: ~12% utilized across the cluster  
- RAM: ~67% utilized
- Metrics retained for 7 days on Longhorn-backed storage

## Managing the Cluster from My Phone

This one's a bit ridiculous but it actually works great.

Tailscale on Android + Termux gives me a full terminal on 
my Samsung S23 Ultra. I SSH into the cluster, run kubectl, 
check pod logs, restart deployments — all from my phone.

The browser dashboards (Grafana, Headlamp, ArgoCD) 
work fine on mobile too, especially Headlamp which 
is genuinely well designed for smaller screens.

I've done real troubleshooting sessions from my phone 
while away from my desk. Would not have expected that 
to be practical, but it is.

## Security

I ran Kubescape against MITRE ATT&CK and NSA frameworks.
Scores: MITRE 74%, NSA 69%. For a homelab, that's solid.

The failing checks are almost all expected — Longhorn 
needs privileged containers, MetalLB needs HostNetwork, 
RKE2 system pods use HostPath mounts. These aren't real 
risks in this setup, but they're worth knowing about.

The actual security model is simple: Tailscale is the 
perimeter. Nothing is open to the internet. No ports 
forwarded. Disconnect VPN — access is gone instantly.
Audit logging is enabled at the API server level.

## What's Next

- [ ] Add 2 Raspberry Pis to the cluster as new bare-metal worker nodes 
- [ ] Connect the Pis securely over the existing Tailscale network
- [ ] HTTPS with cert-manager + Let's Encrypt  
- [ ] Falco for runtime threat detection  
- [ ] GitHub Actions → ArgoCD end-to-end pipeline  
- [ ] Deploy a real full-stack application on this cluster  
- [ ] Velero backups to external storage  
- [ ] Expand worker disk sizes (20 GB fills up fast with Prometheus)
