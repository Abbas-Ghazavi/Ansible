
## 1. Skip Tags

```bash
ansible-playbook -i inventory/anisahosts myproject.yml --skip-tags "create_dir2,create_file2"
```

اجرای Playbook بدون اجرای Taskهایی که Tag مشخص‌شده دارند.

## 2. Useful Playbook Options

```bash
# Check syntax
ansible-playbook -i inventory/hosts myproject.yml --syntax-check

# Test without making changes
ansible-playbook -i inventory/hosts myproject.yml --check

# List tasks
ansible-playbook -i inventory/hosts myproject.yml --list-tasks

# List tags
ansible-playbook -i inventory/hosts myproject.yml --list-tags

# Run step by step
ansible-playbook -i inventory/hosts myproject.yml --step

# Show changes (before/after)
ansible-playbook -i inventory/hosts myproject.yml --diff

# Skip a tag
ansible-playbook -i inventory/hosts myproject.yml --skip-tags "create_dir2"
```

## 3. List Tasks Output

```text
playbook: myproject.yml

play #1 (myservers): myservers
  tasks:
    Myprj : Create Dir1 In /home      TAGS: [create_dir1]
    Myprj : Create Dir2 In /home      TAGS: [create_dir2]
    Myprj : Create File1 In /home/dir1 TAGS: [create_file1]
    Myprj : Create File2 In /home/dir2 TAGS: [create_file2]
```

## 4. Extra Variables

```bash
ansible-playbook -i inventory/hosts myproject.yml -e "myvar=123"
```

ارسال متغیر هنگام اجرای Playbook.

## 5. Set Variable & Debug

```yaml
- name: Set variable
  set_fact:
    myvar: 123
  tags: debug_mod

- name: Show my var
  debug:
    var: myvar
  tags: debug_mod
```

- `set_fact`: تعریف متغیر هنگام اجرا.
- `debug`: نمایش مقدار متغیر.

## 6. Variable Names

نام متغیر نباید شامل موارد زیر باشد:

- فاصله
- نقطه (`.`)
- خط تیره (`-`)
- شروع با عدد
- نام‌های رزروشده یا نام‌های دارای تداخل

مثال:

```yaml
mysql_port: 3306
user_name: Amir
```

نکته: متغیرهایی که با `_` شروع می‌شوند، در Ansible به‌صورت عادی قابل استفاده‌اند و الزاماً خصوصی نیستند.

## 7. Variables in vars/main.yml

```yaml
user_list:
  - Amir
  - Abbas

user_list2: [Reza, Iman]
```

هر دو متغیر از نوع List هستند.

## 8. Command Module

```yaml
- name: Command module
  command: cat /home/dir1/file1
  register: catty
  tags: command_mod

- name: Show catty
  debug:
    msg: "{{ catty.stdout }}"
  tags: command_mod
```

- `command`: اجرای دستور.
- `register`: ذخیره خروجی دستور.
- `catty.stdout`: نمایش خروجی دستور.

### Using chdir and creates

```yaml
- name: Command module
  command: cat file1
  args:
    chdir: /home/dir1
    creates: /home/dir4
  register: catty
  tags: command_mod
```

- `chdir`: اجرای دستور در مسیر `/home/dir1`.
- `creates`: اگر `/home/dir4` وجود داشته باشد، دستور اجرا نمی‌شود؛ اگر وجود نداشته باشد، اجرا می‌شود.
- `removes`: اگر تعریف شود، دستور فقط زمانی اجرا می‌شود که مسیر مشخص‌شده وجود داشته باشد.

## 9. Expect Module

```yaml
- name: Expect module
  expect:
    command: passwd LPI
    responses:
      password: "Anisa"
  no_log: true
```

- `expect`: پاسخ‌گویی به ورودی‌های تعاملی.
- `responses`: پاسخ به پیام‌های دستور.
- `no_log: true`: جلوگیری از ثبت اطلاعات حساس در لاگ.

نکته: ماژول `expect` به کتابخانه Python به نام `pexpect` روی هاست مقصد نیاز دارد.
