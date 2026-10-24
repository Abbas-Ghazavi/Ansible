# Ansible Notes

## 1. Ad-hoc Commands

```bash
# OS information
ansible -m shell -a "cat /etc/os-release" myservers

# System information
ansible myservers -m shell -a "df -h && free -m && hostname && ls /home"

# All host facts
ansible myservers -m setup

# OS family
ansible myservers -m setup -a "filter=ansible_os_family"
```

## 2. Ansible Configuration

File: `/etc/ansible/ansible.cfg`

```ini
[defaults]
interpreter_python = auto_silent
log_path = /var/log/ansible.log
```

## 3. Create Project Structure

```bash
mkdir -p /home/ansible/provision && cd /home/ansible/provision && mkdir -p inventory roles/Myprj/{defaults,files,vars,tasks,handlers,templates,meta} && touch myproject.yml inventory/hosts roles/Myprj/{tasks,handlers,vars,defaults,meta}/main.yml
```

## 4. Playbook

File: `myproject.yml`

```yaml
---
- hosts: myservers
  roles:
    - Myprj
```

## 5. Inventory

File: `inventory/hosts`

```ini
[myservers]
192.168.40.137
192.168.40.138
```

## 6. Role Tasks

File: `roles/Myprj/tasks/main.yml`

```yaml
- name: "******** Create Dir1 In /home *********"
  file:
    path: /home/dir1
    state: directory
    owner: root
    group: root
    mode: 755
  tags: create_dir1

- name: "******** Create File1 In /home *********"
  file:
    path: /home/dir1/file1
    state: touch
    owner: root
    group: root
    mode: 644
  tags: create_file1
```

## 7. Run Playbook

```bash
# Run all tasks
ansible-playbook -i inventory/hosts myproject.yml

# Run a specific task by tag
ansible-playbook -i inventory/hosts myproject.yml --tags create_dir1

ansible-playbook -i inventory/hosts myproject.yml --tags create_file1
```
