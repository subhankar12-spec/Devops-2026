# Kubernetes Hands-On Lab 22 — kubeadm Cluster Bootstrap

## 22.1 Objectives

By the end of this lab, you should be able to:

- Understand what `kubeadm` automates versus what you must prepare manually
- Prepare Nodes for cluster installation (container runtime, kernel modules, sysctl settings)
- Initialize a control-plane Node with `kubeadm init`
- Install a Pod network add-on (CNI)
- Join worker Nodes to the cluster with `kubeadm join`
- Generate a new join token and discovery hash when the original has expired
- Understand where kubeadm's generated certificates and manifests live on disk
- Troubleshoot a Node stuck `NotReady` after joining
- Perform common kubeadm tasks quickly for the CKA

## 22.2 Architecture

```
                    kubeadm init (on Node 1)
                            |
        Generates certs, static Pod manifests for
        kube-apiserver, kube-scheduler,
        kube-controller-manager, etcd
                            |
                Control Plane is UP
                            |
        prints a "kubeadm join" command with a
        bootstrap TOKEN + CA cert hash
                            |
        +-------------------+-------------------+
        |                                       |
        v                                       v
   kubeadm join (Node 2)                 kubeadm join (Node 3)
   -> kubelet starts, registers                same
      itself with the API server
      using the token for auth
                            |
        Nodes show NotReady until a
        CNI plugin is installed —
        this is EXPECTED, not a failure
```

Key concept: `kubeadm` deliberately does **not** install a Pod network (CNI) for you — that's a separate, explicit step. A freshly-`kubeadm init`'d cluster will show CoreDNS Pods stuck `Pending` and Nodes stuck `NotReady` until you apply a CNI manifest. This is one of the most common "my cluster is broken" false alarms for people learning kubeadm for the first time.

## 22.3 Prerequisites (All Nodes)

> ⚠️ This lab modifies system-level configuration and is best run on disposable VMs (cloud instances, or local VMs via Multipass/Vagrant) — not on a machine you rely on for other Kubernetes work. Requires at least 2 machines: one for the control plane, one (or more) for workers.

On **every** Node (control-plane and workers):

Disable swap (kubelet refuses to run with swap enabled by default):

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

Load required kernel modules:

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
```

Set required sysctl parameters:

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

sudo sysctl --system
```

Install a container runtime (containerd):

```bash
sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
```

## 22.4 Install kubeadm, kubelet, kubectl (All Nodes)

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg

echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

> Adjust the version path (`v1.34`) to whatever current stable minor version you're targeting — kubeadm requires the control-plane and kubelet versions to be within one minor version of each other.

## 22.5 Lab 1 — Initialize the Control Plane

On the control-plane Node only:

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --control-plane-endpoint=<control-plane-node-ip>
```

`--pod-network-cidr` must match whatever CNI plugin you plan to install (Calico's default is `192.168.0.0/16`). Watch the output — it ends with a `kubeadm join` command including a token and CA hash. **Save this output somewhere** — you'll need it for 22.7.

## 22.6 Configure kubectl Access

As the regular (non-root) user on the control-plane Node:

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Verify:

```bash
kubectl get nodes
```

The control-plane Node appears, but shows `STATUS: NotReady` — expected, since no CNI is installed yet.

```bash
kubectl get pods -n kube-system
```

`coredns` Pods show `Pending` for the same reason.

## 22.7 Lab 2 — Install a Pod Network Add-on (CNI)

Using Calico as an example:

```bash
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/calico.yaml
```

Watch the control-plane Node transition to `Ready`:

```bash
kubectl get nodes -w
```

Confirm CoreDNS is now `Running`:

```bash
kubectl get pods -n kube-system
```

## 22.8 Lab 3 — Join Worker Nodes

On each worker Node, run the exact command printed at the end of `kubeadm init` (22.5):

```bash
sudo kubeadm join <control-plane-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash>
```

Back on the control-plane Node, confirm the new Node registered:

```bash
kubectl get nodes -w
```

Each joining Node briefly shows `NotReady` while its kubelet starts and the CNI finishes configuring its network — this self-resolves within a minute or two on a healthy join.

## 22.9 Regenerate an Expired Join Token

Bootstrap tokens expire after 24 hours by default. If you need to join a Node later:

```bash
kubeadm token create --print-join-command
```

This prints a brand-new, ready-to-use `kubeadm join` command with a fresh token and hash — no need to remember or reconstruct the discovery hash manually.

List currently valid tokens:

```bash
kubeadm token list
```

## 22.10 Where kubeadm's Artifacts Live

```bash
sudo ls /etc/kubernetes/manifests/
```

These are the **static Pod manifests** for `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`, and (if stacked) `etcd` — the kubelet watches this directory directly and runs whatever it finds, without the scheduler ever being involved. Editing a file here and saving it causes the kubelet to restart that static Pod automatically.

```bash
sudo ls /etc/kubernetes/pki/
```

This holds the cluster's CA and all component certificates — the root of trust for the entire cluster.

```bash
cat /etc/kubernetes/admin.conf
```

The cluster-admin kubeconfig kubeadm generated for you in 22.6.

## 22.11 Break It — Node Stuck NotReady After Join

Simulate a common real cause: the kubelet's cgroup driver doesn't match containerd's.

```bash
sudo sed -i 's/SystemdCgroup = true/SystemdCgroup = false/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

On the control-plane Node, check the affected worker:

```bash
kubectl get nodes
```

Status stays `NotReady`.

## 22.12 Diagnose and Recover

On the affected worker Node, check kubelet's own logs (not `kubectl logs` — the kubelet itself isn't a Pod):

```bash
sudo journalctl -u kubelet -f
```

Look for errors mentioning cgroup driver mismatch between the kubelet and the container runtime.

From the control plane, cross-check the Node's conditions:

```bash
kubectl describe node <worker-node-name>
```

Look at the `Conditions:` section — `Ready: False` alongside a `KubeletNotReady` reason and message.

Fix by aligning the cgroup driver back:

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd
sudo systemctl restart kubelet
```

Confirm recovery:

```bash
kubectl get nodes -w
```

## 22.13 CKA Practice Task

**Task**

(Conceptual/command-recall — the actual exam gives you a partially-set-up multi-node environment.)

1. From memory, write the full sequence of prerequisite steps needed on a fresh Node before running `kubeadm init` (swap, kernel modules, sysctl, container runtime).
2. Write the `kubeadm init` command to bootstrap a control plane with pod network CIDR `10.244.0.0/16`.
3. Write the three commands needed to make `kubectl` usable as a regular user right after `kubeadm init`.
4. Write the command to generate a brand-new join command for a worker Node whose original token expired.
5. Identify which directory holds the static Pod manifests, and which holds the cluster's PKI.

**Target Time**

10 minutes (command recall, not live execution)

## 22.14 Troubleshooting Questions

After completing the lab, answer these without looking up the answers:

**Question 1**

Why do Nodes show `NotReady` and CoreDNS Pods show `Pending` immediately after `kubeadm init`, before any error has actually occurred?

**Question 2**

What is `--pod-network-cidr` for, and why must it match your chosen CNI plugin's expectations?

**Question 3**

Why are the control-plane components (`kube-apiserver`, `kube-scheduler`, etc.) run as static Pods rather than regular scheduled Pods?

**Question 4**

What are the two pieces of information a worker Node needs to securely join a cluster, and where do they come from?

**Question 5**

A join token has expired. What single command regenerates a fresh, ready-to-use join command?

**Question 6**

A newly-joined worker Node stays `NotReady` indefinitely. What are the first two places you'd check, and what's a common root cause?

## 22.15 Useful Commands

```bash
sudo kubeadm init --pod-network-cidr=<cidr> --control-plane-endpoint=<ip>
sudo kubeadm join <ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>

kubeadm token create --print-join-command
kubeadm token list

kubectl get nodes
kubectl describe node <name>
kubectl get pods -n kube-system

sudo ls /etc/kubernetes/manifests/
sudo ls /etc/kubernetes/pki/
sudo journalctl -u kubelet -f
sudo systemctl status kubelet
sudo systemctl restart kubelet
```

## 22.16 Cleanup

To fully tear down a kubeadm-created Node (control plane or worker):

```bash
sudo kubeadm reset -f
sudo rm -rf /etc/cni/net.d
sudo rm -rf $HOME/.kube
```

## 22.17 Lab Checklist

- [ ] Prepared Nodes with swap disabled, kernel modules, sysctl settings, and a container runtime
- [ ] Ran `kubeadm init` on the control-plane Node
- [ ] Configured `kubectl` access as a regular user
- [ ] Observed `NotReady` Nodes and `Pending` CoreDNS before installing a CNI
- [ ] Installed a CNI plugin and confirmed the cluster became `Ready`
- [ ] Joined at least one worker Node with `kubeadm join`
- [ ] Regenerated an expired join token with `kubeadm token create --print-join-command`
- [ ] Located static Pod manifests and PKI certificates on disk
- [ ] Reproduced a `NotReady` Node from a cgroup driver mismatch
- [ ] Diagnosed via `journalctl -u kubelet` and `describe node`, then fixed it
- [ ] Completed the CKA command-recall task within 10 minutes
