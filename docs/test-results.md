# Test results — `k3s`

Checks that prove the role works, with the exact command and the expected result. **Result** says what actually happened; `not run` means no evidence yet.
Test project: `/home/bunny/cicd-automation/k3s-test` (inventory: `k3s-master-1`, `k3s-worker-1`, `k3s-worker-2`, user `frqadmin`; IPs in `sysconfig/inventory.yml`).

## Static checks (workstation, no VM)

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| S-1 | YAML lint | `yamllint .` | no output | ✅ 2026-11-11, yamllint 1.38.0 |
| S-2 | Ansible lint | `ansible-lint --profile production` | `Passed: 0 failure(s)` | ✅ 2026-11-11, ansible-lint 26.11.0 on ansible-core 2.16.20 (`/opt/ci`) |
| S-3 | Templates render | render all three templates for a server, an agent and a lone host (token with `"`, `:`, `#`; one registry); parse as YAML | server: token, no `server`; agent: `server: https://<master>:6443`; lone host: server, no token; mirror `http://...`; empty list → `mirrors: {}` | ✅ 2026-11-11, ansible-core 2.20.7 |
| S-4 | Playbook syntax | `ansible-playbook --syntax-check playbooks/site.yml` | `playbook: playbooks/site.yml` | ✅ 2026-11-11 |
| S-5 | Token copy | role `config.yml` in check mode on a fake server + agent (`local`), token file and config dir pointed at scratch paths | agent: `ok: [k3s-worker-1 -> k3s-master-1]`, token = file content trimmed; server: both token tasks skipped, token empty | ✅ 2026-11-11 |

S-3 proves the files are valid YAML with the right values, **not** that k3s or the Helm controller accept them; V-3 to V-5 do that.

## Runtime checks (3 RHEL 9 VMs)

Run from `k3s-test/` unless the command says `(master)` or `(worker)`: those run on that VM. On the master, `kubectl` means `sudo /usr/local/bin/k3s kubectl`. (`/usr/local/bin` is not in sudo's `secure_path` on RHEL, so `sudo k3s` alone is "command not found"; without sudo, the root-only `k3s.yaml` gives "permission denied", by design: `write-kubeconfig-mode: "0600"`).

| # | Check | Command | Expected | Result |
|---|-------|---------|----------|--------|
| V-1 | First run | `ansible-playbook playbooks/site.yml` | `failed=0` on all 3 hosts; on the workers `CONFIG \| Read the cluster token from the server` shows `ok: [k3s-worker-N -> k3s-master-1]` | ✅ 2026-11-11: master `ok=15 changed=10 failed=0`, workers `ok=14 changed=8 failed=0`; both workers read the token from the master (`192.168.122.165`); install, firewall, Argo CD manifest, one restart each. Log `k3s-test/runs/first-run.log` |
| V-2 | Idempotent | same, rerun | `changed=0` on all 3 | ✅ 2026-11-11: `changed=0 failed=0` on all 3 (master ok=13 skipped=3, workers ok=12 skipped=4: install skipped, version matched). Log `k3s-test/runs/second-run.log` |
| V-3 | Cluster | (master) `kubectl get nodes -o wide` | 3 nodes `Ready`; `k3s-master-1` has role `control-plane`; version `v1.37.2+k3s1` | ✅ 2026-11-11: 3 `Ready`, `k3s-master-1` `control-plane`, all `v1.37.2+k3s1`, containerd 2.3.7-k3s1, RHEL 9.8; node names from the inventory |
| V-4 | System and Argo CD pods | (master) `kubectl get pods -A` | `kube-system`: coredns, traefik, metrics-server, local-path-provisioner `Running`, `helm-install-argocd-*` `Completed`; `argocd`: all `Running` | ✅ 2026-11-11: all 7 `argocd` pods `Running`; coredns, traefik, metrics-server, local-path-provisioner `Running`; `helm-install-argocd` `Completed`; `svclb-traefik` on 3 nodes. `helm-install-traefik` restarted twice while it waited for its CRDs, then completed |
| V-5 | Argo CD UI | browser `https://<master-ip>:30443`; password: (master) `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' \| base64 -d` | login as `admin` works (accept the self-signed cert) | ✅ 2026-11-11, user-reported: UI reachable, login works |
| V-6 | Lab registry pull | (worker) `sudo /usr/local/bin/k3s crictl pull 192.168.122.135:5000/test/jenkins-smoke:1` | `Image is up to date for sha256:...` (the image jenkins V-12 pushed) | ✅ 2026-11-11 attempt 2 (master): `crictl pull 192.168.122.135:5000/test/busybox:1` → `Image is up to date for sha256:b116e155…`. Test image pushed by hand on the CI VM as `localhost:5000/test/busybox:1` (same storage, pulled by IP). Attempt 1 failed with `no route to host` (cause, user-confirmed: the CI VM was not started; not a role problem) |
| V-7 | Pods on workers | (master) `kubectl create deployment web --image=docker.io/library/nginx:1.27 --replicas=3`; `kubectl get pods -o wide`; then `kubectl delete deployment web` | 3 pods `Running`, spread over the nodes | ✅ 2026-11-11 scheduling: 3 `web` pods `Running`, one each on master, worker-1, worker-2 (pod IPs 10.42.0/1/2.x). Cross-node: busybox pinned to worker-1 fetched the nginx page from the pods on worker-2 and the master (flannel VXLAN `8472/udp` + trusted pod network work); `web` deleted. The `couldn't attach to pod` warning is harmless: the pod finished before `-it` attached |
| V-8 | SELinux | (each) `getenforce`; `sudo ausearch -m AVC -ts boot` | `Enforcing`; no k3s/container denials | ✅ 2026-11-11, user-reported: `Enforcing` on all 3, no AVC denials since boot |
| V-9 | Firewall | (master) `sudo firewall-cmd --list-ports`; (each) `sudo firewall-cmd --zone=trusted --list-sources` | master: `6443/tcp 10250/tcp 8472/udp 30443/tcp`; workers: `10250/tcp 8472/udp`; all: `10.42.0.0/16 10.43.0.0/16` | ✅ 2026-11-11, user-reported: ports and trusted sources as expected on all 3; Argo CD UI reachable from the laptop |
| V-10 | Reboot | reboot all 3 VMs; V-3 and V-5 again | all `Ready`; Argo CD back | ✅ 2026-11-11, user-reported: after rebooting all 3, nodes `Ready` and Argo CD back; playbook rerun idempotent (`changed=0`) |

V-6 needs the CI VM (registry) up, and the build number of an image that exists there: list them with `curl -s http://192.168.122.135:5000/v2/test/jenkins-smoke/tags/list`.

## Evidence

Save each run's output under `k3s-test/runs/` and fill in the Result column with the date and the key numbers.
