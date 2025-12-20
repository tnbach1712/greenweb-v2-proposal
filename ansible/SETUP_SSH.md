# SSH Setup Guide for Bastion Access

Account: **bachtn**
Bastion IP: **51.159.77.46**
Private Nodes: **192.168.0.100**, **192.168.0.101**

---

## Step 1: Copy SSH key to Bastion

```bash
# Copy your public key to bastion
ssh-copy-id -i ~/.ssh/id_rsa.pub bachtn@51.159.77.46

# If prompted, enter your password for bachtn@51.159.77.46
```

## Step 2: Test Bastion Connection

```bash
# Test SSH to bastion
ssh bachtn@51.159.77.46

# You should be able to login without password
# Type 'exit' to logout
```

## Step 3: From Bastion, copy SSH key to Private Nodes

### Option A: From your local machine (Recommended)

```bash
# Copy SSH key to node1 through bastion
ssh-copy-id -o ProxyJump=bachtn@51.159.77.46 bachtn@192.168.0.100

# Copy SSH key to node2 through bastion
ssh-copy-id -o ProxyJump=bachtn@51.159.77.46 bachtn@192.168.0.101

# Enter password when prompted
```

### Option B: From bastion host

```bash
# SSH to bastion first
ssh bachtn@51.159.77.46

# Generate SSH key on bastion (if needed)
ssh-keygen -t rsa -b 4096
# Press Enter for all prompts (use default location, no passphrase)

# Copy key to private nodes
ssh-copy-id bachtn@192.168.0.100
ssh-copy-id bachtn@192.168.0.101

# Exit bastion
exit
```

## Step 4: Setup SSH Config (Optional but Recommended)

Edit your SSH config:

```bash
nano ~/.ssh/config
```

Add this configuration:

```
# Bastion Host
Host bastion
    HostName 51.159.77.46
    User bachtn
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60

# Private Node 1
Host node1
    HostName 192.168.0.100
    User bachtn
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa

# Private Node 2
Host node2
    HostName 192.168.0.101
    User bachtn
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa
```

Save and exit (Ctrl+X, Y, Enter)

## Step 5: Test Connections

```bash
# Test bastion
ssh bastion echo "Bastion OK"

# Test node1
ssh node1 echo "Node1 OK"

# Test node2
ssh node2 echo "Node2 OK"

# Or test directly
ssh -J bachtn@51.159.77.46 bachtn@192.168.0.100 echo "Node1 OK"
ssh -J bachtn@51.159.77.46 bachtn@192.168.0.101 echo "Node2 OK"
```

## Step 6: Test Ansible

```bash
cd /home/bach/code/plan_df/ansible

# Test ping
ansible all -m ping

# Expected output:
# node1 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
# node2 | SUCCESS => {
#     "changed": false,
#     "ping": "pong"
# }
```

---

## Troubleshooting

### Permission denied (publickey)

```bash
# Make sure your SSH key has correct permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config

# Check if key is added to agent
ssh-add -l

# If not, add it
ssh-add ~/.ssh/id_rsa
```

### Cannot reach private nodes from bastion

```bash
# SSH to bastion
ssh bachtn@51.159.77.46

# Try ping private nodes
ping -c 3 192.168.0.100
ping -c 3 192.168.0.101

# If ping works, try SSH
ssh bachtn@192.168.0.100
ssh bachtn@192.168.0.101
```

### User 'bachtn' needs sudo access for K3s installation

```bash
# SSH to each node and check sudo access
ssh node1 sudo whoami
# Should output: root

# If you get "bachtn is not in the sudoers file", contact your admin
# Or add bachtn to sudo group:
# On each node:
sudo usermod -aG sudo bachtn
```

---

## Quick Commands Reference

```bash
# SSH to bastion
ssh bachtn@51.159.77.46

# SSH to node1 through bastion
ssh -J bachtn@51.159.77.46 bachtn@192.168.0.100

# SSH to node2 through bastion
ssh -J bachtn@51.159.77.46 bachtn@192.168.0.101

# With SSH config:
ssh bastion
ssh node1
ssh node2

# Copy file to node1
scp -o ProxyJump=bachtn@51.159.77.46 file.txt bachtn@192.168.0.100:/tmp/

# Run command on node1
ssh -J bachtn@51.159.77.46 bachtn@192.168.0.100 'uptime'
```

---

## Next Steps

After SSH is configured, run:

```bash
cd /home/bach/code/plan_df/ansible

# Test connection
ansible-playbook test-connection.yml

# Install K3s
ansible-playbook install-k3s.yml
```
