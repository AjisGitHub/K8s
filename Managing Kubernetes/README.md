Managing Kubernetes Clusters with kubeadm
This README provides an overview of the steps and considerations for managing Kubernetes clusters using kubeadm, with a focus on high availability (HA) setups as detailed in the guide from Hackerstack.
Overview
The guide explains how to set up and manage Kubernetes clusters on Linux machines using kubeadm. It covers Kubernetes architecture, cluster initialization, upgrades, and high availability configurations to ensure minimal downtime and robust cluster management. The focus is on practical steps for deploying and maintaining clusters, including version control and upgrades without service disruption.
Prerequisites

Operating System: Debian or Ubuntu-based Linux distributions (e.g., Ubuntu 22.04 or later).
Hardware:
Three or more machines for control plane nodes (odd number recommended for leader election).
Three or more machines for worker nodes.
Minimum 2GB RAM and 2vCPU per machine.


Software:
kubeadm, kubelet, and kubectl installed on all nodes.
Container runtime (e.g., CRI-O or containerd) configured with the same cgroups driver as kubelet (systemd recommended).


Network:
Full network connectivity between all nodes (public or private network).
Access to the Kubernetes container image registry (registry.k8s.io) or preloaded images.


Permissions: Root access or sudo privileges on all nodes.
Swap: Disabled on all nodes to ensure kubelet functionality.

Installation

Set Up Container Runtime:

Install and configure a container runtime like CRI-O or containerd.
Ensure the cgroups driver is set to systemd to match kubelet (default in Kubernetes v1.22+).
Example for CRI-O configuration:crio config | less

Add custom configurations in /etc/crio/crio.conf.d if needed.


Install Kubernetes Components:

Install kubeadm, kubelet, and kubectl on all nodes. Specify the desired Kubernetes version (e.g., v1.29.9).
Example commands for Debian/Ubuntu:KUBERNETES_REPO_VERSION=v1.29
curl -fsSL https://pkgs.k8s.io/core:/stable:/$KUBERNETES_REPO_VERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$KUBERNETES_REPO_VERSION/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list
KUBERNETES_INSTALL_VERSION=1.29.9
sudo apt update && sudo apt install -y kubelet=$KUBERNETES_INSTALL_VERSION* kubeadm=$KUBERNETES_INSTALL_VERSION* kubectl=$KUBERNETES_INSTALL_VERSION*


Prevent automatic upgrades:sudo apt-mark hold kubeadm kubelet kubectl




Initialize the Cluster:

On the first control plane node, initialize the cluster with kubeadm init. Specify the control plane endpoint for HA setups (e.g., a load balancer IP or DNS).
Example:sudo kubeadm init --control-plane-endpoint "vip-k8s-master:8443" --upload-certs


Copy the kubeadm join commands from the output for joining additional control plane and worker nodes.


Join Additional Nodes:

For control plane nodes:sudo kubeadm join <control-plane-endpoint>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash> --control-plane --certificate-key <key>


For worker nodes:sudo kubeadm join <control-plane-endpoint>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>





High Availability Notes
To ensure high availability, configure the cluster with multiple control plane nodes and a load balancer to distribute traffic to kube-apiserver instances. Two HA topologies are supported:

Stacked Control Plane Nodes:

Etcd members and control plane components (e.g., kube-apiserver, kube-scheduler, kube-controller-manager) are co-located on the same nodes.
Requires fewer machines but risks losing both an etcd member and control plane instance if a node fails.
Default topology in kubeadm.


External etcd Cluster:

Etcd runs on separate nodes from the control plane, requiring more infrastructure (at least three control plane nodes and three etcd nodes).
Decouples etcd and control plane, reducing the impact of node failures on cluster redundancy.



Load Balancer Configuration:

Use a TCP load balancer (e.g., HAProxy or cloud-based load balancer) to forward traffic to all healthy control plane nodes on port 6443.
Example HAProxy configuration:frontend kubernetes
    bind <load-balancer-ip>:6443
    option tcplog
    mode tcp
    default_backend kubernetes-master-nodes
backend kubernetes-master-nodes
    mode tcp
    balance roundrobin
    option tcp-check
    server k8s-master-0 <master-ip-0>:6443 check
    server k8s-master-1 <master-ip-1>:6443 check
    server k8s-master-2 <master-ip-2>:6443 check


Ensure the load balancer is highly available (e.g., using Keepalived for a virtual IP).

Recommendations:

Use an odd number of control plane nodes (e.g., 3) to facilitate leader election in case of failures.
Regularly back up the etcd database18 database to prevent data loss.
Deploy a network plugin (e.g., Fl弟弟, Calico) to enable pod communication. Install only one pod network per cluster:kubectl apply -f <network-plugin-config>.yaml



Upgrading the Cluster
To upgrade the cluster to a newer version (e.g., v1.30.5) without service disruption:

Plan the Upgrade:
sudo kubeadm upgrade plan v1.30.5


Upgrade the Control Plane:

On the first control plane node:sudo kubeadm upgrade apply v1.30.5


Drain the node before upgrading:kubectl drain <node-name> --ignore-daemonsets




Upgrade Additional Control Plane Nodes:

Repeat the drain and upgrade process for each additional control plane node.


Upgrade Worker Nodes:

Drain each worker node:kubectl drain <node-name> --ignore-daemonsets


Update kubelet and kubectl:sudo apt-mark unhold kubelet kubectl
sudo apt-get update && sudo apt-get install -y kubelet='1.30.5*' kubectl='1.30.5*'
sudo apt-mark hold kubelet kubectl


Uncordon the node after upgrade:kubectl uncordon <node-name>




Verify the Upgrade:
kubectl get nodes



Note: Do not skip minor versions during upgrades (e.g., upgrade from 1.29.x to 1.30.x, not directly to 1.31.x). Refer to the Kubernetes Version Skew Policy for details.
Troubleshooting

Cluster Access Issues: Verify load balancer connectivity and ensure kube-apiserver is running on port 6443.
Image Pull Failures: Ensure nodes can access registry.k8s.io or pre-load required container images.
Node Not Ready: Check kubelet status and logs:systemctl status kubelet
journalctl -u kubelet


Certificate Issues: If certificates expire, renew them manually or use kubeadm init phase upload-certs to reload certificates.
Report issues to the kubeadm issue tracker.

Additional Resources

Kubernetes Official Documentation
Hackerstack Guide
Creating Highly Available Clusters with kubeadm
Kubernetes Releases

Contributing
For feedback or contributions to this guide, visit the Hackerstack website or open an issue on the Kubernetes kubeadm repository.
