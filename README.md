# Idempotent VPN Gateway with automated deployment via Ansible playbooks

![Ansible](https://img.shields.io/badge/ansible-%231A1918.svg?style=for-the-badge&logo=ansible&logoColor=white)
![WireGuard](https://img.shields.io/badge/wireguard-%2388171A.svg?style=for-the-badge&logo=wireguard&logoColor=white)
![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)

This repository contains an idempotent, Infrastructure-as-Code (IaC) deployment for a secure WireGuard VPN gateway, orchestrated via Ansible.

The architecture is designed with a "Zero-Trust" mindset: it enforces a strict default-deny perimeter, dynamically routes DNS through an encrypted proxy (dnscrypt-proxy), and programmatically locks down all public administrative access once the secure tunnel is established.

## Architecture Overview

* Orchestration - **Ansible**
* Core Tunnel - **WireGuard** (`wg0`)
* Local DNS Caching - `dnsmasq`
* Encrypted DNS Routing (DoH) - `dnscrypt-proxy`
* Firewall State - **UFW**

## Security Posture

* Cryptographic Identity
  * Automated generation and deployment of Ed25519 SSH keys for a dedicated, passwordless automation user.
* Privilege Drop
  * Configuration files are permission-scoped (e.g., `0644` vs `0600`) to respect Linux service privilege-dropping mechanics.
* The Vault Door
  * The deployment sequence permanently removes public SSH access on port 22, restricting all administrative traffic to the internal `10.0.0.0/24` **WireGuard** subnet.
* Supply Chain Verification
  * `dnscrypt-proxy` public resolver lists are cryptographically verified via Minisign keys.
* Garbage Collection
  * Automated detection and removal of orphaned/ghost peers within the **WireGuard** interface.

## Deployment Phases

Because this architecture permanently disables public administrative access, deployment is split into two distinct operational phases to prevent firewall lockouts.

### Phase 1: The Bootstrap

This phase targets the raw metal via its public IP to establish secure automation credentials.

Add the target server's public IP to `inventory_initial.ini`.

Execute the bootstrap playbook:

``` ansible-playbook -i inventory_initial.ini bootstrap_ansible_acc.yml```

_Note: This generates a local SSH keypair, provisions the ansible user, pushes the public key, and dynamically creates the inventory_internal.ini file for Phase 2._


### Phase 2: Core Infrastructure & Lockdown

This phase builds the encrypted tunnel, configures DNS routing, and permanently locks down the public perimeter.

Ensure your local machine has generated the `inventory_internal.ini` from Phase 1.

Execute the master site orchestration:

```ansible-playbook -i inventory_internal.ini execute.yml```


## Peer Provisioning

To add a new device to the VPN:

```ansible-playbook -i inventory_internal.ini add_peer.yml -e "peer_name=laptop peer_ip=10.0.0.2 device_type=desktop"```

_Note: `peer_name, peer_ip, device_type` are the adjustable variables. `device_type` can be either `desktop` or `mobile`._


Client configuration files and mobile QR codes will be safely generated locally in `/tmp/vpn_clients/`.
