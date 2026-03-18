# Ansible for Azure Resource Provisioning

This guide explains how to prepare a Python virtual environment, install Ansible Azure dependencies, authenticate to Azure, and run Ansible playbooks to provision or manage Azure resources.

This README is written to work across common Linux distributions by showing examples for multiple package managers:
- `apt` for Ubuntu/Debian
- `dnf` for RHEL 8+/Rocky/AlmaLinux/Fedora
- `yum` for older RHEL/CentOS environments

Microsoft’s Azure + Ansible setup guidance also uses `yum`-based examples for CentOS and shows both credential-file and environment-variable authentication approaches. :contentReference[oaicite:0]{index=0}

---

## Prerequisites

Before starting, make sure the system has:

- Python 3.11 installed
- `pip`
- One of the supported Linux package managers: `apt`, `dnf`, or `yum`
- Internet access to install packages
- An Azure subscription
- Sufficient Azure RBAC permissions to create, update, or delete resources

The Microsoft Learn article notes that the Ansible control node needs Python installed, and for newer Ansible Core versions, newer Python versions are required. :contentReference[oaicite:1]{index=1}

---

## 1. Verify Python

Check whether Python 3.11 exists on the server:

```bash
which python3.11
python3.11 --version
```
Example expected output:
```log
/usr/bin/python3.11
Python 3.11.x
```
You can also check what Python 3.x versions are available:
```bash
ls /usr/bin/python3.*
```

## 2. Create a Python Virtual Environment

Create a dedicated virtual environment for Azure Ansible work:
```bash
python3.11 -m venv azure_venv
source azure_venv/bin/activate
```
After activation, your shell prompt should show the virtual environment name.

## 3. Install System Packages

Install the base packages required for Python virtual environments and Azure automation.

Ubuntu / Debian
```bash
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip
```
RHEL / Rocky / AlmaLinux / Fedora
```bash
sudo dnf install -y python3.11 python3.11-pip
```
Older CentOS / RHEL with yum
```bash
sudo yum install -y python3 python3-pip
```
The Microsoft Learn article’s CentOS example installs Python 3 and pip with yum install -y python3-pip before installing Ansible Azure components.[Get Started: Configure Ansible on an Azure VM](https://learn.microsoft.com/en-us/azure/developer/ansible/install-on-linux-vm?tabs=azure-cli) and [Ansible on Azure documentation](https://learn.microsoft.com/en-us/azure/developer/ansible/)

## 4. Install Ansible Azure Collection

Install the Azure Ansible collection:
```bash
ansible-galaxy collection install azure.azcollection --force
```