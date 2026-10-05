# Homelab bootstrap (Ansible)

Provisions 3x Debian 12 nodes (Dell OptiPlex 7070 Micro) with:
base updates, common tools, static `/etc/hosts` entries, fail2ban, ufw, and a k3s cluster (1 server + 2 agents).

## 1. Run this from your workstation, not the servers

Ansible is agentless — install it on your main PC/laptop, and it connects out to the 3 nodes over SSH.

```bash
# On your workstation (Debian/Ubuntu example)
sudo apt install -y ansible
ansible-galaxy collection install community.general
```

## 2. Set up SSH key access to all 3 nodes first

Ansible needs passwordless SSH (key-based) to work smoothly:

```bash
ssh-keygen -t ed25519          # skip if you already have a key
ssh-copy-id youruser@192.168.1.101
ssh-copy-id youruser@192.168.1.102
ssh-copy-id youruser@192.168.1.103
```

## 3. Edit these two files before running anything

- **`inventory.ini`** — replace the 3 IPs with your nodes' real static IPs, and `remote_user` in `ansible.cfg` with the actual username you SSH in as.
- **`group_vars/all.yml`** — `cluster_hosts` must list the same 3 IPs/names (used to build `/etc/hosts` on every node).

## 4. Test connectivity first

```bash
ansible k3s_cluster -m ping
```

All 3 should return `SUCCESS`. If not, fix SSH/inventory before continuing.

## 5. Run the playbook

```bash
ansible-playbook site.yml
```

This runs in order:
1. **All nodes** — apt update/upgrade, installs curl/git/htop/fail2ban/ufw etc., disables swap, writes `/etc/hosts`, configures fail2ban + ufw.
2. **node1** — installs k3s server (control-plane), grabs the join token.
3. **node2 & node3** — installs k3s agent, joins the cluster using node1's token automatically (no manual copy-paste needed).
4. **node1** — runs `kubectl get nodes` and prints the result so you can confirm all 3 joined.

## 6. Re-running

The playbook is idempotent — re-running it won't reinstall k3s or redo completed steps, it'll just confirm everything's still in the desired state. Safe to run anytime you want to push a config change (e.g. editing `cluster_hosts` and re-running updates `/etc/hosts` on all nodes at once).

## Notes

- **Docker is intentionally excluded** — k3s bundles containerd, so a separate container runtime isn't needed for the cluster itself.
- **DNS**: this uses `/etc/hosts` for name resolution between the 3 nodes, which is the simplest approach at this scale. If you later add Pi-hole or another DNS server, you can retire this step.
- **ufw**: the playbook enables ufw with a default-deny policy and only allows SSH. If you expose other services directly on a node (outside the cluster), you'll need to add more `ufw` allow rules — ask and I'll add them.
- After the cluster is up, `kubectl` from your workstation needs `/etc/rancher/k3s/k3s.yaml` copied from node1 (`~/.kube/config`) — say the word and I'll add a task/step for that too.
