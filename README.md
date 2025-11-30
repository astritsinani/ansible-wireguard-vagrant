Here is the cleaned-up README with **Step 4 removed**, reorganized so everything still flows naturally.

You can copy/paste this as your new `README.md`:

---

```markdown
# WireGuard Lab on Apple M3 with UTM, Vagrant & Ansible

This project sets up a complete WireGuard lab on an **Apple Silicon (M1/M2/M3) Mac** using:

- **UTM** as the hypervisor (because VirtualBox is not supported on Apple Silicon)
- **Vagrant** for orchestration
- **Ansible** + an Ansible Galaxy WireGuard role to build a 3-node private VPN

Topology:

- **router** – WireGuard hub / VPN router  
- **client1** – WireGuard peer 1  
- **client2** – WireGuard peer 2  

WireGuard runs in a hub-and-spoke model: both clients connect to the router and communicate **through** it. They do not need to know each other's public IPs.

---

## Why UTM on an M3 Mac?

On Apple Silicon Macs:

- ❌ VirtualBox does **not** work  
- ❌ VMware Fusion lacks full Vagrant support  
- ✔️ **UTM** runs ARM Linux VMs at native speed  
- ✔️ **vagrant-utm** plugin allows Vagrant to control UTM VMs  

This project uses:

- UTM → provides the virtual machines  
- Vagrant → manages startup, provisioning, and SSH  
- Ansible → configures WireGuard on all nodes  

---

## Project Layout

```

.
├── Vagrantfile

├── README.md

└── ansible/

├── ansible.cfg

├── requirements.yml

├── inventory.ini

├── site.yml

├── generate_keys.yml

├── router.yml

└── peers.yml

````

- `Vagrantfile` – defines the 3 virtual machines  
- `ansible/` – all provisioning code  
- `requirements.yml` – pulls the WireGuard Galaxy role  
- `inventory.ini` – IP addresses and Ansible groups  
- `site.yml` – top-level playbook orchestrating everything  

---

## Requirements

Install required software via Homebrew:

```bash
brew install --cask utm
brew install vagrant
vagrant plugin install vagrant-utm
brew install ansible
````

---

## Step 1 – Create UTM VMs

You must create the 3 VMs yourself (UTM does not use Vagrant “boxes”):

Create three separate UTM VMs:

* `router`
* `client1`
* `client2`

Each must have:

* Ubuntu/Debian ARM64 installed
* user **vagrant** / password **vagrant**
* passwordless sudo
* OpenSSH server enabled

Inside each VM:

```bash
sudo apt update
sudo apt install -y openssh-server
echo "vagrant ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/vagrant
sudo systemctl enable ssh
sudo systemctl start ssh
```

Make sure the VM name **inside UTM** matches the name used in Vagrant (“router”, “client1”, “client2”).

---

## Step 2 – Configure Static IP Addresses (Required for UTM)

UTM **cannot** apply static IPs from Vagrant, so the IPs must be configured manually *inside* each VM using netplan.

Example for `router` (adjust interface to match `ip a` output):

Edit:

```bash
sudo nano /etc/netplan/99-vagrant.yaml
```

Add:

```yaml
network:
  version: 2
  ethernets:
    enp0s1:
      dhcp4: no
      addresses:
        - 192.168.56.10/24
      gateway4: 192.168.56.1
      nameservers:
        addresses: [1.1.1.1, 8.8.8.8]
```

Use:

* For `router`:  `192.168.56.10/24`
* For `client1`: `192.168.56.11/24`
* For `client2`: `192.168.56.12/24`

Apply:

```bash
sudo netplan apply
```

Verify from the Mac:

```bash
ssh vagrant@192.168.56.10
ssh vagrant@192.168.56.11
ssh vagrant@192.168.56.12
```

If SSH works → Vagrant + Ansible will work.

---

## Step 3 – Install Ansible Galaxy Role

Inside the project folder:

```bash
cd ansible
ansible-galaxy install -r requirements.yml
cd ..
```

This downloads the WireGuard role (`githubixx.ansible_role_wireguard`).

---

## Step 4 – Bring Up the Environment with Vagrant

From the project root:

```bash
vagrant up --provider=utm
```

What happens:

1. Vagrant starts all three UTM VMs
2. Vagrant connects to each VM using SSH
3. Vagrant runs Ansible:

   * Installs WireGuard
   * Generates keys
   * Sets up router as hub
   * Configures peers and routes

If Ansible reports an error, scroll up in the output — the real cause appears above the final “Ansible failed” message.

---

## Step 5 – Test WireGuard

SSH into the virtual machines:

```bash
vagrant ssh router
vagrant ssh client1
vagrant ssh client2
```

Check interfaces:

```bash
sudo wg show
ip a show wg0
```

Test connectivity through the WireGuard tunnel:

### From `client1`:

```bash
ping 10.10.0.3
```

### From `client2`:

```bash
ping 10.10.0.2
```

You should see replies — that confirms the VPN hub is working.

---

## Troubleshooting

### ❌ UNREACHABLE / SSH timeout

Example:

```
fatal: [router]: UNREACHABLE!
Failed to connect to host via ssh
```

Fix:

* Static IP not applied → recheck netplan
* SSH not running → `sudo systemctl start ssh`
* Wrong interface name in netplan → check with `ip a`

### ❌ ansible-playbook: unrecognized arguments: --sudo

You may have old Vagrant configs.

Fix:

* Use `ansible.become = true` in Vagrantfile
* Ensure playbooks use: `become: yes`
* Do NOT use `ansible.sudo = true`

---

## Summary

This project demonstrates how to build a functional WireGuard VPN lab on an Apple Silicon Mac, even though VirtualBox and VMware are not fully supported.

Using:

* **UTM** → Fast native ARM virtualization
* **Vagrant** → Control & provision VMs
* **Ansible** → Fully automate WireGuard configuration

Once the static IPs and SSH connectivity are set inside the UTM VMs, everything becomes repeatable and automated.
