# 🛡️ WireGuard VPN Setup with Ansible

This project automates the installation of a WireGuard VPN server and the generation of client configurations using Ansible. It keeps the simple layout you have been using while adopting the working flow from [cwilliams001/ansible-wg](https://github.com/cwilliams001/ansible-wg).

---

## 🧠 What You Get

* A reproducible Ansible playbook (`playbook.yml`) that provisions WireGuard.
* An inventory file (`inventory.ini`) with a single `[wireguard]` group.
* A role (`roles/wireguard`) that installs packages, configures UFW, creates keys, and renders server/client configs.
* Idempotent tasks that only create keys and configs when they are missing, so you can re-run the playbook safely.

---

## 🖥️ Part 1 – Prepare Your Ansible Control Node

1. Remove any broken global Ansible packages:
   ```bash
   sudo apt remove --purge ansible -y
   sudo apt autoremove -y
   ```

2. Install Python tooling with `pipx`:
   ```bash
   sudo apt update
   sudo apt install -y pipx python3-venv
   pipx ensurepath
   source ~/.bashrc
   ```

3. Install Ansible the recommended way:
   ```bash
   pipx install ansible-core
   pipx inject ansible-core ansible
   ```

   Confirm everything works:
   ```bash
   ansible-playbook --version
   ```

   The role depends on the `community.general` collection for the UFW module. If you built a minimal Ansible environment, install it explicitly:
   ```bash
   ansible-galaxy collection install community.general
   ```

---

## 🗂️ Part 2 – Project Layout

Clone this repository and inspect the important files:

```
wireguard-ansible/
├── inventory.ini
├── playbook.yml
└── roles/
    └── wireguard/
        ├── defaults/main.yml
        ├── handlers/main.yml
        ├── tasks/
        │   ├── generate_clients.yml
        │   └── main.yml
        └── templates/
            ├── client.conf.j2
            └── wg0.conf.j2
```

* `defaults/main.yml` defines sensible defaults (10.10.10.0/24 network, UDP/51820, DNS, MTU, etc.).
* `tasks/main.yml` installs WireGuard, configures UFW NAT/forwarding, and prepares the server configuration.
* `tasks/generate_clients.yml` creates individual client keys/configs and ensures the server has matching `[Peer]` entries.
* Templates render ready-to-import configs for both server and clients.

---

## 🧾 Part 3 – Configure the Inventory

Edit `inventory.ini` and set the public IP/DNS of your WireGuard host. Example:

```
[wireguard]
wireguard ansible_host=203.0.113.10 ansible_user=root ansible_ssh_private_key_file=~/.ssh/id_rsa
```

If you SSH with a password, create and deploy a key first:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa
ssh-copy-id root@203.0.113.10
```

---

## ⚙️ Part 4 – Customize Variables (Optional)

Adjust defaults in `roles/wireguard/defaults/main.yml` or override them per host/group. Important options:

* `wireguard_network`: CIDR of the VPN network.
* `wireguard_server_address`: Address of the server inside the VPN network.
* `wireguard_port`: UDP port exposed on the server.
* `external_interface`: Public-facing network interface; auto-detected but override if your server has multiple NICs.
* `wg_interface`: WireGuard interface name (defaults to `wg0`).
* `wireguard_client_ip_base` & `wireguard_first_client_octet`: Control how client IPs are generated when you answer the prompt.
* `wireguard_endpoint`: Set this to a DNS name if it differs from the `ansible_host`.

---

## ▶️ Part 5 – Run the Playbook

From the project directory:

```bash
ansible-playbook -i inventory.ini playbook.yml
```

You will be asked:

```
How many clients do you want to create?
```

Enter any positive number. Each client gets:

* Keys stored on the server in `/etc/wireguard/clients`.
* A config file `/etc/wireguard/clients/clientX.conf`.
* A matching `[Peer]` section appended to `/etc/wireguard/wg0.conf`.

The role is idempotent: re-running with the same `client_count` preserves existing keys/configs while ensuring the server state is correct.

---

## 📦 Part 6 – Retrieve Client Configs

* Show a config directly on the server:
  ```bash
  ssh root@203.0.113.10
  cat /etc/wireguard/clients/client1.conf
  ```

* Copy it to your workstation:
  ```bash
  scp root@203.0.113.10:/etc/wireguard/clients/client1.conf ~/Desktop/
  ```

* Generate a QR code for mobile clients:
  ```bash
  qrencode -t ansiutf8 < /etc/wireguard/clients/client1.conf
  qrencode -t PNG -o /etc/wireguard/clients/client1.png < /etc/wireguard/clients/client1.conf
  ```

---

## 🔍 Useful WireGuard Commands

```bash
sudo systemctl status wg-quick@wg0
sudo systemctl restart wg-quick@wg0
sudo wg show
```

---

## ✅ Done!

You now have a tested WireGuard automation workflow. Re-run the playbook whenever you need to add more clients or tweak firewall settings—the role will keep your server configuration tidy and consistent.
