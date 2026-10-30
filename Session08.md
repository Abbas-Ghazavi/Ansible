# Ansible — Quick Review Notes

## 1. Integrate Jenkins & Ansible

- Ansible روی Jenkins Agent نصب باشد.
- Playbook و Inventory در Repository قرار بگیرند.
- کلید SSH در Jenkins Credentials ذخیره شود و با SSH Agent استفاده شود.

نمونه `Jenkinsfile`:

```groovy
pipeline {
  agent any
  stages {
    stage('Deploy') {
      steps {
        sshagent(credentials: ['ansible-ssh-key']) {
          sh 'ansible-playbook -i inventory/hosts site.yml'
        }
      }
    }
  }
}
```

`ansible-ssh-key` شناسه‌ی Credential در Jenkins است.

## 2. Ansible & VMware

نصب Collection:

```bash
ansible-galaxy collection install community.vmware
```

نمونه روشن‌کردن VM در vCenter:

```yaml
- name: Power on VM
  community.vmware.vmware_guest:
    hostname: "{{ vcenter_hostname }}"
    username: "{{ vcenter_username }}"
    password: "{{ vcenter_password }}"
    validate_certs: false
    folder: /DC1/vm
    name: test-vm
    state: poweredon
  delegate_to: localhost
```

- ماژول از API مربوط به vCenter استفاده می‌کند؛ اتصال از سیستم کنترل‌کننده انجام می‌شود.
- رمزها را در Ansible Vault نگه دارید. `validate_certs: false` فقط برای محیط آزمایشگاهی و گواهی نامعتبر است.

## 3. Run Ansible on MikroTik

نصب Collectionها:

```bash
ansible-galaxy collection install community.routeros ansible.netcommon
```

نمونه Inventory:

```ini
[routers]
mt1 ansible_host=192.168.1.1

[routers:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_user=admin
```

اجرای دستور روی RouterOS:

```yaml
- name: Get MikroTik identity
  community.routeros.command:
    commands:
      - /system identity print
```

## 4. Run Ansible on Cisco

نصب Collectionها:

```bash
ansible-galaxy collection install cisco.ios ansible.netcommon
```

نمونه Inventory:

```ini
[cisco]
switch1 ansible_host=192.168.1.2

[cisco:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=cisco.ios.ios
ansible_user=admin
ansible_become=true
ansible_become_method=enable
```

تنظیم hostname:

```yaml
- name: Configure Cisco hostname
  cisco.ios.ios_config:
    lines:
      - hostname SW1
```

## 5. Develop Your Own Module

برای ساخت ماژول اختصاصی، یک پوشه‌ی `library` کنار Playbook بسازید:

```text
project/
├── playbook.yml
└── library/
    └── hello.py
```

نمونه‌ی ساده‌ی `library/hello.py`:

```python
#!/usr/bin/python
from ansible.module_utils.basic import AnsibleModule


def main():
    module = AnsibleModule(
        argument_spec={
            'name': {'type': 'str', 'required': True}
        }
    )

    module.exit_json(
        changed=False,
        message='Hello ' + module.params['name']
    )


if __name__ == '__main__':
    main()
```

استفاده در Playbook:

```yaml
- name: Run custom module
  hello:
    name: Abbas
  register: result
```

## 6. Module Format

ساختار کلی یک ماژول Python:

1. دریافت ورودی‌ها با `AnsibleModule` و `argument_spec`.
2. انجام عملیات موردنظر.
3. اعلام نتیجه با یکی از متدهای زیر:

```python
module.exit_json(changed=True, result='OK')
module.fail_json(msg='Something went wrong')
```

- `changed`: آیا تغییری انجام شده است؟
- `failed`: نتیجه‌ی اجرای ناموفق؛ با `fail_json` مدیریت می‌شود.
- `msg`: پیام نتیجه یا خطا.
- خروجی ماژول باید JSON معتبر باشد؛ از `print()` برای پیام‌های معمولی استفاده نکنید.

## 7. Use Return Values by Other Modules

با `register` خروجی یک Task را ذخیره و در Task بعدی استفاده می‌کنیم:

```yaml
- name: Get hostname
  command: hostname
  register: host_result
  changed_when: false

- name: Show hostname
  debug:
    msg: "{{ host_result.stdout }}"
```

خروجی‌های رایج:

- `result.stdout`: خروجی استاندارد دستور
- `result.stderr`: خطای استاندارد
- `result.rc`: کد خروجی
- `result.changed`: وضعیت تغییر

## 8. Ansible & Windows Host

Ansible برای مدیریت Windows معمولاً از WinRM استفاده می‌کند.

روی Control Node:

```bash
pip install pywinrm
```

روی Windows، WinRM باید فعال و قابل دسترس باشد. نمونه فعال‌سازی در PowerShell با دسترسی Administrator:

```powershell
Enable-PSRemoting -Force
```

نمونه Inventory:

```ini
[windows]
win1 ansible_host=192.168.1.10

[windows:vars]
ansible_connection=winrm
ansible_port=5985
ansible_winrm_transport=ntlm
ansible_user=Administrator
```

این نمونه از HTTP برای محیط آزمایش استفاده می‌کند. برای محیط واقعی، WinRM روی HTTPS و پورت 5986 را با گواهی معتبر تنظیم کنید.

## 9. Windows Modules

ماژول‌های Windows معمولاً از پیشوند `win_` استفاده می‌کنند:

```yaml
- name: Test Windows connection
  ansible.windows.win_ping:

- name: Run PowerShell command
  ansible.windows.win_shell: hostname

- name: Start a service
  ansible.windows.win_service:
    name: Spooler
    state: started

- name: Copy file to Windows
  ansible.windows.win_copy:
    src: test.txt
    dest: C:\Temp\test.txt
```

## 10. Ansible & Docker

نصب Collection:

```bash
ansible-galaxy collection install community.docker
```

نمونه اجرای Nginx در Docker:

```yaml
- name: Run nginx container
  community.docker.docker_container:
    name: web
    image: nginx:latest
    state: started
    ports:
      - "8080:80"
```

Docker باید روی میزبان اجراکننده‌ی ماژول در دسترس باشد و ماژول بتواند به Docker daemon متصل شود.

---

## یادآوری سریع

| موضوع | نکته‌ی اصلی |
|---|---|
| Jenkins | اجرای `ansible-playbook` در Pipeline |
| VMware | Collection `community.vmware` و API vCenter |
| MikroTik | `network_cli` و `community.routeros` |
| Cisco | `network_cli` و `cisco.ios` |
| Custom module | Python + `AnsibleModule` |
| Module output | JSON با `exit_json` یا `fail_json` |
| Return values | ذخیره با `register` و استفاده از متغیر |
| Windows | اتصال معمول با WinRM |
| Windows modules | ماژول‌هایی مثل `win_ping` و `win_service` |
| Docker | Collection `community.docker` |
