# Ansible Notes — MySQL User, Conditions, Pause & Loops

## 1. Add MySQL User

ابتدا Collection موردنیاز را نصب می‌کنیم:

```bash
ansible-galaxy collection install community.mysql
```

ساخت کاربر و دادن دسترسی به دیتابیس `mydb`:

```yaml
- name: "Add mysql user abbas"
  community.mysql.mysql_user:
    name: abbas
    password: "12345"
    priv: 'mydb.*:ALL'
    state: present
    host: '%'
    login_unix_socket: /run/mysqld/mysqld.sock
  tags: add
```

بررسی کاربران در MariaDB:

```sql
SELECT User, Host FROM mysql.user;
```

**نکته:** مقدار `host: '%'` اجازه‌ی اتصال از هر میزبان را می‌دهد؛ فقط در صورت نیاز استفاده کنید. رمز عبور نمونه را در محیط واقعی با رمز قوی جایگزین کنید. همچنین ماژول به کتابخانه‌ی پایتون سازگار با MySQL/MariaDB روی میزبان مقصد نیاز دارد.

## 2. Delegate Task to Ansible Controller

با `delegate_to: localhost` تسک روی سیستم کنترل‌کننده‌ی Ansible اجرا می‌شود، نه روی سرور مقصد.

```yaml
- name: "Create file on Ansible controller"
  file:
    path: /home/myfile
    state: touch
  delegate_to: localhost
  tags: test
```

## 3. Conditions — بررسی سیستم‌عامل

اجرای تسک بر اساس سیستم‌عامل:

```yaml
- name: "HELLO Alpine"
  debug:
    msg: "hello alpine"
  when: ansible_os_family == "Alpine"
  tags: cond_1

- name: "HELLO Suse"
  debug:
    msg: "hello Suse"
  when: ansible_os_family == "Suse"
  tags: cond_1
```

**نکته:** مقدار `ansible_os_family` برای Alpine معمولاً `Alpine` نیست؛ برای انتخاب Alpine بهتر است از `ansible_distribution == "Alpine"` استفاده کنید. برای خانواده‌ی SUSE معمولاً مقدار `Suse` است.

## 4. Check Whether a Package Is Installed

ابتدا اطلاعات پکیج‌های نصب‌شده را جمع‌آوری می‌کنیم:

```yaml
- name: "Gather package facts"
  package_facts:
    manager: auto
  tags: cond_1
```

بررسی نصب بودن `httpd`:

```yaml
- name: "Check if httpd is installed"
  debug:
    var: ansible_facts.packages['httpd']
  when: "'httpd' in ansible_facts.packages"
  tags: cond_1
```

بررسی نصب بودن MariaDB:

```yaml
- name: "Check if MariaDB is installed"
  debug:
    msg: "MariaDB is installed!"
  when: "'mariadb' in ansible_facts.packages"
  tags: cond_1
```

**نکته:** نام پکیج در سیستم‌های مختلف ممکن است متفاوت باشد؛ مثلاً `mariadb-server` یا `MariaDB-server`. شرط باید با نام واقعی پکیج نصب‌شده مطابقت داشته باشد.

## 5. Pause — دریافت ورودی از کاربر

دریافت نام کاربر:

```yaml
- name: "Pause"
  pause:
    prompt: "Enter Your Name:"
  register: username
  tags: p

- name: "Show username"
  debug:
    msg: "{{ username.user_input }}"
  tags: p
```

دریافت مسیر بکاپ، در صورتی که متغیر قبلاً تعریف نشده باشد:

```yaml
- name: "Enter backup path"
  pause:
    prompt: "Please enter the backup path"
  register: back
  when: backup_path is not defined
  tags: cond1
```

**نکته:** برای بررسی تعریف‌شدن متغیر، از `backup_path is not defined` استفاده می‌کنیم. در این مثال ورودی در `back.user_input` قرار می‌گیرد؛ برای استفاده از آن می‌توان متغیر را با `set_fact` ذخیره کرد.

## 6. Install Nginx on CentOS 7

```yaml
- name: "Install nginx on CentOS 7"
  yum:
    name: nginx
    state: present
  when:
    - ansible_distribution == "CentOS"
    - ansible_distribution_major_version == "7"
  tags: cond_6
```

این تسک فقط روی CentOS نسخه‌ی 7 اجرا می‌شود. `yum` برای این مثال قدیمی مناسب است؛ در سیستم‌های جدیدتر می‌توان از `package` استفاده کرد.

## 7. Check Whether nginx-mod-mail Is Installed

روش ساده‌تر با `package_facts`:

```yaml
- name: "Gather package facts"
  package_facts:
    manager: auto
  tags: cond_6

- name: "Show result"
  debug:
    msg: "nginx-mod-mail is installed"
  when: "'nginx-mod-mail' in ansible_facts.packages"
  tags: cond_6
```

با این روش نیازی به اجرای `rpm -qa | grep` نیست.

## 8. Read a File and Check Its Content

خواندن فایل:

```yaml
- name: "cat file"
  shell: cat /home/text.txt
  register: catty
  changed_when: false
  tags: cond1
```

نمایش خروجی:

```yaml
- name: "Show catty"
  debug:
    msg: "{{ catty.stdout }}"
  tags: cond1
```

بررسی وجود کلمه‌ی `Ansible` در فایل:

```yaml
- name: "Check file"
  debug:
    msg: "The text file contains the word Ansible"
  when: "'Ansible' in catty.stdout"
  tags: cond1
```

**نکته:** برای این بررسی ساده، عبارت `'Ansible' in catty.stdout` خواناتر از `find()` است. اگر فقط محتوا را می‌خوانیم، `changed_when: false` مانع گزارش تغییر غیرضروری می‌شود.

## 9. Condition with OR

اگر حداقل یکی از دو متغیر `var1` یا `var2` مقدار درست داشته باشد:

```yaml
- name: "If var1 or var2 is true"
  shell: echo "var1"
  when: (var1 | bool) or (var2 | bool)
  tags: cond1
```

## 10. Loop — Create Multiple Files

با `loop` می‌توان چند فایل را با یک تسک ایجاد کرد:

```yaml
- name: "Touch files"
  file:
    path: "/home/{{ item }}"
    state: touch
    owner: root
    group: root
  loop:
    - a1
    - a2
    - a3
  tags: loop
```

این تسک فایل‌های زیر را ایجاد می‌کند:

- `/home/a1`
- `/home/a2`
- `/home/a3`

اگر فایل‌ها از قبل وجود داشته باشند، `state: touch` زمان دسترسی و تغییر آن‌ها را به‌روزرسانی می‌کند.
