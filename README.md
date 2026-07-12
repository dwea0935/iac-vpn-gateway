# Idempotent VPN Gateway with automated deployment via Ansible playbooks

![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![WireGuard](https://img.shields.io/badge/wireguard-%2388171A.svg?style=for-the-badge&logo=wireguard&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

This repository contains an idempotent, Infrastructure-as-Code (IaC) deployment for a secure WireGuard VPN gateway, orchestrated via Ansible.

The architecture is designed with a "Zero-Trust" mindset: it enforces a strict default-deny perimeter, dynamically routes DNS through an encrypted proxy (dnscrypt-proxy), and programmatically locks down all public administrative access once the secure tunnel is established.

Recent architectural upgrades have transitioned this from a static configuration to a highly modular deployment. By utilizing global variables and dynamic interface detection, the infrastructure seamlessly adapts to virtualized host environments (like Proxmox) without routing collisions between the physical LAN and the virtual tunnel.

## Requirements

This architecture is designed for Debian-based distributions. (Tested exclusively on Ubuntu 26.04 LTS).

* **Supported OS:** Theoretical support exists for Ubuntu 20.04+ and Debian 11+, but these have not been actively validated yet. *(Note: RHEL/CentOS/Fedora are not supported out-of-the-box due to hardcoded `apt` and `ufw` module dependencies).*
* **Network Stack:** Fully supports IPv4 and IPv6 dual-stack routing. The WireGuard interface is configured with an IPv6 Unique Local Address (ULA) subnet (`fd42:42:42::/64`) to prevent client-side IPv6 traffic leaks.
* **Init System:** Requires `systemd`. 
* **Cloud Server Ports:** If used in a VPS environment, it needs to have UDP port 51820 and TCP port 22 open in the Cloud Firewall. Navigate to your VPS dashboard settings "Security Groups" or "VPC Network Rules", or similar to add these.
* **Virtualization:** Requires KVM or dedicated hardware (OpenVZ is not supported).

> [!NOTE]
> **DNS Port 53 Handling:** Modern Ubuntu releases utilize a `systemd-resolved` stub listener that binds to port 53. The deployment playbooks automatically disable this stub listener to prevent collision with `dnsmasq`, and safely re-link `/etc/resolv.conf` to maintain host-level DNS resolution.

> [!NOTE]
> **Service Boot Sequencing:** A systemd drop-in override is automatically applied to `dnsmasq` to enforce a strict boot-order dependency, ensuring it never attempts to forward traffic before `dnscrypt-proxy` is fully initialized.

## Architecture Overview

* Orchestration - **Ansible** (Driven by a single source of truth in `group_vars/all.yml`)
* Core Tunnel - **WireGuard** (`wg0` with a maintained server-side client ledger)
* Local DNS Caching - **dnsmasq**
* Encrypted DNS Routing (DoH) - **dnscrypt-proxy**
* Firewall State - **UFW** (With dynamically ingected `*nat` MASQUERADE rules)
* Defense in Depth layer: **fail2ban**

## Security Posture

* Cryptographic Identity
  * Automated generation and deployment of Ed25519 SSH keys for a dedicated, passwordless automation user.
* Privilege Drop
  * Configuration files are permission-scoped (e.g., `0644` vs `0600`) to respect Linux service privilege-dropping mechanics.
* The Vault Door
  * The deployment sequence permanently removes public SSH access on port 22, restricting all administrative traffic to the internal  **WireGuard** subnet of your choice in `group_vars/all.yml' config file.
* Supply Chain Verification
  * `dnscrypt-proxy` public resolver lists are cryptographically verified via Minisign keys.
* Garbage Collection
  * Automated detection and removal of orphaned/ghost peers within the **WireGuard** interface.
* Defense in Depth:
  * While the Zero-Trust perimeter technically renders `fail2ban` redundant (WireGuard silently drops unauthenticated packets, and public SSH is firewalled), it is deployed as a fail-safe. If the UFW state is ever accidentally compromised or misconfigured in the future, `fail2ban` is pre-configured to utilize `ufw` ban actions to immediately halt lateral brute-force attempts on the SSH daemon.

## Deployment Phases

Because this architecture permanently disables public administrative access, deployment is split into two distinct operational phases to prevent firewall lockouts.

### Phase 1: The Bootstrap

This phase targets the raw metal via its public IP to establish secure automation credentials.

Add the target server's public IP to `inventory_initial.ini`.

Execute the bootstrap playbook:

``` ansible-playbook -i inventory_initial.ini bootstrap_ansible_acc.yml -k -K```

_Note: This generates a local SSH keypair, provisions the ansible user, pushes the public key, and dynamically creates the inventory_internal.ini file for Phase 2._


### Phase 2: Core Infrastructure & Lockdown

This phase builds the encrypted tunnel, configures DNS routing, and permanently locks down the public perimeter.

Ensure your local machine has generated the `inventory_internal.ini` from Phase 1.

Execute the master site orchestration:

```ansible-playbook -i inventory_internal.ini execute.yml```


## Peer Provisioning

To add a new device to the VPN:

```ansible-playbook -i inventory_internal.ini add_peer.yml -e "peer_name=laptop peer_ip=10.0.0.2 device_type=desktop"```


_Note: `peer_name, peer_ip, device_type` are the adjustable variables. `device_type` can be either `desktop` or `mobile`. The IP should align with the `vpn_subnet_prefix` defined in your global variables._


Client configuration files and mobile QR codes will be safely generated locally in `/tmp/vpn_clients/`.


To remove an existing device from VPN:

```ansible-playbook -i inventory_internal.ini remove_peer.yml -e "peer_name=laptop"```

_Note: `peer_name` is the same variable you used before to add that peer. It can also be found on the server in the `/etc/wireguard/client_ledger` folder._

