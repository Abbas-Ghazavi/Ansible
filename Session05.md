# Ansible Notes — Advanced Modules

## 1. Wait Module

### Wait for a specific time

```yaml
- name: Wait
  wait_for:
    timeout: 10
  tags: wait_m

- name: Show message
  debug:
    msg: "Finished waiting"
  tags: wait_m
```

`timeout: 10` در این مثال یعنی منتظر ماندن تا ۱۰ ثانیه.

### Wait for Port 8080

```yaml
- name: Wait for port 8080
  wait_for:
    port: 8080
    delay: 10
    timeout: 300
  tags: wait_port

- name: Show message
  debug:
    msg: "Port 8080 is now available"
  tags: wait_port
```

- `port`: پورتی که باید منتظر آماده‌شدن آن بمانیم.
- `delay`: مدت انتظار اولیه قبل از شروع بررسی.
- `timeout`: حداکثر زمان انتظار.

## 2. Ignore Errors

```yaml
- name: Ignore Error
  command: /bin/false
  ignore_errors: yes
  tags: ig

- name: After Error
  debug:
    msg: "Task continued after error"
  tags: ig
```

`ignore_errors: yes` اجازه می‌دهد در صورت شکست این Task، اجرای Taskهای بعدی ادامه پیدا کند.

## 3. Reboot Module

```yaml
- name: Reboot the server
  reboot:
    msg: "Reboot initiated by Ansible"
    connect_timeout: 5
    reboot_timeout: 300
    pre_reboot_delay: 0
    post_reboot_delay: 30
  tags: re

- name: Wait for SSH connection
  wait_for_connection:
    delay: 5
    timeout: 120
  tags: re

- name: Show message
  debug:
    msg: "Remote host has rebooted and is back online."
  tags: re
```

- `connect_timeout`: مهلت برقراری اتصال.
- `reboot_timeout`: حداکثر زمان انتظار برای راه‌اندازی مجدد.
- `post_reboot_delay`: زمان انتظار پس از Reboot.

نکته: ماژول `reboot` خودش منتظر بازگشت سیستم می‌ماند؛ Task دوم برای بررسی مجدد اتصال است.

## 4. Get URL Module

دانلود فایل از اینترنت روی هاست مقصد:

```yaml
- name: Download file
  get_url:
    url: http://download.anisa.co.ir/Course/LPIC-2/named.conf
    dest: /home/named.conf
    mode: '0644'
    timeout: 30
  tags: get
```

- `url`: آدرس فایل.
- `dest`: مسیر و نام فایل مقصد.
- `mode`: مجوز دسترسی فایل.
- `timeout`: حداکثر زمان درخواست.

## 5. Alpine Linux — Install NTP

این مثال مخصوص Alpine Linux است و به Collection زیر نیاز دارد:

```bash
ansible-galaxy collection install community.general
```

```yaml
- name: Install OpenNTPD
  community.general.apk:
    name: openntpd
    state: present
  tags: ntp

- name: Enable and start NTP service
  service:
    name: ntpd
    enabled: yes
    state: started
  tags: ntp
```

نکته: نام سرویس ممکن است بسته به نسخه و پیکربندی Alpine متفاوت باشد. قبل از اجرا آن را بررسی کن.

## 6. Firewalld Module

باز کردن پورت 80 در Firewall سیستم‌های دارای Firewalld:

```yaml
- name: Allow HTTP port
  firewalld:
    port: 80/tcp
    permanent: yes
    zone: public
    state: enabled
    immediate: yes
  tags: firewalld_mod
```

این مثال برای سیستم‌های دارای Firewalld است؛ در Alpine معمولاً از ابزارهای دیگری استفاده می‌شود.

## 7. Alpine Linux — MariaDB

### Initialize Database

```yaml
- name: Initialize MariaDB
  shell: mariadb-install-db --user=mysql --datadir=/var/lib/mysql
  args:
    creates: /var/lib/mysql/mysql
  tags: mariadb
```

`creates` مانع اجرای مجدد دستور در صورت وجود مسیر مشخص‌شده می‌شود.

### Enable and Start MariaDB

```yaml
- name: Enable MariaDB
  shell: |
    rc-update add mariadb default
    rc-service mariadb start
  tags: mariadb
```

این دستورها از ابزار مدیریت سرویس OpenRC در Alpine استفاده می‌کنند.

### Check MariaDB Status

```yaml
- name: Check MariaDB status
  shell: rc-service mariadb status
  register: mariadb_status
  changed_when: false
  tags: mariadb

- name: Show MariaDB status
  debug:
    msg: "MariaDB Status: {{ mariadb_status.stdout }}"
  tags: mariadb
```

`changed_when: false` باعث می‌شود Task بررسی وضعیت، تغییر سیستم گزارش نکند.

## 8. MySQL / MariaDB Database

ابتدا Collection موردنیاز را نصب کن:

```bash
ansible-galaxy collection install community.mysql
```

نصب کتابخانه Python برای اتصال به دیتابیس در Alpine:

```yaml
- name: Install PyMySQL
  community.general.apk:
    name: py3-pymysql
    state: present
  tags: mariadb
```

### Create Database

```yaml
- name: Create database
  community.mysql.mysql_db:
    name: db1
    state: present
    login_unix_socket: /run/mysqld/mysqld.sock
  tags: cre
```

- `name`: نام دیتابیس.
- `state: present`: ساخت دیتابیس در صورت نبودن.
- `login_unix_socket`: مسیر Socket دیتابیس.

نکته: مسیر Socket و دسترسی احراز هویت باید با تنظیمات MariaDB روی هاست مقصد مطابقت داشته باشند.

## 9. Alpine Linux — Firewall

### Install iptables

```yaml
- name: Install iptables
  community.general.apk:
    name: iptables
    state: present
  tags: firewall
```

### Allow Port 3306

```yaml
- name: Allow MySQL port
  iptables:
    chain: INPUT
    protocol: tcp
    destination_port: "3306"
    jump: ACCEPT
    action: insert
  tags: firewall
```

این Task قانون دسترسی به پورت TCP/3306 را به ابتدای زنجیره `INPUT` اضافه می‌کند.

نکته: این قانون لزوماً پس از Reboot باقی نمی‌ماند؛ برای ماندگاری باید قوانین Firewall را جداگانه ذخیره و بازیابی کنی.

