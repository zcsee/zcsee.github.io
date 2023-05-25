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
| **stdin_add_newline** boolean*added in Ansible 2.8*   | If set to `true`, append a newline to stdin data.<br/>**Choices:**` false``true ` ← (default)                                                                                                                                                                                    |
| **strip_empty_ends** boolean*added in Ansible 2.8*    | Strip empty lines from the end of stdout/stderr in result.<br/>**Choices:**` false``true ` ← (default)                                                                                                                                                                           |

### Attributes

| Attribute      | Support                                                                                                                                                     | Description                                                                                                    |
| -------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **check_mode** | **partial**<br/>while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target                            |
| **diff_mode**  | **none**                                                                                                                                                    | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode          |
| **platform**   | **Platform:** **posix**                                                                                                                                     | Target OS/families that can be operated against                                                                |
| **raw**        | **full**                                                                                                                                                    | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

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

| Key                                     | Description                                                                                                                                                      |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **cmd** list / elements=string          | The command executed by the task.<br/>**Returned:** always<br/>**Sample:** `["echo", "hello"]`                                                                   |
| **delta** string                        | The command execution delta time.<br/>**Returned:** always<br/>**Sample:** `"0:00:00.001529"`                                                                    |
| **end** string                          | The command execution end time.<br>**Returned:** always<br/>**Sample:** `"2017-09-29 22:03:48.084657"`                                                           |
| **msg** boolean                         | changed<br/>**Returned:** always<br/>**Sample:** `true`                                                                                                          |
| **rc** integer                          | The command return code (0 means success).<br/>**Returned:** always<br/>**Sample:** `0`                                                                          |
| **start** string                        | The command execution start time.<br/>**Returned:** always<br/>**Sample:** `"2017-09-29 22:03:48.083128"`                                                        |
| **stderr** string                       | The command standard error.<br/>**Returned:** always<br/>**Sample:** `"ls cannot access foo: No such file or directory"`                                         |
| **stderr_lines** list / elements=string | The command standard error split in lines.<br/>**Returned:** always<br/>**Sample:** `[{"u'ls cannot access foo": "No such file or directory'"}, "u'ls \u2026'"]` |
| **stdout** string                       | The command standard output.<br/>**Returned:** always<br/>**Sample:** `"Clustering node rabbit@slave1 with rabbit@master \u2026"`                                |
| **stdout_lines** list / elements=string | The command standard output split in lines.<br/>**Returned:** always<br/>**Sample:** `["u'Clustering node rabbit@slave1 with rabbit@master \u2026'"]`            |

## shell 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)

Parameters

| Parameter                                           | Comments                                                                                                                                                  |
| --------------------------------------------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **chdir** path*added in Ansible 0.6*                | Change into this directory before running the command.                                                                                                    |
| **cmd** string                                      | The command to run followed by optional arguments.                                                                                                        |
| **creates** path                                    | A filename, when it already exists, this step will **not** be run.                                                                                        |
| **executable** path*added in Ansible 0.9*           | Change the shell used to execute the command.This expects an absolute path to the executable.                                                             |
| **free_form** string                                | The shell module takes a free form command to run, as a string.There is no actual parameter named ‘free form’.See the examples on how to use this module. |
| **removes** path*added in Ansible 0.8*              | A filename, when it does not exist, this step will **not** be run.                                                                                        |
| **stdin** string*added in Ansible 2.4*              | Set the stdin of the command directly to the specified value.                                                                                             |
| **stdin_add_newline** boolean*added in Ansible 2.8* | Whether to append a newline to stdin data.**Choices:**` false``true ` ← (default)                                                                         |

Attributes
| Attribute | Support | Description |
| ------ | :------ |:------|
| **check_mode** | **partial**<br/>while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode** | **Support:** **none** | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform** | **Platform:** **posix** | Target OS/families that can be operated against |
| **raw** | **Support:** **full** | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

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

| Key                                     | Description                                                                                                                                                      |
| --------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **cmd** string                          | The command executed by the task.<br>**Returned:** always<br/>**Sample:** `"rabbitmqctl join_cluster rabbit@master"`                                             |
| **delta** string                        | The command execution delta time.<br/>**Returned:** always<br/>**Sample:** `"0:00:00.325771"`                                                                    |
| **end** string                          | The command execution end time.<br/>**Returned:** always<br/>**Sample:** `"2016-02-25 09:18:26.755339"`                                                          |
| **msg** boolean                         | changed<br/>**Returned:** always<br/>**Sample:** `true`                                                                                                          |
| **rc** integer                          | The command return code (0 means success).<br/>**Returned:** always<br/>**Sample:** `0`                                                                          |
| **start** string                        | The command execution start time.<br/>**Returned:** always<br/>**Sample:** `"2016-02-25 09:18:26.429568"`                                                        |
| **stderr** string                       | The command standard error.<br/>**Returned:** always<br/>**Sample:** `"ls: cannot access foo: No such file or directory"`                                        |
| **stderr_lines** list / elements=string | The command standard error split in lines.<br/>**Returned:** always<br/>**Sample:** `[{"u'ls cannot access foo": "No such file or directory'"}, "u'ls \u2026'"]` |
| **stdout** string                       | The command standard output.<br/>**Returned:** always<br/>**Sample:** `"Clustering node rabbit@slave1 with rabbit@master \u2026"`                                |
| **stdout_lines** list / elements=string | The command standard output split in lines.<br/>**Returned:** always<br/>**Sample:** `["u'Clustering node rabbit@slave1 with rabbit@master \u2026'"]`            |

## script 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/script_module.html)

**Parameters**

| Parameter                                   | Comments                                                                                                        |
| ------------------------------------------- | --------------------------------------------------------------------------------------------------------------- |
| **chdir** string*added in Ansible 2.4*      | Change into this directory on the remote node before running the script.                                        |
| **cmd** string                              | Path to the local script to run followed by optional arguments.                                                 |
| **creates** string*added in Ansible 1.5*    | A filename on the remote node, when it already exists, this step will **not** be run.                           |
| **decrypt** boolean*added in Ansible 2.4*   | This option controls the autodecryption of source files using vault.<br>**Choices:**` false``true ` ← (default) |
| **executable** string*added in Ansible 2.6* | Name or path of a executable to invoke the script with.                                                         |
| **free_form** string                        | Path to the local script file followed by optional arguments.                                                   |
| **removes** string*added in Ansible 1.5*    | A filename on the remote node, when it does not exist, this step will **not** be run.                           |

**Attributes**

| Attribute                | Support                                                                                                                                                    | Description                                                                                                    |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------- |
| **check_mode**           | **partial** <br>while the script itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target                            |
| **diff_mode**            | **none**                                                                                                                                                   | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode          |
| **platform**             | **Platforms:** **all**<br>This action is one of the few that requires no Python on the remote as it passes the command directly into the connection string | Target OS/families that can be operated against                                                                |
| **raw**                  | **full**                                                                                                                                                   | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |
| **safe_file_operations** | **none**                                                                                                                                                   | Uses Ansible’s strict file operation functions to ensure proper permissions and avoid data corruption          |
| **vault**                | **full**                                                                                                                                                   | Can automatically decrypt Ansible vaulted files                                                                |

**Examples**

```yaml
- name: Run a script with arguments (free form)
  ansible.builtin.script: /some/local/script.sh --some-argument 1234

- name: Run a script with arguments (using 'cmd' parameter)
  ansible.builtin.script:
    cmd: /some/local/script.sh --some-argument 1234

- name: Run a script only if file.txt does not exist on the remote node
  ansible.builtin.script: /some/local/create_file.sh --some-argument 1234
  args:
    creates: /the/created/file.txt

- name: Run a script only if file.txt exists on the remote node
  ansible.builtin.script: /some/local/remove_file.sh --some-argument 1234
  args:
    removes: /the/removed/file.txt

- name: Run a script using an executable in a non-system path
  ansible.builtin.script: /some/local/script
  args:
    executable: /some/remote/executable

- name: Run a script using an executable in a system path
  ansible.builtin.script: /some/local/script.py
  args:
    executable: python3

- name: Run a Powershell script on a windows host
  script: subdirectories/under/path/with/your/playbook/script.ps1
```

## copy 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html)

**Parameters**

| Parameter                                                | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
| -------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **attributes** aliases: attrstring*added in Ansible 2.3* | The attributes the resulting filesystem object should have.To get supported flags look at the man page for _chattr_ on the target system.This string should contain the attributes in the same order as the one displayed by _lsattr_.The `=` operator is assumed as default, otherwise `+` or `-` operators need to be included in the string.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **backup** boolean*added in Ansible 0.7*                 | Create a backup file including the timestamp information so you can get the original file back if you somehow clobbered it incorrectly. <br/>**Choices:**`false` ← (default)`true`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| **checksum** string*added in Ansible 2.5*                | SHA1 checksum of the file being transferred.Used to validate that the copy of the file was successful.If this is not provided, ansible will use the local calculated checksum of the src file.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **content** string*added in Ansible 1.1*                 | When used instead of `src`, sets the contents of a file directly to the specified value.Works only when `dest` is a file. Creates the file if it does not exist.For advanced formatting or if `content` contains a variable, use the [ansible.builtin.template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html#ansible-collections-ansible-builtin-template-module) module.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **decrypt** boolean*added in Ansible 2.4*                | This option controls the autodecryption of source files using vault. <br/>**Choices:**` false``true ` ← (default)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **dest** path / required                                 | Remote absolute path where the file should be copied to.If `src` is a directory, this must be a directory too.If `dest` is a non-existent path and if either `dest` ends with “/” or `src` is a directory, `dest` is created.If _dest_ is a relative path, the starting directory is determined by the remote host.If `src` and `dest` are files, the parent directory of `dest` is not created and the task fails if it does not already exist.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **directory_mode** any*added in Ansible 1.5*             | When doing a recursive copy set the mode for the directories.If this is not set we will use the system defaults.The mode is only set on directories which are newly created, and will not affect those that already existed.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **follow** boolean*added in Ansible 1.8*                 | This flag indicates that filesystem links in the destination, if they exist, should be followed. <br/>**Choices:**`false` ← (default)`true`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **force** boolean*added in Ansible 1.1*                  | Influence whether the remote file must always be replaced.If `true`, the remote file will be replaced when contents are different than the source.If `false`, the file will only be transferred if the destination does not exist. <br>**Choices:**` false``true ` ← (default)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **group** string                                         | Name of the group that should own the filesystem object, as would be fed to _chown_.When left unspecified, it uses the current group of the current user unless you are root, in which case it can preserve the previous ownership.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| **local_follow** boolean*added in Ansible 2.4*           | This flag indicates that filesystem links in the source tree, if they exist, should be followed.<br>**Choices:**` false``true ` ← (default)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **mode** any                                             | The permissions of the destination file or directory.For those used to `/usr/bin/chmod` remember that modes are actually octal numbers. You must either add a leading zero so that Ansible’s YAML parser knows it is an octal number (like `0644` or `01777`) or quote it (like `'644'` or `'1777'`) so Ansible receives a string and can do its own conversion from string into number. Giving Ansible a number without following one of these rules will end up with a decimal number which will have unexpected results.As of Ansible 1.8, the mode may be specified as a symbolic mode (for example, `u+rwx` or `u=rw,g=r,o=r`).As of Ansible 2.3, the mode may also be the special string `preserve`.`preserve` means that the file will be given the same permissions as the source file.When doing a recursive copy, see also `directory_mode`.If `mode` is not specified and the destination file **does not** exist, the default `umask` on the system will be used when setting the mode for the newly created file.If `mode` is not specified and the destination file **does** exist, the mode of the existing file will be used.Specifying `mode` is the best way to ensure files are created with the correct permissions. See CVE-2020-1736 for further details. |
| **owner** string                                         | Name of the user that should own the filesystem object, as would be fed to _chown_.When left unspecified, it uses the current user unless you are root, in which case it can preserve the previous ownership.Specifying a numeric username will be assumed to be a user ID and not a username. Avoid numeric usernames to avoid this confusion.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **remote_src** boolean*added in Ansible 2.0*             | Influence whether `src` needs to be transferred or already is present remotely.If `false`, it will search for `src` on the controller node.If `true` it will search for `src` on the managed (remote) node.`remote_src` supports recursive copying as of version 2.8.`remote_src` only works with `mode=preserve` as of version 2.6.Autodecryption of files does not work when `remote_src=yes`. <br/>**Choices:**`false` ← (default) <br>`true`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **selevel** string                                       | The level part of the SELinux filesystem object context.This is the MLS/MCS attribute, sometimes known as the `range`.When set to `_default`, it will use the `level` portion of the policy if available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **serole** string                                        | The role part of the SELinux filesystem object context.When set to `_default`, it will use the `role` portion of the policy if available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **setype** string                                        | The type part of the SELinux filesystem object context.When set to `_default`, it will use the `type` portion of the policy if available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **seuser** string                                        | The user part of the SELinux filesystem object context.By default it uses the `system` policy, where applicable.When set to `_default`, it will use the `user` portion of the policy if available.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| **src** path                                             | Local path to a file to copy to the remote server.This can be absolute or relative.If path is a directory, it is copied recursively. In this case, if path ends with “/”, only inside contents of that directory are copied to destination. Otherwise, if it does not end with “/”, the directory itself with all contents is copied. This behavior is similar to the `rsync` command line tool.                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **unsafe_writes** boolean*added in Ansible 2.2*          | Influence when to use atomic operation to prevent data corruption or inconsistent reads from the target filesystem object.By default this module uses atomic operations to prevent data corruption or inconsistent reads from the target filesystem objects, but sometimes systems are configured or just broken in ways that prevent this. One example is docker mounted filesystem objects, which cannot be updated atomically from inside the container and can only be written in an unsafe manner.This option allows Ansible to fall back to unsafe methods of updating filesystem objects when atomic operations fail (however, it doesn’t force Ansible to perform unsafe writes).IMPORTANT! Unsafe writes are subject to race conditions and can lead to data corruption. <br/>**Choices:**`false` ← (default)`true`                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **validate** string                                      | The validation command to run before copying the updated file into the final destination.A temporary file path is used to validate, passed in through ‘%s’ which must be present as in the examples below.Also, the command is passed securely so shell features such as expansion and pipes will not work.For an example on how to handle more complex validation than what this option provides, see [handling complex validation](https://docs.ansible.com/ansible/latest/reference_appendices/faq.html#complex-configuration-validation).                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |

**Attributes**

| Attribute                | Support                          | Description                                                                                                                                                                                                                                                                                                             |
| ------------------------ | -------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **action**               | **full**                         | Indicates this has a corresponding action plugin so some parts of the options can be executed on the controller                                                                                                                                                                                                         |
| **async**                | **none**                         | Supports being used with the `async` keyword                                                                                                                                                                                                                                                                            |
| **bypass_host_loop**     | **none**                         | Forces a ‘global’ task that does not execute per host, this bypasses per host templating and serial, throttle and other loop considerationsConditionals will work as if `run_once` is being used, variables used will be from the first available hostThis action will not work normally outside of lockstep strategies |
| **check_mode**           | **full**                         | Can run in check_mode and return changed status prediction without modifying target                                                                                                                                                                                                                                     |
| **diff_mode**            | **full**                         | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode                                                                                                                                                                                                                   |
| **platform**             | **Platform:** **posix**          | Target OS/families that can be operated against                                                                                                                                                                                                                                                                         |
| **safe_file_operations** | **full**                         | Uses Ansible’s strict file operation functions to ensure proper permissions and avoid data corruption                                                                                                                                                                                                                   |
| **vault**                | **full\***added in Ansible 2.2\* | Can automatically decrypt Ansible vaulted files                                                                                                                                                                                                                                                                         |

**Examples**

```yaml
- name: Copy file with owner and permissions
  ansible.builtin.copy:
    src: /srv/myfiles/foo.conf
    dest: /etc/foo.conf
    owner: foo
    group: foo
    mode: "0644"

- name: Copy file with owner and permission, using symbolic representation
  ansible.builtin.copy:
    src: /srv/myfiles/foo.conf
    dest: /etc/foo.conf
    owner: foo
    group: foo
    mode: u=rw,g=r,o=r

- name: Another symbolic mode example, adding some permissions and removing others
  ansible.builtin.copy:
    src: /srv/myfiles/foo.conf
    dest: /etc/foo.conf
    owner: foo
    group: foo
    mode: u+rw,g-wx,o-rwx

- name: Copy a new "ntp.conf" file into place, backing up the original if it differs from the copied version
  ansible.builtin.copy:
    src: /mine/ntp.conf
    dest: /etc/ntp.conf
    owner: root
    group: root
    mode: "0644"
    backup: yes

- name: Copy a new "sudoers" file into place, after passing validation with visudo
  ansible.builtin.copy:
    src: /mine/sudoers
    dest: /etc/sudoers
    validate: /usr/sbin/visudo -csf %s

- name: Copy a "sudoers" file on the remote machine for editing
  ansible.builtin.copy:
    src: /etc/sudoers
    dest: /etc/sudoers.edit
    remote_src: yes
    validate: /usr/sbin/visudo -csf %s

- name: Copy using inline content
  ansible.builtin.copy:
    content: "# This file was moved to /etc/other.conf"
    dest: /etc/mine.conf

- name: If follow=yes, /path/to/file will be overwritten by contents of foo.conf
  ansible.builtin.copy:
    src: /etc/foo.conf
    dest: /path/to/link # link to /path/to/file
    follow: yes

- name: If follow=no, /path/to/link will become a file and be overwritten by contents of foo.conf
  ansible.builtin.copy:
    src: /etc/foo.conf
    dest: /path/to/link # link to /path/to/file
    follow: no
```

**Return Values**

| Key                    | Description                                                                                                                                                                    |
| ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **backup_file** string | Name of backup file created. <br>**Returned:** changed and if backup=yes <br/>**Sample:** `"/path/to/file.txt.2015-02-12@22:09~"`                                              |
| **checksum** string    | SHA1 checksum of the file after running copy. <br/>**Returned:** success <br/>**Sample:** `"6e642bb8dd5c2e027bf21dd923337cbb4214f827"`                                         |
| **dest** string        | Destination file/path. <br/>**Returned:** success <br/>**Sample:** `"/path/to/file.txt"`                                                                                       |
| **gid** integer        | Group id of the file, after execution. <br/>**Returned:** success <br/>**Sample:** `100`                                                                                       |
| **group** string       | Group of the file, after execution. <br/>**Returned:** success <br/>**Sample:** `"httpd"`                                                                                      |
| **md5sum** string      | MD5 checksum of the file after running copy. <br/>**Returned:** when supported <br/>**Sample:** `"2a5aeecc61dc98c4d780b14b330e3282"`                                           |
| **mode** string        | Permissions of the target, after execution. <br/>**Returned:** success <br/>**Sample:** `"0644"`                                                                               |
| **owner** string       | Owner of the file, after execution. <br/>**Returned:** success <br/>**Sample:** `"httpd"`                                                                                      |
| **size** integer       | Size of the target, after execution. <br/>**Returned:** success <br/>**Sample:** `1220`                                                                                        |
| **src** string         | Source file used for the copy on the target machine. <br/>**Returned:** changed <br/>**Sample:** `"/home/httpd/.ansible/tmp/ansible-tmp-1423796390.97-147729857856000/source"` |
| **state** string       | State of the target, after execution. <br/>**Returned:** success <br/>**Sample:** `"file"`                                                                                     |
| **uid** integer        | Owner id of the file, after execution. <br/>**Returned:** success <br/>**Sample:** `100`                                                                                       |

## fetch 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/fetch_module.html)

**Parameters**

| Parameter                                           | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                            |
| --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **dest** string / required                          | A directory to save the file into.For example, if the _dest_ directory is `/backup` a _src_ file named `/etc/profile` on host `host.example.com`, would be saved into `/backup/host.example.com/etc/profile`. The host name is based on the inventory name.                                                                                                                                                                                         |
| **fail_on_missing** boolean*added in Ansible 1.1*   | When set to `true`, the task will fail if the remote file cannot be read for any reason.Prior to Ansible 2.5, setting this would only fail if the source file was missing.The default was changed to `true` in Ansible 2.5. <br>**Choices:**` false``true ` ← (default)                                                                                                                                                                             |
| **flat** boolean*added in Ansible 1.2*              | Allows you to override the default behavior of appending hostname/path/to/file to the destination.If `dest` ends with ‘/’, it will use the basename of the source file, similar to the copy module.This can be useful if working with a single host, or if retrieving files that are uniquely named per host.If using multiple hosts with the same filename, the file will be overwritten for each host. <br/>**Choices:**`false` ← (default)`true` |
| **src** string / required                           | The file on the remote system to fetch.This _must_ be a file, not a directory.Recursive fetching may be supported in a later release.                                                                                                                                                                                                                                                                                                               |
| **validate_checksum** boolean*added in Ansible 1.4* | Verify that the source and destination checksums match after the files are fetched. <br/>**Choices:**` false``true ` ← (default)                                                                                                                                                                                                                                                                                                                    |

**Attributes**

| Attribute                | Support                               | Description                                                                                                                                                                                                                                                                                                             |
| ------------------------ | ------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **action**               | **full**                              | Indicates this has a corresponding action plugin so some parts of the options can be executed on the controller                                                                                                                                                                                                         |
| **async**                | **none**                              | Supports being used with the `async` keyword                                                                                                                                                                                                                                                                            |
| **bypass_host_loop**     | **none**                              | Forces a ‘global’ task that does not execute per host, this bypasses per host templating and serial, throttle and other loop considerationsConditionals will work as if `run_once` is being used, variables used will be from the first available hostThis action will not work normally outside of lockstep strategies |
| **check_mode**           | **full**                              | Can run in check_mode and return changed status prediction without modifying target                                                                                                                                                                                                                                     |
| **diff_mode**            | **full**                              | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode                                                                                                                                                                                                                   |
| **platform**             | **Platforms:** **posix**, **windows** | Target OS/families that can be operated against                                                                                                                                                                                                                                                                         |
| **safe_file_operations** | **none**                              | Uses Ansible’s strict file operation functions to ensure proper permissions and avoid data corruption                                                                                                                                                                                                                   |
| **vault**                | **none**                              | Can automatically decrypt Ansible vaulted files                                                                                                                                                                                                                                                                         |

**Examples**

```yaml
- name: Store file into /tmp/fetched/host.example.com/tmp/somefile
  ansible.builtin.fetch:
    src: /tmp/somefile
    dest: /tmp/fetched

- name: Specifying a path directly
  ansible.builtin.fetch:
    src: /tmp/somefile
    dest: /tmp/prefix-{{ inventory_hostname }}
    flat: yes

- name: Specifying a destination path
  ansible.builtin.fetch:
    src: /tmp/uniquefile
    dest: /tmp/special/
    flat: yes

- name: Storing in a path relative to the playbook
  ansible.builtin.fetch:
    src: /tmp/uniquefile
    dest: special/prefix-{{ inventory_hostname }}
    flat: yes
```

## file 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html)

**Parameters**

**Attributes**

**Examples**

**Return Values**

## unarchive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)

Parameters

Attributes

Examples

Return Values

## archive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)

Parameters

Attributes

Examples

Return Values

## hostname 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/hostname_module.html)

Parameters

Attributes

Examples

Return Values

## cron 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html)

Parameters

Attributes

Examples

Return Values

## yum_repository 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)

Parameters

Attributes

Examples

Return Values

## dnf 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/dnf_module.html)

Parameters

Attributes

Examples

Return Values

## service 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/service_module.html)

Parameters

Attributes

Examples

Return Values

## firewalld 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/posix/firewalld_module.html)

Parameters

Attributes

Examples

Return Values

## user 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/user_module.html)

Parameters

Attributes

Examples

Return Values

## group 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/group_module.html)

Parameters

Attributes

Examples

Return Values

## lineinfile 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/lineinfile_module.html)

Parameters

Attributes

Examples

Return Values

## replace 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/replace_module.html)

Parameters

Attributes

Examples

Return Values

## setup 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/setup_module.html)

Parameters

Attributes

Examples

Return Values

## debug 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/debug_module.html)

Parameters

Attributes

Examples

Return Values
