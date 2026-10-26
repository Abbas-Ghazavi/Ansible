# Ansible Notes — Modules

## 1. Variables & Debug

```yaml
- name: Set variable
  set_fact:
    myvar: 123
  tags: debug_mod

- name: Show variable
  debug:
    var: myvar
  tags: debug_mod

- name: Show user list
  debug:
    msg: "{{ user_list }}"
  tags: show_list

- name: Show user list 2
  debug:
    msg: "{{ user_list2 }}"
  tags: show_list2
```

**Variables — `vars/main.yml`**

```yaml
user_list:
  - Amir
  - Abbas

user_list2: [Reza, Iman]

mysql_version: 8
```

نکته: نام متغیر نباید فاصله، نقطه یا خط تیره داشته باشد و نباید با عدد شروع شود. برای نام‌گذاری از `_` استفاده کن.

## 2. Command Module

اجرای دستور بدون Shell:

```yaml
- name: Test command module
  command: mkdir /home/test
  tags: command_mod
```

نکته: اگر پوشه از قبل وجود داشته باشد، دستور `mkdir` خطا می‌دهد.

**Register — ذخیره خروجی دستور**

```yaml
- name: Run cat command
  command: cat /home/dir1/file1
  register: catty
  tags: command_mod

- name: Show output
  debug:
    msg: "{{ catty.stdout }}"
  tags: command_mod
```

**`chdir` و `creates`**

```yaml
- name: Run command
  command: cat file1
  args:
    chdir: /home/dir1
    creates: /home/dir4
  register: catty
  tags: command_mod
```

- `chdir`: اجرای دستور از مسیر مشخص‌شده.
- `creates`: اگر `/home/dir4` وجود داشته باشد، دستور اجرا نمی‌شود.

## 3. Script Module

اجرای اسکریپت از سیستم کنترل‌کننده روی هاست مقصد:

```yaml
- name: Run script
  script: script.sh
  tags: sh
```

فایل `script.sh` را در مسیر زیر قرار بده:

`roles/Myprj/files/script.sh`

برای جلوگیری از اجرای اسکریپت در صورت وجود یک مسیر، می‌توان از `creates` یا `removes` استفاده کرد.

## 4. Shell Module

اجرای دستورهای Shell مانند `&&` و Pipe:

```yaml
- name: Shell commands
  shell: |
    echo "Hello" > /home/dir1/file1
    cat /home/dir1/file1
    df -h
    ps aux | grep -i ansible
  register: logss
  tags: she

- name: Show output
  debug:
    var: logss.stdout
  tags: she
```

نکته: برای دستورهای ساده از `command` استفاده کن؛ برای قابلیت‌های Shell از `shell`.

## 5. Copy Module

کپی فایل از Controller به هاست مقصد:

```yaml
- name: Copy script
  copy:
    src: script.sh
    dest: /home/dir1/script.sh
    owner: root
    group: root
    mode: '0644'
  tags: cpp
```

فایل منبع در مسیر `roles/Myprj/files/script.sh` قرار می‌گیرد.

## 6. Fetch Module

کپی فایل از هاست مقصد به Controller:

```yaml
- name: Fetch file
  fetch:
    src: /home/dir1/file1
    dest: /home/
  tags: fetch_mod
```

نکته: به‌صورت پیش‌فرض، فایل‌های هاست‌های مختلف در مسیرهای جداگانه ذخیره می‌شوند.

## 7. Group & User Modules

**Create Group**

```yaml
- name: Create group
  group:
    name: Devops
    state: present
  tags: cg
```

**Create User**

```yaml
- name: Create user
  user:
    name: Abbas
    group: Devops
    state: present
    shell: /bin/bash
  tags: cre
```

- `present`: ایجاد در صورت نبودن.
- `absent`: حذف.
- برای ساخت گروه یا کاربر سیستمی معمولاً به دسترسی `root` نیاز داری؛ در Playbook می‌توانی `become: true` قرار بدهی.

## 8. DNF / YUM Modules

برای سیستم‌های Fedora و RHEL از `dnf` استفاده کن. نام `yum` در بسیاری از نسخه‌ها همچنان به‌عنوان سازگاری قدیمی در دسترس است.

**Install Nginx**

```yaml
- name: Install nginx
  dnf:
    name: nginx
    state: present
  tags: yum_mod
```

**Update Nginx**

```yaml
- name: Update nginx
  dnf:
    name: nginx
    state: latest
  tags: update_nginx
```

**Update All Packages Except Kernel**

```yaml
- name: Update packages
  dnf:
    name: '*'
    state: latest
    exclude: kernel*
  tags: update_packages
```

- `present`: نصب پکیج در صورت نیاز.
- `latest`: نصب یا به‌روزرسانی به آخرین نسخه موجود.
- `exclude`: مستثنا کردن پکیج‌های مشخص‌شده.

### YUM Repository Module

افزودن مخزن EPEL در سیستم‌های سازگار با RHEL:

```yaml
- name: Add EPEL repository
  yum_repository:
    name: epel
    description: Extra Packages for Enterprise Linux
    baseurl: https://download.fedoraproject.org/pub/epel/$releasever/Everything/$basearch/
    gpgcheck: yes
    enabled: yes
    state: present
  tags: yum_repo_mod
```

نکته: برای نصب واقعی، از معتبر بودن آدرس مخزن و نصب بودن کلید GPG مربوط به EPEL مطمئن شو.

برای حذف مخزن، `state: absent` را تنظیم کن.

## 9. APT Module

برای سیستم‌های Ubuntu و Debian.

**Remove Nginx**

```yaml
- name: Remove nginx
  apt:
    name:
      - nginx
      - nginx-common
    state: absent
    purge: yes
  tags: apt_mod
```

**Remove Unused Dependencies**

```yaml
- name: Remove unused packages
  apt:
    autoremove: yes
  tags: apt_mod
```

- `state: absent`: حذف پکیج.
- `purge: yes`: حذف فایل‌های تنظیمات مربوط به پکیج.
- `autoremove: yes`: حذف وابستگی‌های بلااستفاده.

## 10. Package Facts

بررسی نصب بودن پکیج‌ها:

```yaml
- name: Check installed packages
  package_facts:
    manager: auto
  tags: httpd_list

- name: Show httpd status
  debug:
    msg: "{{ 'httpd is installed' if 'httpd' in ansible_facts.packages else 'httpd is NOT installed' }}"
  tags: httpd_list
```

این مثال برای سیستم‌هایی است که نام پکیج Apache در آن‌ها `httpd` است؛ در Ubuntu معمولاً نام آن `apache2` است.

## 11. Template Module

برای ساخت فایل تنظیمات با استفاده از متغیرها و قالب Jinja2.

```yaml
- name: Configure nginx
  template:
    src: nginx.conf.j2
    dest: /etc/nginx/nginx.conf
    owner: root
    group: root
    mode: '0644'
    backup: yes
  tags: template_mod
```

فایل قالب را در مسیر زیر قرار بده:

`roles/Myprj/templates/nginx.conf.j2`

- `backup: yes`: تهیه نسخه پشتیبان قبل از جایگزینی فایل.
- `force: yes`: جایگزینی فایل مقصد در صورت نیاز؛ معمولاً مقدار پیش‌فرض آن فعال است.

## 12. Lineinfile Module

برای اضافه یا جایگزین کردن یک خط در فایل.

```yaml
- name: Change nginx port
  lineinfile:
    path: /etc/nginx/nginx.conf
    regexp: '^\s*listen\s+80 default_server;'
    line: 'listen 8080 default_server;'
    backup: yes
  tags: lineinfile_mod
```

- `path`: مسیر فایل.
- `regexp`: الگوی خطی که باید پیدا شود.
- `line`: خط جدید یا جایگزین.
- `insertafter`: درج بعد از خط منطبق.
- `insertbefore`: درج قبل از خط منطبق.

اگر الگو پیدا نشود و `insertafter` یا `insertbefore` مشخص نشده باشد، خط جدید به انتهای فایل اضافه می‌شود.

## 13. Replace Module

برای جایگزینی تمام تطبیق‌های یک الگو در فایل:

```yaml
- name: Replace nginx port
  replace:
    path: /etc/nginx/nginx.conf
    regexp: 'listen 80 default_server;'
    replace: 'listen 8080 default_server;'
    backup: yes
  tags: replace_mod
```

- `regexp`: متن یا الگوی مورد جست‌وجو.
- `replace`: متن جایگزین.
- `backup: yes`: تهیه نسخه پشتیبان.

**تفاوت:** `lineinfile` برای مدیریت یک خط مناسب است؛ `replace` می‌تواند تمام تطبیق‌های الگو را جایگزین کند.

## 14. Service Module

مدیریت سرویس‌ها:

```yaml
- name: Enable and start nginx
  service:
    name: nginx
    enabled: yes
    state: started
  tags: nginx_service
```

- `enabled: yes`: اجرا هنگام بوت سیستم.
- `state: started`: اطمینان از روشن بودن سرویس.
- `state: stopped`: توقف سرویس.
- `state: restarted`: راه‌اندازی مجدد.
- `state: reloaded`: بارگذاری مجدد تنظیمات، در صورت پشتیبانی سرویس.

## 15. Archive Module

فشرده‌سازی فایل‌های موجود روی هاست مقصد:

```yaml
- name: Create archive
  archive:
    path: /home/TestDir2/
    dest: /tmp/ArcTestDir2.tar.gz
    format: gz
    exclude_path:
      - /home/TestDir2/TestFile2
  tags: archive_mod
```

- `path`: فایل یا مسیری که باید آرشیو شود.
- `dest`: مسیر فایل خروجی.
- `format`: فرمت فشرده‌سازی، مانند `gz`، `bz2`، `xz` یا `zip`.
- `exclude_path`: مسیرهایی که نباید وارد آرشیو شوند.
- `remove: yes`: حذف فایل‌های منبع پس از آرشیو شدن؛ با احتیاط استفاده کن.

## 16. Unarchive Module

استخراج آرشیو:

```yaml
- name: Extract archive
  unarchive:
    src: /tmp/ArcTestDir2.tar.gz
    dest: /home/TestDir2
    remote_src: yes
    keep_newer: yes
  tags: unarchive_mod
```

- `src`: فایل آرشیو.
- `dest`: مسیر استخراج.
- `remote_src: yes`: آرشیو از قبل روی هاست مقصد قرار دارد.
- `keep_newer: yes`: فایل‌های موجود و جدیدتر از نسخه‌های داخل آرشیو حفظ می‌شوند.
- `creates`: در صورت وجود مسیر مشخص‌شده، استخراج انجام نمی‌شود.

اگر فایل آرشیو روی Controller باشد، `remote_src: yes` را حذف کن تا Ansible آن را به هاست مقصد منتقل کند.

