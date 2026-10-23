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

## 3. Create your local config files

Your real IPs and username go in two git-ignored files, so they never end up in the repo:

```bash
cp inventory.example.ini inventory.ini
```

- **`inventory.ini`** — replace the 3 IPs with your nodes' real static IPs, and `YOUR_SSH_USER` with the username you SSH in as.
- **`group_vars/all/local.yml`** — create it with your values. It loads after `group_vars/all/defaults.yml`, so anything here overrides the defaults:

  ```yaml
  ---
  # Must match the IPs in inventory.ini (used to build /etc/hosts on every node)
  cluster_hosts:
    - { ip: "192.168.0.151", name: "node1" }
    - { ip: "192.168.0.152", name: "node2" }
    - { ip: "192.168.0.153", name: "node3" }

  # Subnet the nodes are on; k3s ports are only opened to this range
  lan_subnet: "192.168.0.0/24"
  ```

## 4. Test connectivity first

```bash
ansible k3s_cluster -m ping
```

All 3 should return `SUCCESS`. If not, fix SSH/inventory before continuing.

This must pass using your SSH **key**: the playbook turns off password logins, so if you're still getting in with a password you'll be locked out.

## 5. Run the playbook

```bash
ansible-playbook site.yml
```

This runs in order:
1. **All nodes** — apt update/upgrade, installs curl/git/htop/fail2ban/ufw etc., turns on automatic security updates, hardens SSH, disables swap, writes `/etc/hosts`, configures fail2ban, and enables ufw with default-deny, allowing SSH and the k3s ports from the LAN only.
2. **node1** — installs the k3s server (control-plane) at the pinned `k3s_version`, grabs the join token.
3. **node2 & node3** — installs the k3s agent at the same version, joins the cluster using node1's token automatically (no manual copy-paste needed).
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

## 7. Upgrading k3s

k3s is pinned by `k3s_version` in `group_vars/all/defaults.yml`, so every node (including any you add later) runs the same release. To upgrade, set a newer version from the [k3s releases](https://github.com/k3s-io/k3s/releases) (in `local.yml` or `defaults.yml`) and re-run the playbook. It upgrades node1 first, then the agents. Don't skip minor versions (e.g. go 1.36 → 1.37, not 1.36 → 1.38).

## 8. Re-running

The playbook is idempotent — re-running it won't reinstall k3s or redo completed steps, it'll just confirm everything's still in the desired state. Safe to run anytime you want to push a config change (e.g. editing `cluster_hosts` and re-running updates `/etc/hosts` on all nodes at once).

## Notes

- **Docker is intentionally excluded** — k3s bundles containerd, so a separate container runtime isn't needed for the cluster itself.
- **DNS**: this uses `/etc/hosts` for name resolution between the 3 nodes, which is the simplest approach at this scale. If you later add Pi-hole or another DNS server, you can retire this step.
- **ufw**: default-deny. Allowed: SSH from `ssh_allowed_cidr` (defaults to `lan_subnet`; widen it if you manage the nodes from another subnet, e.g. a VPN); 6443/tcp (API server), 8472/udp (Flannel VXLAN) and 10250/tcp (kubelet) from `lan_subnet`; and all traffic from the k3s pod (`10.42.0.0/16`) and service (`10.43.0.0/16`) networks. If you expose other services directly on a node, add more `ufw` allow rules in `site.yml`.
- **SSH**: `/etc/ssh/sshd_config.d/10-hardening.conf` allows key-only logins, blocks root, and only lets in the `ansible_user` from your inventory. If you need another account to SSH in, add it there.
- **Host key checking is on.** `ssh-copy-id` saves each node's host key the first time you connect. If you reinstall a node, Ansible will refuse to connect until you remove the old key: `ssh-keygen -R <node-ip>`, then SSH in once to accept the new one.
- **Security updates** install automatically every day from Debian's security archive. Nodes are never rebooted automatically; if `/var/run/reboot-required` exists, reboot nodes one at a time (agents first, node1 last).
