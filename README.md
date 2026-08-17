# AMPRNet Sverige – Crisis Information Server

This repository contains an Ansible playbook for automatically installing and configuring a central server for AMPRNet Sveriges local crisis information system.

The system uses:

- **DokuWiki** for creating and maintaining crisis-related information.
- **Syncthing** for distributing selected DokuWiki content to local nodes.
- **Ansible** for automating the installation and configuration of the central server.

The purpose of this repository is to provide a reproducible Ansible-based installation of a central DokuWiki and Syncthing server for AMPRNet Sverige. The actual Syncthing distribution setup is intentionally left to the administrator and is configured manually through the Syncthing web interface.

## Architecture

The central server contains a DokuWiki installation whose pages are stored as files on the server.

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
    │
    ├── Local node #1
    ├── Local node #2
    └── Other nodes
```

Administrators decide manually which folders should be distributed to which nodes.

For example, one node might receive:

```text
global/
kommun/norrtalje/
trygghetspunkt/norrtalje/
```

while another node may receive a different selection of folders.

## Repository Structure

```text
amprnet-crisis-wiki/
├── install.yml
├── inventory.ini
├── README.md
└── .gitignore
```

### `install.yml`

The Ansible playbook that installs and configures the central server.

### `inventory.ini`

The Ansible inventory containing the central server's IP address and SSH username.

### `README.md`

Installation and administration instructions.

### `.gitignore`

Prevents local files and other unnecessary data from being committed to the repository.

## Requirements

### Central server

The server should have:

- A supported Linux distribution (tested on Ubuntu Server 26.04).
- SSH access from the computer running Ansible.
- A user with `sudo` privileges.
- Network access for downloading packages and the DokuWiki release.
- An IP address reachable from the administrator's computer.

### Ansible workstation

The computer running Ansible needs:

- Ansible installed.
- SSH access to the central server.

## Installation

### 1. Clone the repository

Clone the repository on the computer that will be used to install the central server.

```bash
git clone https://github.com/apple-crust/amprnet-crisis-wiki.git
cd amprnet-crisis-wiki
```

### 2. Configure the inventory

Open:

```bash
nano inventory.ini
```

Replace the IP address and username with the SSH details of the central server.

For example:

```ini
[central]
192.0.2.10 ansible_user=your_username
```

could become:

```ini
[central]
10.0.0.15 ansible_user=admin
```

### 3. Test the Ansible connection

Verify that Ansible can connect to the server:

```bash
ansible -i inventory.ini central -m ping -k -K
```

A successful connection should return:

```text
pong
```

### 4. Check the playbook syntax

Before running the installation, check the Ansible playbook syntax:

```bash
ansible-playbook -i inventory.ini install.yml --syntax-check
```

### 5. Run the installation

Run:

```bash
ansible-playbook -i inventory.ini install.yml -k -K
```

The playbook installs and configures:

- Apache2
- PHP
- Required PHP extensions
- DokuWiki
- ACL support
- Syncthing
- The `crisis-sync` service user
- Required permissions and ACLs
- The DokuWiki page directories
- The `syncthing@crisis-sync` systemd service

The following directories are created automatically:

```text
/var/www/html/data/pages/global
/var/www/html/data/pages/kommun
/var/www/html/data/pages/trygghetspunkt
```

## DokuWiki

After the Ansible installation has completed, open the central server's web address:

```text
http://<central-server-ip>/
```

Complete DokuWiki's initial web-based installation and create the administrator account.

DokuWiki namespaces can then be used to organize information. 

### Global information

Information intended to be distributed generally can be placed under:

```text
global:
```

For example:

```text
global:crisisinformation
```

### Municipality information

Municipality-specific information can be placed under:

```text
kommun:<municipality>:
```

For example:

```text
kommun:norrtalje:crisisinformation
```

DokuWiki automatically creates the corresponding namespace directory when content is created.

### Safe point information

Information related to a specific safe point can be organized under:

```text
trygghetspunkt:<municipality>:<safe-point>
```

For example:

```text
trygghetspunkt:norrtalje:point1
```

The exact naming convention for municipalities and safe points can be determined by the people operating the installation.

## Syncthing

Syncthing is installed as a system service running under the dedicated user:

```text
crisis-sync
```

The service is:

```text
syncthing@crisis-sync
```

and is configured to start automatically when the server boots.

The Syncthing web interface is normally available locally on:

```text
http://127.0.0.1:8384
```

The GUI is intentionally **not exposed directly to the network** by the Ansible installation.

### Accessing the Syncthing GUI remotely

Administrators can use SSH port forwarding from their own computer.

For example:

```bash
ssh -L 8384:127.0.0.1:8384 <username>@<central-server-ip>
```

After connecting, open the following address in a web browser on the administrator's own computer:

```text
http://127.0.0.1:8384
```

The administrator can then manage the central server's Syncthing instance without logging into the server's graphical environment.

## Configuring Syncthing Nodes

The connection between the central server and individual nodes is configured **manually** by the administrator.

The Ansible playbook does **not** configure:

- Remote Device IDs
- Syncthing folders
- Folder sharing
- Node-specific folder paths
- Which nodes receive which information

To configure a node, the administrator can:

1. Open the central server's Syncthing GUI.
2. Add the node as a **Remote Device** using its Device ID.
3. Create or select the appropriate Syncthing folder.
4. Set the folder path on the central server.
5. Share the folder with the appropriate remote device.
6. Accept the folder on the remote node.
7. Configure the local path on the remote node.
8. Repeat as necessary for additional folders.

For example, a central server might use:

```text
/var/www/html/data/pages/global
/var/www/html/data/pages/kommun/norrtalje
/var/www/html/data/pages/trygghetspunkt/norrtalje
```

A node can then be configured to receive only the folders relevant to that location.

This configuration is intentionally manual because each deployment may have different nodes, municipalities, safe points, and distribution requirements.

## Permissions

DokuWiki is served by Apache using the `www-data` user and group.

Syncthing runs as:

```text
crisis-sync
```

The `crisis-sync` user is a member of the `www-data` group.

ACLs are configured on:

```text
/var/www/html/data/pages
```

This allows both DokuWiki and Syncthing to access the page files required for synchronization.

## What Ansible Automates

The playbook is responsible for creating a working central server environment.

It automates:

- Apache installation and configuration
- PHP installation
- DokuWiki installation
- Apache configuration for DokuWiki
- DokuWiki file ownership and permissions
- ACL support
- Creation of the base DokuWiki namespaces
- Syncthing installation
- Creation of the `crisis-sync` user
- Syncthing permissions
- Syncthing systemd service configuration

## What Remains Manual

The following tasks are intentionally performed by the administrator:

- Completing the DokuWiki web installer
- Creating DokuWiki administrator accounts
- Configuring Syncthing authentication
- Adding remote Syncthing devices
- Configuring Syncthing folders
- Selecting which folders are shared with each node
- Configuring paths on remote nodes
- Creating municipality-specific namespaces
- Creating safe-point-specific namespaces
- Managing the actual crisis information

This separation keeps the Ansible installation generic while allowing each deployment to configure its own Syncthing topology.

## Useful Commands

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

Expected base directories:

```text
global
kommun
trygghetspunkt
```

### Check Ansible playbook syntax

```bash
ansible-playbook -i inventory.ini install.yml --syntax-check
```

### Run the installation again

```bash
ansible-playbook -i inventory.ini install.yml -k -K
```

The playbook can be run again if necessary to ensure that the configured server state is present.

## Project Goal

The goal of this project is to provide AMPRNet Sverige with a **reproducible and easily deployable central crisis information server**.

The central server provides the DokuWiki content and Syncthing distribution mechanism, while individual administrators retain control over the actual distribution topology.

This allows different nodes to receive different subsets of the available information while keeping the central installation consistent across deployments.