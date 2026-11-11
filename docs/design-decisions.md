# Design decisions — `k3s`

Why the role is built this way. Format: **Decision** → **Why** → **Changed** (if it was).

---

## D1: The official install.sh, pinned to the release tag

**Decision:** download `https://raw.githubusercontent.com/k3s-io/k3s/<k3s_version>/install.sh` and run it with `INSTALL_K3S_VERSION` and `INSTALL_K3S_EXEC`, only when `k3s --version` does not show `k3s_version`.

**Why:**
- It is the k3s-supported install path. Read for the pinned tag (2026-11-11): it checks the binary's sha256 against the release (`Download sha256 does not match` otherwise), installs `container-selinux` and `k3s-selinux` from `rpm.rancher.io` on RHEL, writes the systemd unit, the env file and `k3s-uninstall.sh`.
- Writing the binary download, SELinux packages and units ourselves would copy that logic and drift from upstream.
- Pinned to a tag, not `get.k3s.io`, so every run uses the same script. A new `k3s_version` re-runs it: that is the upgrade.

**Not used:** the `k3s-io/k3s-ansible` collection. It is a full playbook set with its own inventory layout; the user's roles follow one layout (`mongodb-cis`).

## D2: Settings in `config.yaml`, not on the command line

**Decision:** server URL, token and flags go in `/etc/rancher/k3s/config.yaml`; install.sh gets only the command (`server` or `agent`).

**Why:** k3s reads this file at every start, and its keys are the command-line flags. A change is a template change plus a restart; the systemd unit stays as install.sh wrote it. install.sh only demands `K3S_URL`/`K3S_TOKEN` when it has to guess the command (`setup_env`); with `INSTALL_K3S_EXEC=agent` it does not.

## D3: Node type from inventory groups; a lone host is a server

**Decision:** `k3s_agent` group → agent, anything else → server.

**Why:** the inventory already says which VM is which; no per-host variable. A host in no group with no token becomes a 1-node cluster, which is exactly what `ansible-pipeline`'s VM test does (one VM, defaults only), so the role can pass CI in Batch 3 without CI-only variables.

## D4: Two plays, server first

**Decision:** the lab playbook (`cicd_lab_master/playbooks/main_playbook.yml`) runs the role on `k3s_server` (phase 2), then on `k3s_agent` (phase 3).

**Why:** agents need a running API on 6443 to join. One play over all hosts would install the agents while the server starts; they retry, but the first run would be slower and its errors harder to read.

## D5: Argo CD through k3s's Helm controller

**Decision:** a `HelmChart` manifest in `/var/lib/rancher/k3s/server/manifests/`; k3s installs the chart.

**Why:** k3s ships a Helm controller that applies every file in that folder. So no `helm` binary, no `kubectl` tasks, no `kubernetes.core` collection and no Python Kubernetes libraries on the nodes. The chart version is pinned in the manifest; changing it upgrades Argo CD.

**UI:** `NodePort` on `k3s_argocd_nodeport` (30443, the chart's default), HTTPS with Argo CD's self-signed cert. Login `admin`, password from `argocd-initial-admin-secret`. No Ingress yet: Traefik could serve it later.

**Limit:** removing the file or setting `k3s_argocd_enabled: false` does not uninstall Argo CD.

## D6: firewalld stays on

**Decision:** open the ports from the k3s RHEL docs instead of disabling firewalld (which the docs also allow).

**Why:** same as the other roles: the firewall stays on, with only what is needed open. Ports: `6443/tcp` (server), `10250/tcp`, `8472/udp` (all), the NodePort for Argo CD, and the pod/service networks trusted.

## D7: `selinux: true`

**Decision:** k3s runs with SELinux support; the host stays `Enforcing`.

**Why:** RHEL default. install.sh installs the `k3s-selinux` policy; `selinux: true` makes k3s's containerd label containers. Check V-8 looks for denials.

## D8: Plain-HTTP pulls from the lab registry

**Decision:** `registries.yaml` maps each `k3s_insecure_registries` entry to an `http://` endpoint.

**Why:** user decision (2026-11-11), so the cluster can run images Jenkins pushes. The registry has no TLS yet (registry D3).

**For the CI → CD link later:** an image is pulled by the name it was pushed under. On the cluster, `localhost:5000` would mean the node itself, so Jenkins must push as `192.168.122.135:5000/...`, which needs that entry in `jenkins_insecure_registries` too.

## D9: The server generates the token; agents read it

**Decision:** `k3s_token` is empty by default. The server then generates one on its first start (`/var/lib/rancher/k3s/server/node-token`); each agent reads that file from the server (`slurp` with `delegate_to`) and writes it into its `config.yaml`.

**Why:** user request, 2026-11-11: in the earlier manual lab no token was created; the generated one was copied to the workers. Automating that copy removes the vault step. The file is root-only on the server, and the value never shows in the output (`no_log`).

**Changed 2026-11-11:** the first version required `k3s_token` from ansible-vault for agents.

**Limits:** the agents need the server's play to have run first (two plays, D4). A rebuilt server generates a new token, so the agents must be reinstalled; to avoid that, set your own `k3s_token` (ansible-vault) from the start, and every node uses it.

