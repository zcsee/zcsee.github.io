---
layout: post
title: 'ansible-playbook demo'
subtitle: 'ansible_常用模块示例'
date: 2023-05-24
categories: linux
author: Jason
cover: 'assets/img/profile.png'
tags: ansible playbook demo
---

## command模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html#ansible-collections-ansible-builtin-command-module)

### Parameters

| Parameter                                             | Comments                                                     |
| ----------------------------------------------------- | :----------------------------------------------------------- |
| **argv** list / elements=string*added in Ansible 2.6* | Passes the command as a list rather than a string.Use `argv` to avoid quoting values that would otherwise be interpreted incorrectly (for example “user name”).Only the string (free form) or the list (argv) form can be provided, not both. One or the other must be provided. |
| **chdir** path*added in Ansible 0.6*                  | Change into this directory before running the command.       |
| **cmd** string                                        | The command to run.                                          |
| **creates** path                                      | A filename or (since 2.0) glob pattern. If a matching file already exists, this step **will not** be run.This is checked before *removes* is checked. |
| **free_form** string                                  | The command module takes a free form string as a command to run.There is no actual parameter named ‘free form’. |
| **removes** path*added in Ansible 0.8*                | A filename or (since 2.0) glob pattern. If a matching file exists, this step **will** be run.This is checked after *creates* is checked. |
| **stdin** string*added in Ansible 2.4*                | Set the stdin of the command directly to the specified value. |
| **stdin_add_newline** boolean*added in Ansible 2.8*   | If set to `true`, append a newline to stdin data.**Choices:**`false``true` ← (default) |
| **strip_empty_ends** boolean*added in Ansible 2.8*    | Strip empty lines from the end of stdout/stderr in result.**Choices:**`false``true` ← (default) |

### Attributes

| Attribute      | Support                                                      | Description                                                  |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **check_mode** | **partial**while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**  | **none**                                                     | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform**   | **Platform:** **posix**                                      | Target OS/families that can be operated against              |
| **raw**        | **full**                                                     | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

### Examples

```
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

| Key                                     | Description                                                  |
| --------------------------------------- | ------------------------------------------------------------ |
| **cmd** list / elements=string          | The command executed by the task.**Returned:** always**Sample:** `["echo", "hello"]` |
| **delta** string                        | The command execution delta time.**Returned:** always**Sample:** `"0:00:00.001529"` |
| **end** string                          | The command execution end time.**Returned:** always**Sample:** `"2017-09-29 22:03:48.084657"` |
| **msg** boolean                         | changed**Returned:** always**Sample:** `true`                |
| **rc** integer                          | The command return code (0 means success).**Returned:** always**Sample:** `0` |
| **start** string                        | The command execution start time.**Returned:** always**Sample:** `"2017-09-29 22:03:48.083128"` |
| **stderr** string                       | The command standard error.**Returned:** always**Sample:** `"ls cannot access foo: No such file or directory"` |
| **stderr_lines** list / elements=string | The command standard error split in lines.**Returned:** always**Sample:** `[{"u'ls cannot access foo": "No such file or directory'"}, "u'ls \u2026'"]` |
| **stdout** string                       | The command standard output.**Returned:** always**Sample:** `"Clustering node rabbit@slave1 with rabbit@master \u2026"` |
| **stdout_lines** list / elements=string | The command standard output split in lines.**Returned:** always**Sample:** `["u'Clustering node rabbit@slave1 with rabbit@master \u2026'"]` |

## shell模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)

## script模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html)

## copy模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)

## fetch模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html)

## file模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)

## unarchive模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)

## archive模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)

## hostname模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/hostname_module.html)

## cron模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html)

## yum_repository模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)

## dnf模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)

## service模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)

## firewalld模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)

## user模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)

## group模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/group_module.html)

## lineinfile模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html)

## replace模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/replace_module.html)

## setup模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

## debug模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)

