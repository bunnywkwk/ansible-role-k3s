# k3s

Ansible role that builds a **k3s** Kubernetes cluster on **RHEL 9**: one server (control plane) and any number of agents (workers), with **Argo CD** running on it. It prepares the CD environment: no apps are deployed.

Built for the lab: `k3s-master-1` + `k3s-worker-1/2` on the local KVM, pulling images from the CI host's registry. Later the same role targets AWS. It does not depend on the `registry` or `jenkins` roles: the registry address comes in through `k3s_insecure_registries`.

## What it does

```text
1. CONFIG    agents read the server's token → /etc/rancher/k3s/config.yaml (server URL, token, node name, SELinux) + registries.yaml
2. INSTALL   pinned official install.sh → k3s binary (sha256 checked), k3s-selinux, systemd unit → started, enabled
3. FIREWALL  10250/tcp, 8472/udp on all nodes; 6443/tcp + Argo CD NodePort on the server; pod/service networks trusted
4. ARGOCD    HelmChart manifest in k3s's manifests folder → k3s installs Argo CD itself (server only)
```

**Server or agent comes from the inventory:** hosts in the `k3s_agent` group join as workers; every other host is a server. One host on its own is a 1-node cluster.

A second run reports `changed=0`. k3s restarts only when `config.yaml` or `registries.yaml` changes. A new `k3s_version` upgrades the node.

## Requirements

- RHEL 9, registered (for `container-selinux`), with access to `github.com` (k3s), `rpm.rancher.io` (k3s-selinux) and `argoproj.github.io` (Argo CD chart).
- About 2 vCPU / 2 GB RAM per node; the server runs Argo CD as well.
- ansible-core 2.16 or later; collections: `ansible-galaxy collection install -r requirements.yml` (`ansible.posix`).
- firewalld running, unless `k3s_manage_firewall: false`.

## Variables

| Variable | Default | Purpose |
| --- | --- | --- |
| `k3s_version` | `v1.37.2+k3s1` | k3s release; changing it upgrades |
| `k3s_node_type` | `agent` in group `k3s_agent`, else `server` | Role of the host |
| `k3s_token` | `""` | Join secret. Empty: the server generates one and agents read it from the server. Set it (ansible-vault) only to choose your own |
| `k3s_server_url` | `https://<first k3s_server host>:6443` | Where agents join |
| `k3s_insecure_registries` | `[]` | Plain-HTTP registries containerd may pull from, e.g. `["192.168.122.135:5000"]` |
| `k3s_argocd_enabled` | `true` | Install Argo CD on the server |
| `k3s_argocd_chart_version` | `10.17.1` | argo-cd Helm chart (Argo CD v3.6.1) |
| `k3s_argocd_nodeport` | `30443` | Argo CD UI at `https://<server>:30443` |
| `k3s_manage_firewall` | `true` | Open the k3s ports in firewalld |

## Quick start

```yaml
# sysconfig/inventory.yml
k3s_cluster:
  children:
    k3s_server:
      hosts:
        k3s-master-1: { ansible_host: 192.168.122.50 }
    k3s_agent:
      hosts:
        k3s-worker-1: { ansible_host: 192.168.122.51 }
        k3s-worker-2: { ansible_host: 192.168.122.52 }
```

```yaml
# playbooks/site.yml: server first, so agents join a running server
- name: Set up the k3s server
  hosts: k3s_server
  become: true
  roles:
    - k3s

- name: Join the k3s agents
  hosts: k3s_agent
  become: true
  roles:
    - k3s
```

```bash
ansible-playbook playbooks/site.yml                        # rerun: changed=0
ssh <server> sudo k3s kubectl get nodes                    # 3 nodes Ready
```

Argo CD admin password (user `admin`):

```bash
ssh <server> "sudo k3s kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d"
```

The lab test project is `k3s-test/` next to this repo.

## Docs

| File | Read it for |
| --- | --- |
| [docs/code-walkthrough.md](docs/code-walkthrough.md) | What each file and task does, and why |
| [docs/design-decisions.md](docs/design-decisions.md) | Why install.sh, inventory groups, the generated token, HelmChart for Argo CD (`D1`–`D9`) |
| [docs/test-results.md](docs/test-results.md) | Checks that prove the cluster works: commands, expected and actual results |

## Known limits

- One server only (no HA control plane).
- Setting `k3s_argocd_enabled` back to `false` leaves Argo CD installed.
- Changing `k3s_token` on a running cluster breaks the join; set it once (or never: the generated one is the default).
- A rebuilt server generates a new token; the agents then need reinstalling (or set your own `k3s_token` from the start).
- Plain-HTTP registry pulls: lab network only.
