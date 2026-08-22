# AMPRNet Sverige – Crisis Information Server

This repository contains an Ansible playbook for automatically installing and configuring a central server for AMPRNet Sveriges local crisis information system.

The system uses:

- **DokuWiki** for creating and maintaining crisis-related information.
- **Syncthing** for distributing selected DokuWiki content to local nodes.
- **Ansible** for automating the central server installation.

The Syncthing distribution setup is intentionally left manual so that each deployment can decide which nodes receive which information.

## Architecture

```text
DokuWiki
    │
    ▼
/var/www/html/data/pages/
    ├── global/
    ├── kommun/
    └── trygghetspunkt/
    │
    ▼
Syncthing
    ├── Local node #1
    ├── Local node #2
    └── Other nodes
```

Administrators manually decide which folders are distributed to each node.

Example:

```text
global/
kommun/norrtalje/
trygghetspunkt/norrtalje-1/
```

## Repository Structure

```text
amprnet-crisis-wiki/
├── install.yml
├── inventory.ini
├── README.md
└── .gitignore
```

- `install.yml` — Ansible playbook for installing and configuring the central server.
- `inventory.ini` — Central server IP address and SSH username.
- `README.md` — Installation and administration instructions.
- `.gitignore` — Prevents unnecessary local files from being committed.

## Requirements

### Central server

- Supported Linux distribution (tested on Ubuntu Server 26.04).
- SSH access from the Ansible workstation.
- A user with `sudo` privileges.
- An IP address reachable from the administrator's computer.

### Ansible workstation

- Ansible installed.
- SSH access to the central server.

# Installation

## 1. Clone the repository

```bash
git clone https://github.com/apple-crust/amprnet-crisis-wiki.git
cd amprnet-crisis-wiki
```

## 2. Configure the inventory

Open:

```bash
nano inventory.ini
```

Replace the placeholder IP address and username with the central server's SSH details.

Example:

```ini
[central]
10.0.0.15 ansible_user=admin
```

## 3. Test the Ansible connection

```bash
ansible -i inventory.ini central -m ping -k -K
```

A successful connection should return:

```text
pong
```

## 4. Check the playbook syntax

```bash
ansible-playbook -i inventory.ini install.yml --syntax-check
```

## 5. Run the installation

For the first installation:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory.ini install.yml -k -K
```

`ANSIBLE_HOST_KEY_CHECKING=False` disables Ansible SSH host key checking for this command.

The playbook installs and configures:

- Apache2
- PHP and required extensions
- DokuWiki
- ACL support
- Syncthing
- The `crisis-sync` user
- Required permissions and ACLs
- DokuWiki page directories
- The `syncthing@crisis-sync` systemd service

The following directories are created automatically:

```text
/var/www/html/data/pages/global
/var/www/html/data/pages/kommun
/var/www/html/data/pages/trygghetspunkt
```

## 6. Complete the DokuWiki installation

After Ansible finishes, open:

```text
http://<central-server-ip>/install.php
```

Complete the DokuWiki web installer and create the administrator account.

## DokuWiki Structure

The installation provides three base namespaces:

```text
global
kommun
trygghetspunkt
```

Examples:

```text
global:crisisinformation
kommun:norrtalje:crisisinformation
trygghetspunkt:norrtalje:point1
```

The exact naming conventions can be determined by the administrators operating the installation.

# Syncthing

Syncthing runs as the dedicated system user:

```text
crisis-sync
```

and as the systemd service:

```text
syncthing@crisis-sync
```

The service starts automatically when the server boots.

The Syncthing GUI is normally available locally at:

```text
http://127.0.0.1:8384
```

The GUI is **not exposed directly to the network** by the Ansible installation.

## Accessing Syncthing remotely

Administrators can access the central server's Syncthing GUI from their own computers using SSH port forwarding:

```bash
ssh -L 8384:127.0.0.1:8384 <username>@<central-server-ip>
```

For example:

```bash
ssh -L 8384:127.0.0.1:8384 admin@10.0.0.15
```

Keep the SSH connection open and visit:

```text
http://127.0.0.1:8384
```

in the administrator's own browser. This allows administrators to manage the central Syncthing instance without exposing the GUI directly to the network.

# Configuring Syncthing Nodes

Node configuration is inentionally **manual**.

The Ansible playbook does **not** configure:

- Remote Device IDs
- Syncthing folders
- Folder sharing
- Node-specific paths
- Which nodes receive which information

The administrator configures each node through the Syncthing GUI:

1. Add the node as a **Remote Device** using its Device ID.
2. Create or select the appropriate folder.
3. Set the folder path on the central server.
4. Share the folder with the appropriate node.
5. Accept the folder on the remote node.
6. Configure the local path on the remote node.

For example, the central server may use:

```text
/var/www/html/data/pages/global
/var/www/html/data/pages/kommun/norrtalje
/var/www/html/data/pages/trygghetspunkt/norrtalje
```

Each node can receive only the folders relevant to its location.

# Permissions

DokuWiki is served by Apache using `www-data`.

Syncthing runs as `crisis-sync`, which is a member of the `www-data` group.

ACLs are configured on:

```text
/var/www/html/data/pages
```

This allows both DokuWiki and Syncthing to access the page files required for synchronization.

# What Ansible Automates

The playbook automates the installation and configuration of the central server:

- Apache2
- PHP and required extensions
- DokuWiki
- Apache configuration
- DokuWiki ownership and permissions
- ACL support
- Base DokuWiki namespaces
- Syncthing
- `crisis-sync`
- Syncthing permissions
- Syncthing systemd service

# What Remains Manual

The administrator is responsible for:

- Completing the DokuWiki web installer
- Creating DokuWiki administrator accounts
- Configuring Syncthing authentication
- Adding remote devices
- Configuring Syncthing folders
- Selecting which folders are shared with each node
- Configuring paths on remote nodes
- Creating municipality and safe-point namespaces
- Managing the actual crisis information

# Common Setup Issues

## Sudo permissions

The playbook uses:

```yaml
become: yes
```

The SSH user therefore needs sufficient sudo privileges. Also, the newest version of sudo can cause timeout issues.

If required, check the sudo configuration with:

```bash
sudo visudo
```

A possible configuration is:

```text
username ALL=(ALL) NOPASSWD:ALL
```

Replace `username` with the actual SSH username.

> **Security note:** `NOPASSWD:ALL` grants unrestricted sudo access without requiring a password. A more restrictive sudo policy is preferable for production systems. You can however comment the solution above after the installation. 

If passwordless sudo is configured, `-K` is not required:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory.ini install.yml -k
```

## SSH host key verification

If the first Ansible connection fails because the server's SSH host key is not yet known, use:

```bash
ANSIBLE_HOST_KEY_CHECKING=False ansible-playbook -i inventory.ini install.yml -k -K
```

This disables host key checking for that command only.

## DokuWiki installer

If DokuWiki has not yet been initialized, open:

```text
http://<central-server-ip>/install.php
```

and complete the web installation.

# Useful Commands

### Check Apache

```bash
sudo systemctl status apache2
```

### Check Syncthing

```bash
sudo systemctl status syncthing@crisis-sync
```

### Check DokuWiki page directories

```bash
sudo ls -la /var/www/html/data/pages
```

Expected directories:

```text
global
kommun
trygghetspunkt
```

### Check Ansible syntax

```bash
ansible-playbook -i inventory.ini install.yml --syntax-check
```

### Test the Ansible connection

```bash
ansible -i inventory.ini central -m ping -k -K
```

### Run the installation again

```bash
ansible-playbook -i inventory.ini install.yml -k -K
```

The playbook can safely be run again when necessary to ensure the configured server state is present.

# Project Goal

The goal of this project is to provide AMPRNet Sverige with a **reproducible and easily deployable central crisis information server**.

The installation provides:

- A DokuWiki server for creating and maintaining crisis information.
- A Syncthing service for distributing selected information to local nodes.
- A consistent directory structure for global, municipality-specific, and safe-point information.
- An Ansible-based installation that can be repeated on a new central server.

The actual Syncthing distribution topology remains under administrator control. Each deployment can therefore decide which nodes receive which information while keeping the central server installation consistent.