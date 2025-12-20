# SSH Bastion Setup Guide

## Overview

- **Bastion Host**: 51.159.77.46 (public IP)
- **Private Node 1**: 192.168.0.100
- **Private Node 2**: 192.168.0.101

---

## Method 1: SSH ProxyJump (Recommended)

### One-time Access

```bash
# SSH to node 1 through bastion
ssh -J root@51.159.77.46 root@192.168.0.100

# SSH to node 2 through bastion
ssh -J root@51.159.77.46 root@192.168.0.101

# Or with username
ssh -J user@51.159.77.46 user@192.168.0.100
```

### Permanent Configuration

Create/edit `~/.ssh/config`:

```bash
# Open SSH config
nano ~/.ssh/config
```

Add this configuration:

```
# Bastion Host
Host bastion
    HostName 51.159.77.46
    User root
    IdentityFile ~/.ssh/id_rsa
    ServerAliveInterval 60
    ServerAliveCountMax 3

# Private Node 1
Host node1
    HostName 192.168.0.100
    User root
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa

# Private Node 2
Host node2
    HostName 192.168.0.101
    User root
    ProxyJump bastion
    IdentityFile ~/.ssh/id_rsa

# Wildcard for all private nodes
Host 192.168.0.*
    ProxyJump bastion
    User root
    IdentityFile ~/.ssh/id_rsa
```

Now you can simply use:

```bash
# SSH to bastion
ssh bastion

# SSH to node 1
ssh node1

# SSH to node 2
ssh node2

# Or directly
ssh 192.168.0.100
ssh 192.168.0.101
```

---

## Method 2: SSH Agent Forwarding

```bash
# Add your key to SSH agent
ssh-add ~/.ssh/id_rsa

# SSH to bastion with agent forwarding
ssh -A root@51.159.77.46

# Once on bastion, SSH to private nodes
ssh root@192.168.0.100
ssh root@192.168.0.101
```

---

## Method 3: SSH Tunnel (Port Forwarding)

```bash
# Create tunnel for node 1
ssh -L 2201:192.168.0.100:22 root@51.159.77.46

# In another terminal, SSH through the tunnel
ssh -p 2201 root@localhost
```

---

## Setup SSH Keys (If not already done)

### 1. Generate SSH key on your local machine

```bash
# Generate SSH key (if you don't have one)
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"

# Press Enter to accept default location (~/.ssh/id_rsa)
# Set a passphrase (optional but recommended)
```

### 2. Copy SSH key to bastion

```bash
# Copy your public key to bastion
ssh-copy-id root@51.159.77.46

# Or manually
cat ~/.ssh/id_rsa.pub | ssh root@51.159.77.46 "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

### 3. Copy SSH key to private nodes

**Option A: From your local machine (using ProxyJump)**

```bash
# Copy to node 1
ssh-copy-id -o ProxyJump=root@51.159.77.46 root@192.168.0.100

# Copy to node 2
ssh-copy-id -o ProxyJump=root@51.159.77.46 root@192.168.0.101
```

**Option B: From bastion host**

```bash
# SSH to bastion
ssh root@51.159.77.46

# Generate key on bastion (if needed)
ssh-keygen -t rsa -b 4096

# Copy to private nodes
ssh-copy-id root@192.168.0.100
ssh-copy-id root@192.168.0.101
```

---

## Test Connectivity

```bash
# Test bastion connection
ssh bastion echo "Bastion connected!"

# Test node 1 connection
ssh node1 echo "Node 1 connected!"

# Test node 2 connection
ssh node2 echo "Node 2 connected!"

# Test with actual IP
ssh -J root@51.159.77.46 root@192.168.0.100 uptime
```

---

## Troubleshooting

### Connection Refused

```bash
# Check if SSH is running on bastion
ssh root@51.159.77.46 'systemctl status sshd'

# Check if you can reach private nodes from bastion
ssh root@51.159.77.46 'ping -c 3 192.168.0.100'
ssh root@51.159.77.46 'ssh -v root@192.168.0.100'
```

### Permission Denied

```bash
# Check SSH key permissions
chmod 600 ~/.ssh/id_rsa
chmod 644 ~/.ssh/id_rsa.pub
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config

# Verify your public key is on the servers
ssh root@51.159.77.46 'cat ~/.ssh/authorized_keys'
```

### Firewall Issues

```bash
# On bastion, check if port 22 is open
ssh root@51.159.77.46 'sudo ufw status'

# Allow SSH if needed
ssh root@51.159.77.46 'sudo ufw allow 22/tcp'
```

---

## Security Best Practices

### 1. Use SSH Config with specific keys

```
Host bastion
    HostName 51.159.77.46
    User root
    IdentityFile ~/.ssh/bastion_key
    PasswordAuthentication no

Host node*
    ProxyJump bastion
    User root
    IdentityFile ~/.ssh/nodes_key
    PasswordAuthentication no
```

### 2. Disable password authentication on servers

```bash
# On each server, edit SSH config
sudo nano /etc/ssh/sshd_config

# Set these values:
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin prohibit-password

# Restart SSH
sudo systemctl restart sshd
```

### 3. Use SSH Agent with timeout

```bash
# Start SSH agent with 1-hour timeout
ssh-add -t 3600 ~/.ssh/id_rsa
```

---

## Quick Reference

```bash
# Direct commands through bastion
ssh -J bastion node1 'hostname'
ssh -J bastion node2 'uptime'

# Copy files through bastion
scp -o ProxyJump=bastion file.txt node1:/tmp/
scp -o ProxyJump=bastion node1:/etc/hosts ./

# Run command on all nodes
for node in node1 node2; do
  ssh $node 'hostname && uptime'
done
```
