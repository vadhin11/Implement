# Ansible for Azure Resource Provisioning

This guide explains how to prepare a Python virtual environment, install Ansible Azure dependencies, authenticate to Azure, and run Ansible playbooks to provision or manage Azure resources.

This README is written to work across common Linux distributions by showing examples for multiple package managers:
- `apt` for Ubuntu/Debian
- `dnf` for RHEL 8+/Rocky/AlmaLinux/Fedora
- `yum` for older RHEL/CentOS environments

Microsoft’s Azure + Ansible setup guidance also uses `yum`-based examples for CentOS and shows both credential-file and environment-variable authentication approaches.

---

## Prerequisites

Before starting, make sure the system has:

- Python 3.11 installed
- `pip`
- One of the supported Linux package managers: `apt`, `dnf`, or `yum`
- Internet access to install packages
- An Azure subscription
- Sufficient Azure RBAC permissions to create, update, or delete resources

The Microsoft Learn article notes that the Ansible control node needs Python installed, and for newer Ansible Core versions, newer Python versions are required.

---

## 1. Verify Python

Check whether Python 3.11 exists on the server:

```bash
which python3.11
python3.11 --version
```

Example expected output:

```bash
/usr/bin/python3.11
Python 3.11.x
```

You can also check what Python 3.x versions are available:

```bash
ls /usr/bin/python3.*
```

---

## 2. Create a Python Virtual Environment

Create a dedicated virtual environment for Azure Ansible work:

```bash
python3.11 -m venv azure_venv
source azure_venv/bin/activate
```

After activation, your shell prompt should show the virtual environment name.

---

## 3. Install System Packages

Install the base packages required for Python virtual environments and Azure automation.

### Ubuntu / Debian

```bash
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip
```

### RHEL / Rocky / AlmaLinux / Fedora

```bash
sudo dnf install -y python3.11 python3.11-pip
```

### Older CentOS / RHEL with yum

```bash
sudo yum install -y python3 python3-pip
```

The Microsoft Learn article’s CentOS example installs Python 3 and pip with `yum install -y python3-pip` before installing Ansible Azure components.

---

## 4. Install Ansible Azure Collection

Install the Azure Ansible collection:

```bash
ansible-galaxy collection install azure.azcollection --force
```

Microsoft Learn documents `azure.azcollection` as the collection used for Ansible 2.10+ Azure automation.

---

## 5. Upgrade pip

Upgrade `pip` inside the virtual environment:

```bash
python -m pip install --upgrade pip
pip --version
```

Microsoft Learn also recommends upgrading pip before installing Azure-related Python packages.

---

## 6. Install Azure Collection Python Dependencies

Check the dependency file from the installed Azure collection:

```bash
ls ~/.ansible/collections/ansible_collections/azure/azcollection/
```

Depending on collection version, install dependencies from one of the following files:

### Option A: If `requirements.txt` exists

```bash
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

### Option B: If `requirements-azure.txt` exists

```bash
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements-azure.txt
```

Microsoft Learn’s example for `azure.azcollection` uses `requirements-azure.txt`.

---

## 7. Install Azure CLI

Azure CLI installation depends on the Linux distribution.

### Ubuntu / Debian

```bash
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash
```

### RHEL / Rocky / AlmaLinux

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo dnf install -y https://packages.microsoft.com/config/rhel/9.0/packages-microsoft-prod.rpm
sudo dnf install -y azure-cli
```

### Older yum-based systems

```bash
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
sudo yum install -y https://packages.microsoft.com/config/rhel/9.0/packages-microsoft-prod.rpm
sudo yum install -y azure-cli
```

Verify installation:

```bash
az version
```

---

## 8. Authenticate to Azure

Login interactively:

```bash
az login
```

Confirm the active subscription:

```bash
az account show
```

List subscriptions:

```bash
az account list --output table
```

Set the target subscription if needed:

```bash
az account set --subscription "<SUBSCRIPTION_ID_OR_NAME>"
```

---

## 9. Configure Azure Credentials

You can authenticate Ansible in two common ways.

Microsoft Learn documents both of these approaches: using a local credentials file and using environment variables. It also notes that credential files are better suited for development environments.

### Option 1: Azure credentials file

Create the Azure config directory:

```bash
mkdir -p ~/.azure
```

Create or edit the credentials file:

```bash
vi ~/.azure/credentials
```

Example content:

```ini
[default]
subscription_id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
client_id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
secret=xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
tenant=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

Set secure permissions:

```bash
chmod 600 ~/.azure/credentials
```

### Option 2: Environment variables

```bash
export AZURE_SUBSCRIPTION_ID=<subscription_id>
export AZURE_CLIENT_ID=<service_principal_app_id>
export AZURE_SECRET=<service_principal_password>
export AZURE_TENANT=<service_principal_tenant_id>
```

---

## 10. Run the Ansible Playbook

Execute the playbook using the virtual environment’s Ansible binary:

```bash
/home/admin/azure_venv/bin/ansible-playbook azure_ansible/delete_rg.yml
```

Or, after activating the virtual environment:

```bash
ansible-playbook azure_ansible/delete_rg.yml
```

---

## Example Project Structure

```bash
.
├── azure_venv/
├── azure_ansible/
│   ├── delete_rg.yml
│   ├── create_rg.yml
│   ├── create_vnet.yml
│   └── inventory
└── README.md
```

---

## Example Playbook: Delete a Resource Group

File: `azure_ansible/delete_rg.yml`

```yaml
---
- name: Delete Azure Resource Group
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Delete a resource group
      azure.azcollection.azure_rm_resourcegroup:
        name: myResourceGroup
        state: absent
```

Microsoft Learn includes both ad-hoc and playbook examples for creating and deleting Azure resource groups with Ansible.

---

## Example Playbook: Create a Resource Group

File: `azure_ansible/create_rg.yml`

```yaml
---
- name: Create Azure Resource Group
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Create resource group
      azure.azcollection.azure_rm_resourcegroup:
        name: myResourceGroup
        location: southeastasia
        state: present
```

---

## Example Playbook: Create a Virtual Network with Public and Private Subnets

File: `azure_ansible/create_vnet.yml`

```yaml
---
- name: Create Azure VNet and Subnets
  hosts: localhost
  connection: local
  gather_facts: false

  tasks:
    - name: Create virtual network
      azure.azcollection.azure_rm_virtualnetwork:
        resource_group: myResourceGroup
        name: myVirtualNetwork
        address_prefixes_cidr:
          - 10.1.0.0/16
          - 172.100.0.0/16
        dns_servers:
          - 127.0.0.1
          - 127.0.0.2
        tags:
          testing: testing
          delete: on-exit
        subnets:
          - name: public-subnet
            address_prefix: 10.1.1.0/24
          - name: private-subnet
            address_prefix: 10.1.2.0/24
```

---

## Example Ad-Hoc Command

Create a resource group with an ad-hoc Ansible command:

```bash
ansible localhost -m azure.azcollection.azure_rm_resourcegroup -a "name=myResourceGroup location=southeastasia"
```

Microsoft Learn explicitly shows this `azure.azcollection.azure_rm_resourcegroup` ad-hoc usage pattern for Ansible 2.10+.

---

## Troubleshooting

### AuthorizationFailed

Example:

```text
AuthorizationFailed
The client does not have authorization to perform action ...
```

Cause:
- The Azure user or Service Principal does not have the required RBAC role.

Fix:
- Assign a suitable role such as `Contributor`, `Network Contributor`, or another least-privilege role required for the target resources.
- Confirm the correct subscription is selected with:

```bash
az account show
```

### Azure collection not found

Check installed collections:

```bash
ansible-galaxy collection list | grep azure
```

Reinstall if needed:

```bash
ansible-galaxy collection install azure.azcollection --force
```

### Python dependency issue

Reinstall collection dependencies:

```bash
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
```

or:

```bash
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements-azure.txt
```

### Wrong Python interpreter

Check the interpreter and Ansible binary in use:

```bash
which python
which ansible-playbook
```

Run Ansible directly from the virtual environment if needed:

```bash
/home/admin/azure_venv/bin/ansible-playbook azure_ansible/delete_rg.yml
```

---

## Command Summary

```bash
which python3.11
python3.11 --version
python3.11 -m venv azure_venv
source azure_venv/bin/activate
ansible-galaxy collection install azure.azcollection --force
python -m pip install --upgrade pip
pip --version
ls ~/.ansible/collections/ansible_collections/azure/azcollection/
pip install -r ~/.ansible/collections/ansible_collections/azure/azcollection/requirements.txt
az login
mkdir -p ~/.azure
vi ~/.azure/credentials
/home/admin/azure_venv/bin/ansible-playbook azure_ansible/delete_rg.yml
```

---

## Reference

- Microsoft Learn: *Get Started - Configure Ansible on an Azure VM*  
  https://learn.microsoft.com/en-us/azure/developer/ansible/install-on-linux-vm?tabs=azure-cli
