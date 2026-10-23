# Homelab bootstrap (Ansible)

Provisions 3x Debian 13 nodes (Dell OptiPlex 7070 Micro) with:
base updates, common tools, static `/etc/hosts` entries, fail2ban, ufw, and a k3s cluster (1 server + 2 agents).

## 1. Run this from your workstation, not the servers

Ansible is agentless — install it on your main PC/laptop, and it connects out to the 3 nodes over SSH.

```bash
# On your workstation
sudo apt install -y ansible          # Debian/Ubuntu
brew install ansible                 # macOS
ansible-galaxy collection install community.general
```

Windows needs WSL2; Ansible doesn't run natively on Windows.

### What each node needs before Ansible can connect

Ansible can't install the things it uses to connect, so set these up by hand on all 3 nodes (easiest during the Debian installer: tick **SSH server** and **standard system utilities**, and leave the root password blank so your user gets sudo):

- `openssh-server` running
- `python3` (check with `python3 --version`)
- A user with **passwordless sudo**. `ansible.cfg` sets `become_ask_pass = False`, so either add `youruser ALL=(ALL) NOPASSWD:ALL` to a file in `/etc/sudoers.d/`, or run the playbook with `-K` to be prompted for the sudo password.
- A static IP, e.g. a DHCP reservation on your router. The playbook does not assign IPs.

## 2. Set up SSH key access to all 3 nodes first

Ansible needs passwordless SSH (key-based) to work smoothly:

```bash
ssh-keygen -t ed25519          # skip if you already have a key
ssh-copy-id youruser@192.168.1.101
ssh-copy-id youruser@192.168.1.102
ssh-copy-id youruser@192.168.1.103
```

## 3. Edit these three files before running anything

- **`inventory.ini`** — replace the 3 IPs with your nodes' real static IPs.
- **`ansible.cfg`** — replace `remote_user = YOUR_SSH_USER` with the actual username you SSH in as.
- **`group_vars/all.yml`** — `cluster_hosts` must list the same 3 IPs/names (used to build `/etc/hosts` on every node), and `lan_subnet` must be the subnet those IPs are on (k3s ports are only opened to this range).

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
1. **All nodes** — apt update/upgrade, installs curl/git/htop/fail2ban/ufw etc., disables swap, writes `/etc/hosts`, configures fail2ban, and enables ufw with default-deny, allowing SSH plus the k3s ports from `lan_subnet`.
2. **node1** — installs k3s server (control-plane), grabs the join token.
3. **node2 & node3** — installs k3s agent, joins the cluster using node1's token automatically (no manual copy-paste needed).
4. **node1** — waits until all 3 nodes report `Ready` (up to 5 minutes), then prints `kubectl get nodes` so you can confirm.

## 6. Use kubectl from your workstation (optional)

`/etc/rancher/k3s/k3s.yaml` on node1 is only readable by root, so fetch it with `sudo` rather than `scp`. This writes to its own file so it doesn't overwrite an existing `~/.kube/config`:

```bash
mkdir -p ~/.kube
ssh youruser@<node1-ip> sudo cat /etc/rancher/k3s/k3s.yaml > ~/.kube/homelab.yaml
sed -i.bak 's/127.0.0.1/<node1-ip>/' ~/.kube/homelab.yaml   # works on Linux and macOS
export KUBECONFIG=~/.kube/homelab.yaml
kubectl get nodes
```

## 7. Re-running

The playbook is idempotent — re-running it won't reinstall k3s or redo completed steps, it'll just confirm everything's still in the desired state. Safe to run anytime you want to push a config change (e.g. editing `cluster_hosts` and re-running updates `/etc/hosts` on all nodes at once).

## Notes

- **Docker is intentionally excluded** — k3s bundles containerd, so a separate container runtime isn't needed for the cluster itself.
- **DNS**: this uses `/etc/hosts` for name resolution between the 3 nodes, which is the simplest approach at this scale. If you later add Pi-hole or another DNS server, you can retire this step.
- **ufw**: default-deny. Allowed: SSH from anywhere; 6443/tcp (API server), 8472/udp (Flannel VXLAN) and 10250/tcp (kubelet) from `lan_subnet`; and all traffic from the k3s pod (`10.42.0.0/16`) and service (`10.43.0.0/16`) networks. If you expose other services directly on a node, add more `ufw` allow rules in `site.yml`.
