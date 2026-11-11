# Code walkthrough — `k3s`

What each file does and why, briefly. Deeper reasons: `Dn` in [design-decisions.md](design-decisions.md).
**Written by:** Claude, at the user's request (2026-11-11); the user reviews.

| # | File | Status |
|---|------|--------|
| 1 | `meta/main.yml`, `meta/argument_specs.yml` | done |
| 2 | `.yamllint`, `.ansible-lint`, `.gitignore` | done, copied from the jenkins role |
| 3 | `defaults/main.yml`, `vars/main.yml` | done |
| 4 | `tasks/main.yml`, `config.yml`, `install.yml`, `firewall.yml`, `argocd.yml` | done, lint clean; run on 3 RHEL 9.8 VMs 2026-11-11 (see test-results) |
| 5 | `handlers/main.yml` | done |
| 6 | `templates/config.yaml.j2`, `registries.yaml.j2`, `argocd.yaml.j2` | done, rendered for server, agent and lone host, parsed as YAML |
| 7 | `requirements.yml`, `VERSION`, `README.md` | done |

---

## How it works in one picture

```text
             play 1: k3s_server                               play 2: k3s_agent (x2): read node-token from the server first
config.yaml (selinux) ─────────┐                    config.yaml (server URL, token, selinux) ──┐
registries.yaml ───────────────┤                    registries.yaml ───────────────────────────┤
install.sh → k3s server ◄──────┘                    install.sh → k3s agent ◄───────────────────┘
   │  :6443 API ◄──────────────────────── agents join with the token ─────────────┘
   │
   └─ manifests/argocd.yaml ──► k3s Helm controller ──► namespace argocd, Argo CD pods
                                                         UI: https://<server>:30443
pods pull 192.168.122.135:5000/... ──(registries.yaml: http)──► registry on the CI VM
```

---

## 1. `meta/`

| File | Does | Why |
|------|------|-----|
| `meta/main.yml` | `bunnywkwk.k3s`, EL 9, ansible-core 2.16, no dependencies | Same rules as the other roles |
| `meta/argument_specs.yml` | Types; `k3s_node_type` only `server` or `agent` | A typo like `k3s_node_type: worker` stops the play before anything changes |

## 2. `defaults/main.yml` and `vars/main.yml`

| File | Holds |
|------|-------|
| `defaults/main.yml` | Your choices: version, node type, token, server URL, registries, Argo CD, firewall |
| `vars/main.yml` | Fixed internals: binary and folder paths, service name per node type, install.sh URL, k3s's pod/service networks |

| Variable | Default | Why this default |
|----------|---------|------------------|
| `k3s_version` | `v1.37.2+k3s1` | The k3s **stable** channel on 2026-11-11 (`update.k3s.io/v1-release/channels`) |
| `k3s_node_type` | from the `k3s_agent` group | The inventory decides; a lone host is a server (D3) |
| `k3s_token` | `""` | The server generates one; agents read it from the server (D9) |
| `k3s_server_url` | first `k3s_server` host's `ansible_host`, port 6443 | Agents need the API address; no IP written in the role |
| `k3s_argocd_chart_version` | `10.17.1` | argo-helm chart released 2026-11-09 (Argo CD v3.6.1, Kubernetes ≥ 1.25); not the same-day 10.17.2 |
| `k3s_argocd_nodeport` | `30443` | The chart's own default `nodePortHttps` |

`k3s_bin` is the full path `/usr/local/bin/k3s`: on RHEL, sudo's `secure_path` does not include `/usr/local/bin`, so a bare `k3s` would not be found under `become`.

## 3. Tasks

### `tasks/main.yml`

Config, install, firewall, Argo CD (server only, when enabled). Config comes **first** because `install.sh` starts k3s, and that first start must already read `config.yaml`.

### `tasks/config.yml`

| Task | Does | Why |
|------|------|-----|
| `CONFIG \| Read the cluster token from the server` | Agents only, when `k3s_token` is empty: `slurp` of `/var/lib/rancher/k3s/server/node-token`, run on the server (`delegate_to`), `no_log` | The server generates this token on its first start; this is the copy you did by hand in the earlier lab (D9). If the server is not up yet, it fails with "file not found" |
| `CONFIG \| Use the server's token` | `k3s_token` = that file's content, trimmed | `set_fact` beats the role default, so `config.yaml` gets it |
| `CONFIG \| Create the k3s config folder` | `/etc/rancher/k3s` | Must exist before the templates; k3s only makes it when installed |
| `CONFIG \| Write config.yaml` | `0600`, `diff: false`; notifies restart | k3s reads it at every start (keys = command-line flags). Holds the token |
| `CONFIG \| Write registries.yaml` | One HTTP mirror per `k3s_insecure_registries` entry; notifies restart | containerd only reads it at start |

### `tasks/install.yml`

| Task | Does | Why |
|------|------|-----|
| `INSTALL \| Read the installed k3s version` | `k3s --version`, never fails, never "changed" | Decides whether to run the installer |
| `INSTALL \| Download the pinned k3s install script` | `install.sh` from the release tag to `/usr/local/lib/k3s-install-<version>.sh` | Same script as `curl get.k3s.io`, but pinned and kept for a look (D1) |
| `INSTALL \| Install k3s` | `sh install.sh` with `INSTALL_K3S_VERSION` and `INSTALL_K3S_EXEC=server\|agent`, only if the version differs | The script downloads the binary and checks its sha256, installs `container-selinux` + `k3s-selinux`, writes `k3s.service` or `k3s-agent.service` and starts it. Agents get `server` and `token` from `config.yaml`, not from environment variables |
| `INSTALL \| Start and enable k3s` | started, enabled | Brings the service back if someone stopped it |

### `tasks/firewall.yml`

| Task | Does | Why (k3s docs for RHEL) |
|------|------|-----|
| `FIREWALL \| Open the node ports` | `10250/tcp`, `8472/udp` on every node | kubelet (logs, exec, metrics); flannel VXLAN carries pod traffic between nodes |
| `FIREWALL \| Open the server ports` | `6443/tcp`, `k3s_argocd_nodeport/tcp` on the server | Kubernetes API (agents join, kubectl); the Argo CD UI |
| `FIREWALL \| Trust the pod and service networks` | `10.42.0.0/16`, `10.43.0.0/16` as sources in zone `trusted` | Pods talk to each other and to services; firewalld would otherwise drop it |

### `tasks/argocd.yml` (server only)

| Task | Does | Why |
|------|------|-----|
| `ARGOCD \| Create the k3s manifests folder` | `/var/lib/rancher/k3s/server/manifests` | k3s makes it on first start, but the template should never rely on a folder it did not make (jenkins T-J1) |
| `ARGOCD \| Write the Argo CD HelmChart manifest` | `argocd.yaml` there | k3s applies every file in this folder; its Helm controller then installs the chart (D5) |

## 4. `handlers/main.yml`

| Handler | Does | Why |
|---------|------|-----|
| `Restart k3s` | Restarts `k3s` or `k3s-agent` | Only when `config.yaml` or `registries.yaml` changed |

## 5. Templates

| Template | Becomes | Key lines |
|----------|---------|-----------|
| `config.yaml.j2` | `/etc/rancher/k3s/config.yaml` | Agent: `server`; both: `token` (if set), `node-name` = inventory name (minimal VMs often share `localhost.localdomain`; duplicate node names cannot join), `selinux: true` (k3s runs containerd with SELinux labels); server: `write-kubeconfig-mode: "0600"` (only root reads the admin kubeconfig). Values pass through `to_json`, so any character in the token is safe |
| `registries.yaml.j2` | `/etc/rancher/k3s/registries.yaml` | `mirrors: { "<host:port>": { endpoint: ["http://<host:port>"] } }`; `mirrors: {}` when the list is empty |
| `argocd.yaml.j2` | `/var/lib/rancher/k3s/server/manifests/argocd.yaml` | `HelmChart` (`helm.cattle.io/v1`) for `argo-cd` from `https://argoproj.github.io/argo-helm`, pinned version, namespace `argocd` (created), `server.service.type: NodePort`, `nodePortHttps` |
