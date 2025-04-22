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

first create a ubuntu server image with openssh server installed.

dockerfile
```dockerfile
FROM ubuntu


RUN apt update -y
RUN apt install -y python3 python3-pip openssh-server
RUN mkdir /var/run/sshd
RUN echo 'root:root' | chpasswd
RUN echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config && \
    echo 'PasswordAuthentication no' >> /etc/ssh/sshd_config
    
# RUN ssh-keygen -A

EXPOSE 22

CMD ["/usr/sbin/sshd", "-D"]
```

build this in to ubuntu-server
```bash
docker build -t ubuntu-server .
```

First, let's create 5 Ubuntu-based containers named server1 to server5:

```bash
# Generate SSH key pair on the host machine (will be shared with all containers)
mkdir -p .ssh
ssh-keygen -t rsa -b 4096 -f ./.ssh/ansible_key -N ""
cp ./.ssh/ansible_key ~/.ssh/ansible_key
chmod 700 ~/.ssh
chmod 600 ~/.ssh/ansible_key
chmod 644 ~/.ssh/ansible_key.pub
eval $(ssh-agent -s)
ssh-add ~/.ssh/ansible_key


# Create the containers
for i in {1..5}; do
  echo -e "\nstarting server${i} from ubuntu-server image \n"
  docker run -d --name server${i} -p 220${i}:22 -v ./.ssh/ansible_key.pub:/root/.ssh/authorized_keys -v ./.ssh/ansible_key:/root/.ssh/id_rsa ubuntu-server sleep infinity
done
```

> to stop and remove these containers `for i in {1..5}; do docker stop server${i} && docker rm server${i};done`
## Step 3: Create Ansible Inventory File

First find IP of docker containers using

```bash
for i in {1..5}; do
  echo -e "\n IP if server${i}"
  docker inspect server${i} | grep IPAddress
done
```


in my case IPs are `172.17.0.4,172.17.0.5,172.17.0.6,172.17.0.7,172.17.0.8`
shown as

```bash
 IP if server1
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.4",
                    "IPAddress": "172.17.0.4",

 IP if server2
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.5",
                    "IPAddress": "172.17.0.5",

 IP if server3
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.6",
                    "IPAddress": "172.17.0.6",

 IP if server4
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.7",
                    "IPAddress": "172.17.0.7",

 IP if server5
            "SecondaryIPAddresses": null,
            "IPAddress": "172.17.0.8",
                    "IPAddress": "172.17.0.8",
```



Create `inventory.ini`:

```ini
[servers]



[servers:vars]
ansible_user=root
ansible_ssh_private_key_file=./ansible_key
ansible_python_interpreter=/usr/bin/python3
```

check ansible accesibility/connectivity to servers by pinging to verify ssh key pair.

```bash
#ansible ping
ansible -i inventory.ini servers -m ping```

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




### **Summary of Key Steps for Installation**
1. **Install Ansible** (`apt/yum/brew install ansible`).
2. **Generate SSH key pair** (`ssh-keygen`).
3. **Copy public key** (`ssh-copy-id` or Ansible playbook).
4. **Configure Ansible** to use the private key (`ansible.cfg` or inventory file).
5. **Test connectivity** (`ansible -m ping`).




