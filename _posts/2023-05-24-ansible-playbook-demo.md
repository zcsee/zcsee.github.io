---
layout: post
title: "ansible-playbook demo"
subtitle: "ansible_常用模块示例"
date: 2023-05-24
categories: linux
author: Jason
cover: "assets/img/profile.png"
tags: ansible playbook demo
---

## command 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html#ansible-collections-ansible-builtin-command-module)

### Parameters

| Parameter                                             | Comments                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **argv** list / elements=string*added in Ansible 2.6* | Passes the command as a list rather than a string.Use `argv` to avoid quoting values that would otherwise be interpreted incorrectly (for example “user name”).Only the string (free form) or the list (argv) form can be provided, not both. One or the other must be provided. |
| **chdir** path*added in Ansible 0.6*                  | Change into this directory before running the command.                                                                                                                                                                                                                           |
| **cmd** string                                        | The command to run.                                                                                                                                                                                                                                                              |
| **creates** path                                      | A filename or (since 2.0) glob pattern. If a matching file already exists, this step **will not** be run.This is checked before _removes_ is checked.                                                                                                                            |
| **free_form** string                                  | The command module takes a free form string as a command to run.There is no actual parameter named ‘free form’.                                                                                                                                                                  |
| **removes** path*added in Ansible 0.8*                | A filename or (since 2.0) glob pattern. If a matching file exists, this step **will** be run.This is checked after _creates_ is checked.                                                                                                                                         |
| **stdin** string*added in Ansible 2.4*                | Set the stdin of the command directly to the specified value.                                                                                                                                                                                                                    |
| **stdin_add_newline** boolean*added in Ansible 2.8*   | If set to `true`, append a newline to stdin data.**Choices:**` false``true ` ← (default)                                                                                                                                                                                         |
| **strip_empty_ends** boolean*added in Ansible 2.8*    | Strip empty lines from the end of stdout/stderr in result.**Choices:**` false``true ` ← (default)                                                                                                                                                                                |

### Attributes

| Attribute      | Support                                                                                                                                                | Description                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| **check_mode** | **partial**while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target                            |
| **diff_mode**  | **none**                                                                                                                                               | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode          |
| **platform**   | **Platform:** **posix**                                                                                                                                | Target OS/families that can be operated against                                                                |
| **raw**        | **full**                                                                                                                                               | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

### Examples

```yaml
- name: Return motd to registered var
  ansible.builtin.command: cat /etc/motd
  register: mymotd

# free-form (string) arguments, all arguments on one line
- name: Run command if /path/to/database does not exist (without 'args')
  ansible.builtin.command: /usr/bin/make_database.sh db_user db_name creates=/path/to/database

# free-form (string) arguments, some arguments on separate lines with the 'args' keyword
# 'args' is a task keyword, passed at the same level as the module
- name: Run command if /path/to/database does not exist (with 'args' keyword)
  ansible.builtin.command: /usr/bin/make_database.sh db_user db_name
  args:
    creates: /path/to/database

# 'cmd' is module parameter
- name: Run command if /path/to/database does not exist (with 'cmd' parameter)
  ansible.builtin.command:
    cmd: /usr/bin/make_database.sh db_user db_name
    creates: /path/to/database

- name: Change the working directory to somedir/ and run the command as db_owner if /path/to/database does not exist
  ansible.builtin.command: /usr/bin/make_database.sh db_user db_name
  become: yes
  become_user: db_owner
  args:
    chdir: somedir/
    creates: /path/to/database

# argv (list) arguments, each argument on a separate line, 'args' keyword not necessary
# 'argv' is a parameter, indented one level from the module
- name: Use 'argv' to send a command as a list - leave 'command' empty
  ansible.builtin.command:
    argv:
      - /usr/bin/make_database.sh
      - Username with whitespace
      - dbname with whitespace
    creates: /path/to/database

- name: Safely use templated variable to run command. Always use the quote filter to avoid injection issues
  ansible.builtin.command: cat {{ myfile|quote }}
  register: myoutput
```

### Return Values

| Key                                     | Description                                                                                                                                            |
| --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **cmd** list / elements=string          | The command executed by the task.**Returned:** always**Sample:** `["echo", "hello"]`                                                                   |
| **delta** string                        | The command execution delta time.**Returned:** always**Sample:** `"0:00:00.001529"`                                                                    |
| **end** string                          | The command execution end time.**Returned:** always**Sample:** `"2017-09-29 22:03:48.084657"`                                                          |
| **msg** boolean                         | changed**Returned:** always**Sample:** `true`                                                                                                          |
| **rc** integer                          | The command return code (0 means success).**Returned:** always**Sample:** `0`                                                                          |
| **start** string                        | The command execution start time.**Returned:** always**Sample:** `"2017-09-29 22:03:48.083128"`                                                        |
| **stderr** string                       | The command standard error.**Returned:** always**Sample:** `"ls cannot access foo: No such file or directory"`                                         |
| **stderr_lines** list / elements=string | The command standard error split in lines.**Returned:** always**Sample:** `[{"u'ls cannot access foo": "No such file or directory'"}, "u'ls \u2026'"]` |
| **stdout** string                       | The command standard output.**Returned:** always**Sample:** `"Clustering node rabbit@slave1 with rabbit@master \u2026"`                                |
| **stdout_lines** list / elements=string | The command standard output split in lines.**Returned:** always**Sample:** `["u'Clustering node rabbit@slave1 with rabbit@master \u2026'"]`            |

## shell 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)

Parameters

| Parameter                                             | Comments                                                                                                                                                                                                                                                                         |
| ----------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **chdir** path*added in Ansible 0.6*               | Change into this directory before running the command.                                                                                                    |
| **cmd** string                                      | The command to run followed by optional arguments.                                                                                                        |
| **creates** path                                    | A filename, when it already exists, this step will **not** be run.                                                                                        |
| **executable** path*added in Ansible 0.9*           | Change the shell used to execute the command.This expects an absolute path to the executable.                                                             |
| **free_form** string                                | The shell module takes a free form command to run, as a string.There is no actual parameter named ‘free form’.See the examples on how to use this module. |
| **removes** path*added in Ansible 0.8*              | A filename, when it does not exist, this step will **not** be run.                                                                                        |
| **stdin** string*added in Ansible 2.4*              | Set the stdin of the command directly to the specified value.                                                                                             |
| **stdin_add_newline** boolean*added in Ansible 2.8* | Whether to append a newline to stdin data.**Choices:**` false``true ` ← (default)                                                                         |

Attributes
| Attribute      | Support                                                                                                                                                | Description                                                                                                    |
| -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------- |
| **check_mode** | **Support:** **partial**while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target                            |
| **diff_mode**   | **Support:** **none**                                                                                                                                               | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode          |
| **platform**    | **Platform:** **posix**                                                                                                                                             | Target OS/families that can be operated against                                                                |
| **raw**         | **Support:** **full**                                                                                                                                               | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

Examples

```yaml
- name: Execute the command in remote shell; stdout goes to the specified file on the remote
  ansible.builtin.shell: somescript.sh >> somelog.txt

- name: Change the working directory to somedir/ before executing the command
  ansible.builtin.shell: somescript.sh >> somelog.txt
  args:
    chdir: somedir/

# You can also use the 'args' form to provide the options.
- name: This command will change the working directory to somedir/ and will only run when somedir/somelog.txt doesn't exist
  ansible.builtin.shell: somescript.sh >> somelog.txt
  args:
    chdir: somedir/
    creates: somelog.txt

# You can also use the 'cmd' parameter instead of free form format.
- name: This command will change the working directory to somedir/
  ansible.builtin.shell:
    cmd: ls -l | grep log
    chdir: somedir/

- name: Run a command that uses non-posix shell-isms (in this example /bin/sh doesn't handle redirection and wildcards together but bash does)
  ansible.builtin.shell: cat < /tmp/*txt
  args:
    executable: /bin/bash

- name: Run a command using a templated variable (always use quote filter to avoid injection)
  ansible.builtin.shell: cat {{ myfile|quote }}

# You can use shell to run other executables to perform actions inline
- name: Run expect to wait for a successful PXE boot via out-of-band CIMC
  ansible.builtin.shell: |
    set timeout 300
    spawn ssh admin@{{ cimc_host }}

    expect "password:"
    send "{{ cimc_password }}\n"

    expect "\n{{ cimc_name }}"
    send "connect host\n"

    expect "pxeboot.n12"
    send "\n"

    exit 0
  args:
    executable: /usr/bin/expect
  delegate_to: localhost
```

Return Values

## script 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html)

## copy 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)

## fetch 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html)

## file 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)

## unarchive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)

## archive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)

## hostname 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/hostname_module.html)

## cron 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html)

## yum_repository 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)

## dnf 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)

## service 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)

## firewalld 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)

## user 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)

## group 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/group_module.html)

## lineinfile 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html)

## replace 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/replace_module.html)

## setup 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

## debug 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)
