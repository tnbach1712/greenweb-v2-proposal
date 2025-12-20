# Fix Ansible ModuleNotFoundError

## Problem
```
ModuleNotFoundError: No module named 'ansible.module_utils.six.moves'
```

This error occurs because **Ansible 2.10.8 is too old** and incompatible with Python 3.12 on remote hosts.

---

## Solution: Upgrade Ansible

### Option 1: Using pipx (Recommended)

```bash
# Install pipx if not installed
sudo apt update
sudo apt install pipx -y
pipx ensurepath

# Restart shell or run:
source ~/.bashrc

# Install latest Ansible
pipx install --include-deps ansible

# Verify
ansible --version
# Should show ansible 2.16+ or higher
```

### Option 2: Using pip3

```bash
# Remove old Ansible
sudo apt remove ansible -y

# Install latest Ansible via pip
pip3 install --user ansible

# Add to PATH (if not already)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Verify
ansible --version
```

### Option 3: Using Ansible PPA (Ubuntu)

```bash
# Remove old version
sudo apt remove ansible -y

# Add Ansible PPA
sudo apt update
sudo apt install software-properties-common -y
sudo add-apt-repository --yes --update ppa:ansible/ansible

# Install latest Ansible
sudo apt install ansible -y

# Verify
ansible --version
```

---

## After Upgrade: Test Again

```bash
cd /home/bach/code/plan_df/ansible

# Test ping
ansible all -m ping

# Expected output:
# node1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

---

## Quick Command

Run this one-liner to upgrade:

```bash
# Remove old + install new
sudo apt remove ansible -y && \
sudo add-apt-repository --yes --update ppa:ansible/ansible && \
sudo apt install ansible -y && \
ansible --version
```

Then test:

```bash
cd /home/bach/code/plan_df/ansible && ansible all -m ping
```
