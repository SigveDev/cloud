# VPS bootstrap: Ubuntu, SSH, firewall, and K3s

This documents the procedure used in the setup conversation. It is a rebuild guide, not a script to rerun against an already configured server. Actual public server/client addresses and credentials are omitted. Replace placeholders locally; never execute them literally.

## 1. Provider and first login

Select Hostinger's plain **Ubuntu Server 26.04 LTS** template. Reinstallation erases existing disk contents; protect any existing data before rebuilding. The observed installation reported Ubuntu 26.04.1 LTS, x86_64, 2 CPUs, about 8 GB RAM, and a 96 GB root filesystem.

Obtain the VPS public address and initial root login details from Hostinger. On Windows:

```powershell
ssh root@<VPS_PUBLIC_IP>
```

Use Hostinger Web Console to verify the SSH host fingerprint:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

An Ubuntu reinstall changed the server identity during this project. After independently verifying the new fingerprint, remove only the stale saved identity on Windows and reconnect:

```powershell
ssh-keygen -R <VPS_PUBLIC_IP>
ssh root@<VPS_PUBLIC_IP>
```

Do not disable SSH host-key verification to bypass a mismatch.

## 2. Inventory and packages

On the VPS:

```bash
cat /etc/os-release
nproc
free -h
df -h /
uname -m
apt update
apt upgrade
apt install sudo
```

The initial swap allocation was zero. The later Kubernetes preparation checked it with `swapon --show`.

## 3. Administrator and SSH key

On the VPS as root:

```bash
adduser sigve
usermod -aG sudo sigve
```

Choose the account password interactively; it is not recorded here. On Windows, generate a dedicated key with a passphrase, without overwriting an existing file:

```powershell
ssh-keygen -t ed25519 -f "$env:USERPROFILE\.ssh\id_ed25519_sigve_cloud" -C "sigve-cloud"
Get-Content "$env:USERPROFILE\.ssh\id_ed25519_sigve_cloud.pub" | ssh sigve@<VPS_PUBLIC_IP> 'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys'
ssh -o PasswordAuthentication=no -o KbdInteractiveAuthentication=no -i "$env:USERPROFILE\.ssh\id_ed25519_sigve_cloud" sigve@<VPS_PUBLIC_IP>
```

Then test on the server:

```bash
whoami
sudo whoami
```

Expected identities: `sigve`, then `root`. Keep the original working session open during access changes.

## 4. Windows SSH alias

Edit `$env:USERPROFILE\.ssh\config` on Windows. The exact filename is **config**, not `config.txt`. Preserve other existing entries.

```text
Host sigve-cloud
    HostName <VPS_PUBLIC_IP>
    User sigve
    IdentityFile ~/.ssh/id_ed25519_sigve_cloud
    IdentitiesOnly yes
```

If Notepad created `config.txt`, rename it only if no proper `config` file already exists. The setup conversation encountered and corrected this filename issue.

Test from a new PowerShell window:

```powershell
ssh sigve-cloud
```

## 5. SSH hardening

After successful key-only access and sudo testing, on the VPS:

```bash
sudo tee /etc/ssh/sshd_config.d/00-sd-cloud.conf >/dev/null <<'EOF'
PubkeyAuthentication yes
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin no
EOF
sudo /usr/sbin/sshd -t
```

Check effective settings before reloading:

```bash
sudo /usr/sbin/sshd -T | grep -E '^(permitrootlogin|passwordauthentication|kbdinteractiveauthentication|pubkeyauthentication) '
```

Expect public-key authentication enabled, with root, password, and keyboard-interactive login disabled. Conditional Match rules may require connection-specific verification if present. If validation succeeds:

```bash
sudo systemctl reload ssh
```

Test a second new `ssh sigve-cloud` connection before closing the original session. Disabling password SSH authentication does not remove the local password needed for sudo.

## 6. Provider firewall and reboot

Create and attach Hostinger rules permitting TCP 22 from `<CLIENT_PUBLIC_IP_CIDR>`, TCP 80/443 from the internet, and denying other inbound traffic. An IPv4 client-only source uses `/32`; an IPv6 client-only source uses `/128`. Update this rule when a dynamic client address changes. Do not put the actual client address in Git.

Keep Hostinger Web Console available for recovery. Test a new SSH session after applying the rules. Avoid opening the Kubernetes API or NodePort ranges generally to the internet.

The setup required a reboot after Ubuntu updates:

```bash
sudo reboot
```

After the server returns, reconnect from Windows with `ssh sigve-cloud`.

## 7. Prepare Kubernetes

Install base utilities if needed:

```bash
sudo apt install ca-certificates curl git jq unzip
```

With the provider firewall attached and verified, the chosen initial setup disables UFW to avoid unconfigured Pod/Service forwarding conflicts:

```bash
sudo ufw disable
swapon --show
```

The UFW command succeeded in the observed server output. The swap check should be empty for the selected initial setup. Do not assume disabling UFW configures any provider firewall.

K3s is designed for modern Linux. The generic documentation is not a certification of every K3s version on Ubuntu 26.04; inspect release-specific requirements and validate DNS/networking/storage after installation. Pod/Service CIDRs must not overlap future LAN/VPN routes.

## 8. Install pinned K3s

For a fresh server, create the configuration:

```bash
sudo install -d -m 0755 /etc/rancher/k3s
sudo tee /etc/rancher/k3s/config.yaml >/dev/null <<'EOF'
node-name: k3s01
write-kubeconfig-mode: "0600"
secrets-encryption: true
EOF
```

Download the official installer and execute only after a successful download:

```bash
curl -fL https://get.k3s.io -o /tmp/install-k3s.sh
sudo env INSTALL_K3S_VERSION='v1.36.5+k3s1' sh /tmp/install-k3s.sh
```

This pin was selected from the stable channel during setup. Review supported releases/security updates before a later rebuild. It installs K3s/containerd and kubectl as a service, retaining the default SQLite datastore and bundled components. Do not additionally install Docker Engine or host Nginx for this architecture.

## 9. Validate

```bash
sudo systemctl is-active k3s
sudo kubectl wait --for=condition=Ready node/k3s01 --timeout=180s
sudo kubectl get nodes -o wide
sudo kubectl get pods -A
sudo k3s secrets-encrypt status
```

Expect an active service, Ready node, healthy system Pods, and encryption enabled. Completed installation Jobs are normal. Investigate failures using:

```bash
sudo journalctl -u k3s -n 80 --no-pager
```

Kubeconfigs, server tokens, encryption keys, passwords, and real addresses remain outside the repository. Continue with [GUI setup](GUI-SETUP.md), then [application deployment](DEPLOY-HELLO-WORLD.md).
