# AnsibleDemo
Ansible demo with docker container as servers

# The Need for Ansible in Server Management

Ansible is a powerful automation tool that addresses several critical challenges in server management:

1. **Scalability**: Managing multiple servers manually becomes impractical as infrastructure grows.
2. **Consistency**: Ensures identical configurations across all servers, reducing "works on my machine" issues.
3. **Efficiency**: Automates repetitive tasks, saving time and reducing human error.
4. **Idempotency**: Operations can be run multiple times without causing unintended changes.
5. **Infrastructure as Code**: Configuration is version-controlled and documented.

# Hands-on Ansible Exercise with Docker Containers

Let's create a complete setup with 5 Docker containers managed by Ansible.

## Step 1: Create Docker Containers

First, let's create 5 Ubuntu-based containers named server1 to server5:

```bash
# Generate SSH key pair on the host machine (will be shared with all containers)
ssh-keygen -t rsa -b 4096 -f ./ansible_key -N ""

# Create the containers
for i in {1..5}; do
  docker run -d --name server$i -v $(pwd)/ansible_key.pub:/root/.ssh/authorized_keys -v $(pwd)/ansible_key:/root/.ssh/id_rsa ubuntu sleep infinity
done
```

## Step 2: Install Required Software in Containers

We need to SSH into each container briefly to set up SSH server:

```bash
for i in {1..5}; do
  docker exec server$i bash -c "apt-get update && apt-get install -y openssh-server python3"
  docker exec server$i service ssh start
done
```

## Step 3: Create Ansible Inventory File

Create `inventory.ini`:

```ini
[servers]
server1 ansible_host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server1)
server2 ansible_host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server2)
server3 ansible_host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server3)
server4 ansible_host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server4)
server5 ansible_host=$(docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' server5)

[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=./ansible_key
ansible_python_interpreter=/usr/bin/python3
```

## Step 4: Create Ansible Playbook

Create `playbook.yml`:

```yaml
---
- name: Configure multiple servers
  hosts: servers
  become: yes

  tasks:
    - name: Update apt package index
      apt:
        update_cache: yes

    - name: Install Python 3.13 (or latest available)
      apt:
        name: python3
        state: latest

    - name: Create test file with content
      copy:
        dest: /root/test_file.txt
        content: |
          This is a test file created by Ansible
          Server name: {{ inventory_hostname }}
          Current date: {{ ansible_date_time.date }}

    - name: Display system information
      command: uname -a
      register: uname_output
      
    - name: Show disk space
      command: df -h
      register: disk_space

    - name: Print results
      debug:
        msg: 
          - "System info: {{ uname_output.stdout }}"
          - "Disk space: {{ disk_space.stdout_lines }}"
```

## Step 5: Run the Ansible Playbook

```bash
ansible-playbook -i inventory.ini playbook.yml
```

## Step 6: Verify Changes in Docker Containers

To verify the changes were applied correctly to all containers:

1. **Check Python version**:
   ```bash
   for i in {1..5}; do
     docker exec server$i python3 --version
   done
   ```

2. **Verify test file creation**:
   ```bash
   for i in {1..5}; do
     echo "Contents on server$i:"
     docker exec server$i cat /root/test_file.txt
   done
   ```

3. **Check system information** (should match Ansible output):
   ```bash
   for i in {1..5}; do
     docker exec server$i uname -a
   done
   ```

4. **Verify disk space** (should match Ansible output):
   ```bash
   for i in {1..5}; do
     docker exec server$i df -h
   done
   ```

## Additional Verification with Ansible

You can also use Ansible itself to verify the changes:

```bash
# Ad-hoc command to check file existence
ansible servers -i inventory.ini -m stat -a "path=/root/test_file.txt"

# Check Python version on all servers
ansible servers -i inventory.ini -m command -a "python3 --version"
```

This exercise demonstrates how Ansible can efficiently manage multiple servers with identical configurations, ensuring consistency across your infrastructure. The verification steps confirm that all changes were applied uniformly to each container.





---

# Ansible install 

Here are the steps to install Ansible and set up SSH key-based authentication for managing remote servers:

### **1. Install Ansible**

#### **On Ubuntu/Debian:**
```bash
sudo apt update
sudo apt install -y ansible
```

#### **On CentOS/RHEL:**
```bash
sudo yum install epel-release -y
sudo yum install ansible -y
```

#### **On macOS (using Homebrew):**
```bash
brew install ansible
```

#### **Verify Installation:**
```bash
ansible --version
```

---

### **2. Generate SSH Key Pair (if not already done)**
Ansible uses SSH to connect to remote servers. A key pair ensures passwordless authentication.

#### **Generate a new SSH key:**
```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/ansible_key -N ""
```
- `-t rsa`: Specifies RSA encryption
- `-b 4096`: Uses a 4096-bit key (more secure)
- `-f ~/.ssh/ansible_key`: Saves the key as `ansible_key` in `.ssh` directory
- `-N ""`: Sets an empty passphrase (optional, but useful for automation)

#### **Check Generated Keys:**
```bash
ls -l ~/.ssh/ansible_key*
```
- `ansible_key` → Private key (keep secure)
- `ansible_key.pub` → Public key (distribute to servers)

---

### **3. Copy the Public Key to Remote Servers**
Ansible needs the public key (`ansible_key.pub`) on all target servers.

#### **Option 1: Manual Copy (for a few servers)**
```bash
ssh-copy-id -i ~/.ssh/ansible_key.pub user@remote_server_ip
```
(Replace `user` and `remote_server_ip` with actual values.)

#### **Option 2: Automate with Ansible (for multiple servers)**
Create a playbook (`setup_ssh_keys.yml`):
```yaml
---
- name: Deploy SSH keys
  hosts: all
  become: yes
  vars:
    ansible_user: "your_remote_user"
    ansible_ssh_private_key_file: "~/.ssh/ansible_key"
  
  tasks:
    - name: Ensure .ssh directory exists
      file:
        path: ~/.ssh
        state: directory
        mode: '0700'

    - name: Copy public key to authorized_keys
      ansible.posix.authorized_key:
        user: "{{ ansible_user }}"
        state: present
        key: "{{ lookup('file', '~/.ssh/ansible_key.pub') }}"
```

Run it:
```bash
ansible-playbook -i inventory.ini setup_ssh_keys.yml
```

---

### **4. Configure Ansible to Use the SSH Key**
Edit (or create) `~/.ansible.cfg`:
```ini
[defaults]
inventory = ./inventory.ini
private_key_file = ~/.ssh/ansible_key
host_key_checking = False
```

Or specify the key in the **inventory file (`inventory.ini`)**:
```ini
[webservers]
server1 ansible_host=192.168.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/ansible_key
server2 ansible_host=192.168.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/ansible_key
```

---

### **5. Test SSH Key-Based Authentication**
```bash
ansible all -i inventory.ini -m ping
```
- If successful, you’ll see:
  ```json
  server1 | SUCCESS => { "ping": "pong" }
  ```

---

### **6. (Optional) Disable Password Authentication (Security Hardening)**
If you want to enforce key-based SSH only, modify `/etc/ssh/sshd_config` on remote servers:
```ini
PasswordAuthentication no
```
Then restart SSH:
```bash
sudo systemctl restart sshd
```

---

### **Summary of Key Steps for Installation**
1. **Install Ansible** (`apt/yum/brew install ansible`).
2. **Generate SSH key pair** (`ssh-keygen`).
3. **Copy public key** (`ssh-copy-id` or Ansible playbook).
4. **Configure Ansible** to use the private key (`ansible.cfg` or inventory file).
5. **Test connectivity** (`ansible -m ping`).




