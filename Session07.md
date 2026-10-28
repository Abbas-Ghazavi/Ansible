# Ansible Notes — Loops, Handlers, Vault & Tomcat

## 1. Loop — Create Multiple Databases

با استفاده از `loop` می‌توان چند دیتابیس را با یک تسک ایجاد کرد.

```yaml
- name: "Create Multiple databases"
  community.mysql.mysql_db:
    name: "{{ item }}"
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock
  loop:
    - clientDB
    - employeeDB
    - providerDB
  tags: loop1
```

در هر بار اجرای حلقه، مقدار `item` یکی از نام‌های دیتابیس است.

## 2. Loop — Create Multiple MySQL Users

ساخت چند کاربر با یک رمز عبور:

```yaml
- name: "Add user DB"
  community.mysql.mysql_user:
    name: "{{ item }}"
    state: present
    host: '%'
    login_unix_socket: /run/mysqld/mysqld.sock
    password: "{{ mysql_password }}"
  loop:
    - milad
    - mojtaba
  no_log: true
  tags: loop1
```

**نکته‌ها:**
- ابتدا Collection موردنیاز را نصب کنید:

```bash
ansible-galaxy collection install community.mysql
```

- متغیر `mysql_password` را در متغیرهای پروژه یا Ansible Vault تعریف کنید.
- `no_log: true` از نمایش اطلاعات این تسک در خروجی Ansible جلوگیری می‌کند.
- مقدار `host: '%'` دسترسی اتصال از میزبان‌های مختلف را مجاز می‌کند؛ آن را فقط در صورت نیاز استفاده کنید.

## 3. Loop with Nested Lists

فرض کنید می‌خواهیم دو کاربر را برای سه دیتابیس ایجاد کنیم و به هر کاربر روی هر دیتابیس دسترسی بدهیم.

```yaml
- name: "Create users with privileges on multiple databases"
  community.mysql.mysql_user:
    name: "{{ item[0] }}"
    priv: "{{ item[1] }}.*:ALL"
    append_privs: yes
    password: "{{ mysql_password }}"
    state: present
    host: '%'
    login_unix_socket: /run/mysqld/mysqld.sock
  with_nested:
    - ['milad', 'mojtaba']
    - ['clientDB', 'employeeDB', 'providerDB']
  no_log: true
  tags: loop
```

در این مثال:

- `item[0]` نام کاربر است.
- `item[1]` نام دیتابیس است.
- `append_privs: yes` دسترسی‌های جدید را به دسترسی‌های قبلی اضافه می‌کند.

حاصل ترکیب دو کاربر و سه دیتابیس، شش ترکیب کاربر و دیتابیس خواهد بود.

## 4. Loop over a Dictionary

اگر متغیری به نام `my_dict` داشته باشیم:

```yaml
vars:
  my_dict:
    name: abbas
    role: admin
```

می‌توانیم با `dict2items` روی کلیدها و مقادیر آن حلقه بزنیم:

```yaml
- name: "Loop over a dict"
  debug:
    msg: "{{ item.key }} - {{ item.value }}"
  loop: "{{ my_dict | dict2items }}"
  tags: loop2
```

خروجی:

```text
name - abbas
role - admin
```

## 5. Loop over Inventory Hosts

نمایش سرورهای موجود در گروه `servers`:

```yaml
- name: "Loop over inventory"
  debug:
    msg: "{{ item }}"
  loop: "{{ groups['servers'] }}"
  tags: loop3
```

نمایش تمام میزبان‌های Inventory:

```yaml
- name: "Loop over all hosts"
  debug:
    msg: "{{ item }}"
  loop: "{{ groups['all'] }}"
  tags: loop4
```

**نکته:** نام گروه `servers` باید دقیقاً با نام گروه تعریف‌شده در Inventory مطابقت داشته باشد.

## 6. Loop Control — Pause Between Items

با `loop_control.pause` می‌توان بین اجرای هر آیتم از حلقه مکث کرد.

```yaml
- name: "Loop over hosts with pause"
  debug:
    msg: "{{ item }}"
  loop: "{{ groups['all'] }}"
  loop_control:
    pause: 10
  tags: loop4
```

در این مثال بین آیتم‌ها ۱۰ ثانیه مکث می‌شود.

## 7. Track the Loop Index

با `index_var` می‌توان شماره‌ی هر آیتم در حلقه را ذخیره کرد.

```yaml
- name: "Track the index"
  debug:
    msg: "The {{ item }} index is {{ my_index }}"
  loop:
    - first
    - second
    - third
  loop_control:
    index_var: my_index
  tags: loop_5
```

**نکته:** شماره‌ی index از صفر شروع می‌شود؛ بنابراین مقدارهای آن `0`، `1` و `2` هستند.

## 8. Retry a Task Until a Condition Is Met

اجرای مجدد یک تسک تا زمانی که شرط مشخص‌شده برقرار شود:

```yaml
- name: "Retry a task until a condition is met"
  shell: cat /home/text.txt
  register: result
  until: "'ansible' in result.stdout"
  retries: 3
  delay: 10
  changed_when: false
  tags: loop11
```

در این مثال:

- `register: result` خروجی دستور را ذخیره می‌کند.
- `until` مشخص می‌کند تسک چه زمانی موفق محسوب شود.
- `retries: 3` تعداد تلاش‌های مجاز را تعیین می‌کند.
- `delay: 10` فاصله‌ی بین تلاش‌ها را به ثانیه مشخص می‌کند.
- `changed_when: false` مانع گزارش تغییر برای این بررسی می‌شود.

اگر کلمه‌ی `ansible` پس از تلاش‌های مجاز در خروجی پیدا نشود، تسک موفق نخواهد شد.

---

## 9. Import Another Playbook

برای اجرای یک Playbook دیگر از `import_playbook` استفاده می‌کنیم.

فایل اصلی، مثلاً `myproject.yml`:

```yaml
---
- hosts: test-hosts
  roles:
    - prj1

- import_playbook: prj2.yml
```

فایل `prj2.yml`:

```yaml
---
- hosts: all
  tasks:
    - name: "This is from prj2"
      debug:
        msg: "Hello from the second playbook!"
```

**نکته:** دستور `import_playbook` در سطح اصلی فایل قرار می‌گیرد؛ آن را داخل بخش `tasks` قرار ندهید.

---

## 10. Handlers — اجرای تسک در صورت تغییر

Handler شبیه یک تسک معمولی است، با این تفاوت که معمولاً فقط وقتی اجرا می‌شود که یک تسک دیگر با `notify` آن را فراخوانی کند.

### Task

```yaml
- name: "Update Tomcat config"
  template:
    src: tomcat_server.xml.j2
    dest: "{{ Tomcat.Conf_Dir }}/server.xml"
  notify: restart tomcat
```

اگر فایل مقصد واقعاً تغییر کند، Handler با نام `restart tomcat` فراخوانی می‌شود.

### Handler

در فایل `handlers/main.yml`:

```yaml
- name: restart tomcat
  service:
    name: tomcat
    state: restarted
```

### مزیت Handler

اگر چند تسک تنظیمات را تغییر دهند و همگی یک Handler را فراخوانی کنند، آن Handler در همان نوبت اجرای بخش مربوطه فقط یک بار اجرا می‌شود.

اگر هیچ تغییری رخ ندهد، Handler نیز اجرا نمی‌شود.

**نکته:** مقدار `notify` باید با نام Handler مطابقت داشته باشد. برای مثال، `notify: restart tomcat` باید Handlerای با نام `restart tomcat` داشته باشد.

---

## 11. Ansible Vault — رمزگذاری فایل‌ها و متغیرها

Ansible Vault برای رمزگذاری فایل‌ها و اطلاعات محرمانه مانند رمز عبور استفاده می‌شود.

### ایجاد فایل رمزگذاری‌شده

```bash
ansible-vault create test.yml
```

### ویرایش فایل رمزگذاری‌شده

```bash
ansible-vault edit prj3.yml
```

### تغییر رمز Vault

```bash
ansible-vault rekey prj3.yml
```

### مشاهده‌ی محتوای فایل

```bash
ansible-vault view prj1.yml
```

### رمزگذاری یک فایل موجود

```bash
ansible-vault encrypt prj1.yml
```

### رمزگشایی فایل

```bash
ansible-vault decrypt prj1.yml
```

**هشدار:** دستور `decrypt` فایل را به حالت غیررمزگذاری‌شده برمی‌گرداند؛ بنابراین از آن فقط وقتی استفاده کنید که لازم است.

### اجرای Playbook با درخواست رمز Vault

```bash
ansible-playbook -i inventory/anisahosts prj1.yml --ask-vault-pass
```

### رمزگذاری یک متغیر

```bash
ansible-vault encrypt_string --ask-vault-pass 'CHANGE_ME' --name 'db_password'
```

این دستور مقدار واردشده را رمزگذاری می‌کند و خروجی YAMLای شامل متغیر `db_password` تولید می‌کند. خروجی را می‌توان در فایل متغیرها قرار داد.

---

## 12. Vault IDs — چند رمز برای محیط‌های مختلف

Vault ID کمک می‌کند فایل‌ها یا متغیرهای مربوط به محیط‌های مختلف، مانند `common`، `staging` و `production`، با رمزهای متفاوت مدیریت شوند.

### ایجاد فایل‌های رمز عبور

نمونه‌ی مسیرها:

```text
~/.vault-pass.common
~/.vault-pass.staging
~/.vault-pass.production
```

این فایل‌ها باید حاوی رمزهای Vault مربوط به هر محیط باشند و مجوز دسترسی آن‌ها محدود باشد. از رمزهای واقعی ساده یا قابل حدس استفاده نکنید.

### تنظیم `ansible.cfg`

```ini
[defaults]
vault_identity_list = common@~/.vault-pass.common, staging@~/.vault-pass.staging, production@~/.vault-pass.production
```

### رمزگذاری فایل با یک Vault ID

```bash
ansible-vault encrypt --vault-id common@prompt prj1.yml
```

### اجرای Playbook با Vault ID

```bash
ansible-playbook -i inventory/anisahosts prj1.yml --vault-id common@prompt
```

در صورت استفاده از چند Vault ID، گزینه‌های موردنیاز را می‌توان تکرار کرد:

```bash
ansible-playbook -i inventory/anisahosts prj1.yml \
  --vault-id common@prompt \
  --vault-id staging@prompt
```

فقط Vault IDهایی را وارد کنید که برای فایل‌ها یا متغیرهای رمزگذاری‌شده‌ی آن Playbook لازم هستند.

---

# 13. Exercise 37 — Install and Configure Apache Tomcat

هدف تمرین:

1. نصب Java
2. دانلود و استخراج Tomcat در `/opt`
3. ساخت لینک نمادین `/opt/tomcat`
4. تنظیم پورت Tomcat روی 80
5. ساخت سرویس systemd
6. کپی فایل WAR
7. بازکردن پورت 80 در فایروال
8. فعال‌سازی و اجرای سرویس Tomcat

### Playbook نمونه

این نمونه با سبک دوره نوشته شده است و برای محیطی مشابه CentOS 7 در نظر گرفته شده است. فایل `sample.war` باید در مسیر قابل دسترس برای Ansible، مثلاً کنار Playbook، موجود باشد.

```yaml
---
- name: Install and Configure Apache Tomcat
  hosts: servers
  become: true

  vars:
    tomcat_version: "9.0.41"
    tomcat_url: "https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.41/bin/apache-tomcat-9.0.41.tar.gz"
    tomcat_archive: "/opt/apache-tomcat-{{ tomcat_version }}.tar.gz"
    tomcat_home: "/opt/tomcat"

  tasks:
    - name: Install Java
      yum:
        name: java-1.8.0-openjdk-devel
        state: present
      tags: Install_Java

    - name: Download Apache Tomcat
      get_url:
        url: "{{ tomcat_url }}"
        dest: "{{ tomcat_archive }}"
      tags: download_tomcat

    - name: Extract Tomcat
      unarchive:
        src: "{{ tomcat_archive }}"
        dest: /opt
        remote_src: yes
        creates: "/opt/apache-tomcat-{{ tomcat_version }}"
      tags: Install_tomcat

    - name: Create Tomcat soft link
      file:
        src: "/opt/apache-tomcat-{{ tomcat_version }}"
        dest: "{{ tomcat_home }}"
        state: link
      tags: tomcat_link

    - name: Configure Tomcat port
      replace:
        path: "{{ tomcat_home }}/conf/server.xml"
        regexp: 'port="8080"'
        replace: 'port="80"'
      tags: config_tomcat

    - name: Create Tomcat service
      copy:
        dest: /etc/systemd/system/tomcat.service
        content: |
          [Unit]
          Description=Apache Tomcat Web Application Container
          After=network.target

          [Service]
          Type=forking
          Environment=JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
          Environment=CATALINA_HOME=/opt/tomcat
          Environment=CATALINA_BASE=/opt/tomcat
          ExecStart=/opt/tomcat/bin/startup.sh
          ExecStop=/opt/tomcat/bin/shutdown.sh
          User=root
          Group=root
          Restart=on-failure
          RestartSec=10

          [Install]
          WantedBy=multi-user.target
      tags: tomcat_service

    - name: Deploy sample WAR file
      copy:
        src: sample.war
        dest: "{{ tomcat_home }}/webapps/sample.war"
      tags: deploy_war

    - name: Configure firewall to allow port 80
      ansible.posix.firewalld:
        port: 80/tcp
        permanent: yes
        state: enabled
        immediate: yes
      tags: config_firewall

    - name: Reload systemd and start Tomcat
      systemd:
        name: tomcat
        state: started
        enabled: yes
        daemon_reload: yes
      tags: start_tomcat
```

### نکته‌های مهم پروژه

- مقدار `tomcat_version` باید با نسخه‌ی موجود در URL و نام پوشه‌ی استخراج‌شده مطابقت داشته باشد.
- مسیر `JAVA_HOME` در سرویس، نمونه است؛ آن را با مسیر Java نصب‌شده روی سرور تطبیق دهید.
- برای استفاده از `ansible.posix.firewalld` ممکن است لازم باشد Collection مربوط را نصب کنید:

```bash
ansible-galaxy collection install ansible.posix
```

- سرویس `firewalld` باید روی مقصد نصب و فعال باشد.
- اجرای Tomcat با کاربر `root` برای محیط واقعی توصیه نمی‌شود؛ در این مثال فقط ساختار سرویس دوره حفظ شده است. همچنین نسخه‌ی Tomcat `9.0.41` قدیمی است و برای محیط عملیاتی باید نسخه‌ی پشتیبانی‌شده و به‌روز انتخاب شود.
- بازکردن مستقیم پورت 80 برای Tomcat نیاز به ملاحظات امنیتی دارد؛ در محیط واقعی معمولاً استفاده از Reverse Proxy و اجرای Tomcat با کاربر غیرممتاز انتخاب مناسب‌تری است.
