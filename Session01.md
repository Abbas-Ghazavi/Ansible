# Ansible Notes

## 1. Install

```bash
pip config --user set global.index-url https://mirror-pypi.runflare.com/simple
pip config --user set global.trusted-host mirror-pypi.runflare.com

sudo dnf install -y ansible-core
ansible --version
ansible -m ping localhost
```

## 2. Inventory

File: `/etc/ansible/hosts`

```ini
# Ungrouped
192.168.100.10

# Groups
[appservers]
192.168.200.10
192.168.200.11

[webservers]
192.168.10.10
192.168.10.11

# Groups of Groups
[anisaservers:children]
appservers
webservers
```

## 3. Passwordless SSH

```bash
# Generate SSH key
ssh-keygen -t rsa

# Copy key to hosts
ssh-copy-id root@192.168.40.137
ssh-copy-id root@192.168.40.138
```

Add to `/etc/ansible/hosts`:

```ini
[myservers]
192.168.40.137
192.168.40.138
```

Test connection:

```bash
ansible -m ping myservers
```

## 4. Useful Commands

```bash
ansible all --list-hosts
ansible-inventory --list
ansible -m ping localhost
ansible -m ping myservers
```
