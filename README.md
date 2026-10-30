# Homelab bootstrap (Ansible)

Turns 3 Debian 13 nodes (Dell OptiPlex 7070 Micro) into a k3s cluster: OS updates, SSH hardening, fail2ban, ufw, automatic security updates, and k3s (1 server + 2 agents) with embedded etcd and Secrets encrypted at rest.

## Requirements

**Workstation** (where Ansible runs; Linux, macOS, or WSL2 on Windows). Install Ansible with `brew install ansible`, or `sudo apt install -y ansible` on Debian/Ubuntu, then the collection:

```bash
ansible-galaxy collection install -r requirements.yml
```

**Each node**, set up by hand (easiest in the Debian installer: tick **SSH server** and **standard system utilities**, and leave the root password blank so your user gets sudo):

- `openssh-server` and `python3`
- A user with passwordless sudo (or run the playbook with `-K` to be prompted)
- A static IP, e.g. a DHCP reservation on your router

## Setup

1. Copy your SSH key to each node, once per node:

   ```bash
   ssh-copy-id youruser@<node-ip>
   ```

2. Create your inventory (git-ignored) and fill in the node IPs, `ansible_user` and `lan_subnet`. Every setting lives in this file.

   ```bash
   cp inventory.example.ini inventory.ini
   ```

3. Check that all nodes respond, using your SSH **key**. The playbook turns off password logins.

   ```bash
   ansible k3s_cluster -m ping
   ```

## Usage

Run everything:

```bash
ansible-playbook site.yml
```

Or one part, by its tag from the table below, for example:

```bash
ansible-playbook site.yml --tags security
```

Add `-K` if your user needs a password for sudo. Run the whole playbook once on new nodes before using tags. It is safe to re-run.

| Play | Tag | What it does |
|---|---|---|
| Preflight checks | always | Stops if `inventory.ini` has placeholders, or if this machine is outside the SSH allow range |
| Base system setup | `base` | One node at a time: drains it if it's in the cluster and updates are pending, upgrades, reboots if needed, uncordons. Installs packages, sets hostnames, disables swap, writes `/etc/hosts` |
| Security hardening | `security` | Unattended security upgrades, hardened SSH, kernel settings, unused protocols and USB storage blocked, cron/sshd permissions, fail2ban, ufw |
| Install k3s server | `k3s` | k3s server on node1 at `k3s_version`, with API audit logging, cluster-wide Pod Security defaults and `protect-kernel-defaults` |
| Install k3s agents | `k3s` | k3s agents on node2/node3, one at a time, drained on upgrade, with `protect-kernel-defaults` |
| Verify cluster | `k3s`, `verify` | Waits until every node is Ready, lists them, and saves the kubeconfig to `~/.kube/homelab.yaml` |
| Security audit | `audit` (on demand only) | Runs Lynis and debsecan on each node and saves the reports to `~/homelab-audit/<date>/` |

## Security audit

Not part of a normal run. Start it explicitly:

```bash
ansible-playbook site.yml --tags audit -K
```

On each node it installs and runs two scanners, then saves their reports to
`~/homelab-audit/<date>/` on your workstation and prints a summary per node:

- **Lynis** audits the host's hardening: SSH, firewall, kernel settings, file
  permissions, installed services. It gives a **hardening index** out of 100 and a
  list of warnings and suggestions, each with a test ID you can look up at
  cisofy.com/lynis/controls.
- **debsecan** lists installed packages with known vulnerabilities (CVEs) that
  already have a fix in Debian. Unattended upgrades should keep this list short or
  empty; if it isn't, run `--tags base` to upgrade.

Not every Lynis suggestion fits a k3s node. Some, like disabling IP forwarding,
would break Kubernetes networking. Treat the warnings as the to-do list and the
suggestions as ideas. Re-run after changes to see the score move.

The counts also go to `/var/lib/node_exporter/textfile_collector/audit.prom` on
each node, as Prometheus metrics (`lynis_hardening_index`, `lynis_warnings`,
`lynis_suggestions`, `debsecan_fixable_vulnerabilities`,
`homelab_audit_last_run_timestamp_seconds`). With the monitoring stack from
[k8s-homelab](https://github.com/abeciaj/k8s-homelab), node-exporter picks them
up and the Homelab Security dashboard shows them per node. The `base` play creates
that directory too, because node-exporter reports a scrape error if it's missing.

## kubectl from your workstation

The Verify play saves the cluster's kubeconfig to `~/.kube/homelab.yaml` (mode 600), pointed at node1. It goes in its own file, so an existing `~/.kube/config` is left alone. To fetch it again without re-running the rest: `ansible-playbook site.yml --tags kubeconfig`.

```bash
export KUBECONFIG=~/.kube/homelab.yaml
kubectl get nodes
```

The file holds a cluster-admin certificate. Never commit it.

## Upgrading k3s

Set a newer `k3s_version` in `inventory.ini` and run `ansible-playbook site.yml --tags k3s`. node1 is upgraded first (the API is briefly unavailable), then each agent is drained, upgraded and uncordoned in turn. Upgrade one minor version at a time (1.36 → 1.37, not 1.36 → 1.38).

## Notes

- **Firewall**: default deny. SSH is allowed from `lan_subnet` (or `ssh_allowed_cidr`). From `lan_subnet`, it also allows 6443/tcp (API), 8472/udp (Flannel), 10250/tcp (kubelet) and 80/443/tcp (Traefik). It allows all traffic from the pod and service networks, and 9100/tcp (node-exporter, for monitoring) from the cluster nodes only.
- **SSH**: key-only, no root login, and only `ansible_user` may log in. No TCP, agent or X11 forwarding, and idle sessions drop after 10 minutes.
- **Kernel**: `templates/sysctl_hardening.conf.j2` sets the six values the kubelet checks under `protect-kernel-defaults`, plus hardening that's safe on a Kubernetes node. The security play applies it before the k3s plays: with a missing value, the kubelet won't start. If that happens, `journalctl -u k3s` (or `k3s-agent`) names the value.
- **Pod Security**: namespaces without their own labels enforce `baseline` and warn about `restricted`; `kube-system` is exempt. Anything that needs host access, like node-exporter, needs a namespace labelled `privileged`. k8s-homelab's `monitoring/namespace.yaml` does this, so apply it before running `--tags k3s`.
- **API audit log**: `/var/lib/rancher/k3s/server/logs/audit.log` on node1. It records who did what, without request bodies, so no Secret values. It keeps 30 days, 10 files of 100MB at most.
- **Recovering from a bad k3s config**: if k3s won't start after a config change, comment out the new lines in `/etc/rancher/k3s/config.yaml` on that node, then `sudo systemctl restart k3s`. Check `journalctl -u k3s` for the cause.
- **Host keys**: host key checking is on. After reinstalling a node, run `ssh-keygen -R <node-ip>` and SSH in once to accept the new key.
- **Updates**: security updates install daily and never reboot. To apply ones that need a reboot, run `--tags base`.
- **Draining**: the other nodes need room for a drained node's pods. If a drain doesn't finish in 5 minutes, the run stops and leaves the node cordoned. To recover, run `sudo k3s kubectl uncordon <node>` on node1.
- **Backups**: none. Scheduled etcd snapshots are off until there's storage off node1 (see `templates/k3s_server_config.yaml.j2`). Take one manually with `sudo k3s etcd-snapshot save`.
- **Before the first install**, decide on `cluster-init` and `secrets-encryption` in `templates/k3s_server_config.yaml.j2`. Changing either later means a data migration or a key rotation.
- **Adding more servers** (HA) needs ports 2379–2380/tcp opened between them for etcd.

## Linting

CI runs `ansible-playbook --syntax-check` and `ansible-lint` (production profile) on every push. Locally:

```bash
pip install ansible-lint && ansible-lint
```
