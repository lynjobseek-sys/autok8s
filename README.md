# autok8s

Ansible playbooks for bringing up a small Kubernetes cluster on Ubuntu 22.04 hosts. Comes with Prometheus + Grafana for cluster metrics, and an optional Cosmos Hub full node as a sample workload.

Single control plane, kubeadm-based, Calico for networking. Not HA. Fine for labs and small dev clusters.

## What you get

- Kubernetes 1.28.2 (kubelet/kubeadm/kubectl pinned)
- Calico CNI v3.27.3 (bird backend by default)
- kube-prometheus stack (Prometheus, Grafana, Alertmanager, exporters)
- Optional Cosmos Hub `gaiad` StatefulSet running in the `cosmos` namespace

## Hosts

| Role | Count | Notes |
|---|---|---|
| Ansible runner | 1 | Reused as the control plane node |
| Control plane | 1 | 4 vCPU, 8 GB RAM |
| Worker | 2 | 8 vCPU, 16 GB RAM if you're running gaiad; smaller if not |

## Setup

```bash
sudo apt update && sudo apt install -y ansible
git clone https://github.com/lynjobseek-sys/autok8s.git
cd autok8s
```

Edit `inventories/group_vars/all.yml` to set your IPs:

```yaml
k8s_master_ip: "10.211.55.13"
k8s_node1_ip:  "10.211.55.15"
k8s_node2_ip:  "10.211.55.16"
```

Then make sure `ssh-copy-id` works from the runner to all three hosts. Run:

```bash
sudo ansible-playbook playbooks/setup_kubernetes.yml
```

## Verify

```bash
kubectl get nodes
# all 3 should be Ready
```

Grafana:

```bash
kubectl -n monitoring port-forward svc/grafana 3000:3000
# http://localhost:3000, set your own admin password on first login
```

Cosmos sync state:

```bash
kubectl -n cosmos exec -it deploy/gaia -- gaiad status | jq .SyncInfo
```

## Roles

```
roles/
  common/         hostname, /etc/hosts, sysctl, swapoff, firewalld off
  containerd/     containerd + systemd cgroup driver
  kube-deps/      kubelet/kubeadm/kubectl pinned versions
  kube-init/      kubeadm init on first master, kubeadm join on the rest
  network-calico/ apply calico manifests
  cluster-addon/  kube-prometheus stack
  workload-cosmos/ optional gaiad StatefulSet
```

If you don't want gaiad, comment out the `workload-cosmos` role import in `playbooks/setup_kubernetes.yml`.

## Config

Cluster-wide knobs live in `inventories/group_vars/all.yml`. The defaults are tuned for Parallels / VMware / bare-metal labs. Pod CIDR `10.244.0.0/16`, service CIDR `10.96.0.0/12`, NodePort range `30000-32767`. Override via `--extra-vars` if you don't want to edit the file.

## TODO

- HA control plane (3 stacked masters behind kube-vip)
- Separate etcd nodes
- Cilium / kube-OVN paths in addition to Calico
- Upgrade and node-removal playbooks
- MetalLB for bare-metal LB
- chrony for time sync

## License

MIT.
