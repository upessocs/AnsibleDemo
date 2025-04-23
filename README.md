# AnsibleDemo
Ansible demo with docker container as servers

# The Need for Ansible in Server Management

Ansible is a powerful automation tool that addresses several critical challenges in server management:

1. **Scalability**: Managing multiple servers manually becomes impractical as infrastructure grows.
2. **Consistency**: Ensures identical configurations across all servers, reducing "works on my machine" issues.
3. **Efficiency**: Automates repetitive tasks, saving time and reducing human error.
4. **Idempotency**: Operations can be run multiple times without causing unintended changes.
5. **Infrastructure as Code**: Configuration is version-controlled and documented.

---
# Create Docker image and test ssh login

> First create ssh key-pair and then create custom ubuntu-server image with open-ssh configured.

# Testing SSH Key Pair Login with Docker and WSL

Here's a step-by-step guide to create and test SSH key pair authentication between WSL and a Docker container running Ubuntu with OpenSSH server:

## 1. Create SSH Key Pair in WSL

First, generate an SSH key pair in your WSL environment:

```bash
# Generate RSA key pair (accept defaults when prompted)
ssh-keygen -t rsa -b 4096

# This creates:
# Private key: ~/.ssh/id_rsa
# Public key: ~/.ssh/id_rsa.pub
```

## 2. Create Dockerfile for Ubuntu SSH Server

Create a Dockerfile with the following content:

```dockerfile
FROM ubuntu:latest

# Update and install openssh-server
RUN apt-get update && \
    apt-get install -y openssh-server && \
    mkdir /var/run/sshd

# Configure SSH
RUN echo 'root:password' | chpasswd
RUN sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config
RUN sed -i 's/#PasswordAuthentication yes/PasswordAuthentication yes/' /etc/ssh/sshd_config

# SSH login fix. Otherwise user is kicked off after login
RUN sed 's@session\s*required\s*pam_loginuid.so@session optional pam_loginuid.so@g' -i /etc/pam.d/sshd

# Copy the public key from your WSL to the container
COPY id_rsa.pub /root/.ssh/authorized_keys
RUN chmod 600 /root/.ssh/authorized_keys

# Expose SSH port
EXPOSE 22

# Start SSH service when container starts
CMD ["/usr/sbin/sshd", "-D"]
```

## 3. Build the Docker Image

```bash
# Copy your public key to the current directory
cp ~/.ssh/id_rsa.pub .

# Build the Docker image
docker build -t ubuntu-server .

# Remove the public key from build directory (optional)
rm id_rsa.pub
```

## 4. Run the Docker Container

```bash
# Run the container with port mapping
docker run -d -p 2222:22 -p 8221:8221 --name ssh-test ubuntu-server
```

## 5. Find the Container IP Address

```bash
# Get the container's IP address
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ssh-test
```

Note this IP address (e.g., 172.17.0.2)

## 6. Test SSH Connections

### Test password authentication:
```bash
ssh root@localhost -p 2222
# When prompted, enter password: "password"
```

### Test key-based authentication:
```bash
ssh -i ~/.ssh/id_rsa root@localhost -p 2222
# Should log in without password prompt
```

## 7. Alternative: Using Container IP Directly

If you prefer to use the container's IP instead of port mapping:

```bash
# Using the IP you got from docker inspect
ssh root@172.17.0.2
# or with key:
ssh -i ~/.ssh/id_rsa root@172.17.0.2
```

## Verification Steps

1. Confirm password authentication works
2. Confirm key-based authentication works without password prompt
3. Verify you can access port 8221 (though nothing is running there yet)
4. Check logs if any issues occur: `docker logs ssh-test`

## Clean Up

When done testing:
```bash
docker stop ssh-test
docker rm ssh-test
```

This setup demonstrates both SSH key-based authentication and password authentication working together in a Docker container.

---



Here's a simplified and corrected version of your Ansible with Docker tutorial:

# Simplified Ansible with Docker Exercise

## Step 1: Create SSH Key Pair
```bash
mkdir -p .ssh
ssh-keygen -t rsa -b 4096 -f ./.ssh/ansible_key -N ""
chmod 600 ./.ssh/ansible_key
```

## Step 2: Create Dockerfile
```dockerfile
FROM ubuntu:latest

RUN apt-get update && \
    apt-get install -y openssh-server python3 && \
    mkdir /var/run/sshd && \
    rm -rf /var/lib/apt/lists/*

RUN echo 'root:root' | chpasswd && \
    sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config && \
    sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

EXPOSE 22
CMD ["/usr/sbin/sshd", "-D"]
```

## Step 3: Build and Run Containers
```bash
# Build image
docker build -t ubuntu-server .

# Create 4 containers
for i in {1..4}; do
  docker run -d --name server${i} \
    -v $(pwd)/.ssh/ansible_key.pub:/root/.ssh/authorized_keys \
    ubuntu-server
done
```

## Step 4: Create Ansible Inventory
```bash
# Get container IPs
echo "[servers]" > inventory.ini
for i in {1..4}; do
  docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server${i} >> inventory.ini
done

# Add inventory variables
cat << EOF >> inventory.ini

[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=./.ssh/ansible_key
ansible_python_interpreter=/usr/bin/python3
EOF
```

## Step 5: Test Connectivity
```bash
# Manual SSH test
ssh -i ./.ssh/ansible_key root@<container_ip>

# Ansible ping test
ansible all -i inventory.ini -m ping
```

## Step 6: Create Playbook (update.yml)
```yaml
---
- name: Update and configure servers
  hosts: all
  become: yes

  tasks:
    - name: Update apt packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install required packages
      apt:
        name: ["vim", "htop", "wget"]
        state: present

    - name: Create test file
      copy:
        dest: /root/ansible_test.txt
        content: "Configured by Ansible on {{ inventory_hostname }}"
```

## Step 7: Run Playbook
```bash
ansible-playbook -i inventory.ini update.yml
```

## Step 8: Verify Changes
```bash
# Using Ansible
ansible all -i inventory.ini -m command -a "cat /root/ansible_test.txt"

# Manually via Docker
for i in {1..4}; do
  docker exec server${i} cat /root/ansible_test.txt
done
```

## Cleanup
```bash
# Stop and remove containers
for i in {1..4}; do docker rm -f server${i}; done
```

Key improvements made:
1. Simplified Dockerfile with proper cleanup
2. Fixed volume mounting for SSH keys
3. Corrected inventory file generation
4. Simplified playbook to focus on core tasks
5. Added proper verification steps
6. Removed unnecessary port mapping since we're using container IPs
7. Fixed SSH key permissions
8. Added proper cleanup command

The workflow is now:
1. Setup SSH keys → 2. Build image → 3. Launch containers → 4. Create inventory → 5. Test → 6. Run playbook → 7. Verify → 8. Cleanup

