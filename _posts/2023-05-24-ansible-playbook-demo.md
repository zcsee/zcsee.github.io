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

### Parameters

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

### Attributes

| Attribute | Support | Description |
| ------ | :------ |:------|
| **check_mode** | **partial**<br/>while the command itself is arbitrary and cannot be subject to the check mode semantics it adds `creates`/`removes` options as a workaround | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode** | **Support:** **none** | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform** | **Platform:** **posix** | Target OS/families that can be operated against |
| **raw** | **Support:** **full** | Indicates if an action takes a ‘raw’ or ‘free form’ string as an option and has it’s own special parsing of it |

### Examples

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

### Return Values

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

| Parameter                                                 | Comments                                                     |
| --------------------------------------------------------- | ------------------------------------------------------------ |
| **access_time** string*added in Ansible 2.7*              | This parameter indicates the time the file’s access time should be set to.Should be `preserve` when no modification is required, `YYYYMMDDHHMM.SS` when using default time format, or `now`. <br>Default is `None` meaning that `preserve` is the default for `state=[file,directory,link,hard]` and `now` is default for `state=touch`. |
| **access_time_format** string*added in Ansible 2.7*       | When used with `access_time`, indicates the time format that must be used.Based on default Python format (see time.strftime doc).<br>**Default:** `"%Y%m%d%H%M.%S"` |
| **attributes** aliases: attrstring*added in Ansible 2.3*  | The attributes the resulting filesystem object should have.To get supported flags look at the man page for *chattr* on the target system.This string should contain the attributes in the same order as the one displayed by *lsattr*.The `=` operator is assumed as default, otherwise `+` or `-` operators need to be included in the string. |
| **follow** boolean*added in Ansible 1.8*                  | This flag indicates that filesystem links, if they exist, should be followed.*follow=yes* and *state=link* can modify *src* when combined with parameters such as *mode*.Previous to Ansible 2.5, this was `false` by default.<br>**Choices:**`false`<br>           `true` ← (default) |
| **force** boolean                                         | Force the creation of the symlinks in two cases: the source file does not exist (but will appear later); the destination exists and is a file (so, we need to unlink the `path` file and create symlink to the `src` file in place of it).<br>**Choices:**`false` ← (default)`true` |
| **group** string                                          | Name of the group that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current group of the current user unless you are root, in which case it can preserve the previous ownership. |
| **mode** any                                              | The permissions the resulting filesystem object should have.For those used to */usr/bin/chmod* remember that modes are actually octal numbers. You must either add a leading zero so that Ansible’s YAML parser knows it is an octal number (like `0644` or `01777`) or quote it (like `'644'` or `'1777'`) so Ansible receives a string and can do its own conversion from string into number.Giving Ansible a number without following one of these rules will end up with a decimal number which will have unexpected results.As of Ansible 1.8, the mode may be specified as a symbolic mode (for example, `u+rwx` or `u=rw,g=r,o=r`).If `mode` is not specified and the destination filesystem object **does not** exist, the default `umask` on the system will be used when setting the mode for the newly created filesystem object.If `mode` is not specified and the destination filesystem object **does** exist, the mode of the existing filesystem object will be used.Specifying `mode` is the best way to ensure filesystem objects are created with the correct permissions. See CVE-2020-1736 for further details. |
| **modification_time** string*added in Ansible 2.7*        | This parameter indicates the time the file’s modification time should be set to.Should be `preserve` when no modification is required, `YYYYMMDDHHMM.SS` when using default time format, or `now`.Default is None meaning that `preserve` is the default for `state=[file,directory,link,hard]` and `now` is default for `state=touch`. |
| **modification_time_format** string*added in Ansible 2.7* | When used with `modification_time`, indicates the time format that must be used.Based on default Python format (see time.strftime doc).**Default:** `"%Y%m%d%H%M.%S"` |
| **owner** string                                          | Name of the user that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current user unless you are root, in which case it can preserve the previous ownership.Specifying a numeric username will be assumed to be a user ID and not a username. Avoid numeric usernames to avoid this confusion. |
| **path** aliases: dest, namepath / required               | Path to the file being managed.                              |
| **recurse** boolean*added in Ansible 1.1*                 | Recursively set the specified file attributes on directory contents.This applies only when `state` is set to `directory`.**Choices:**`false` ← (default)`true` |
| **selevel** string                                        | The level part of the SELinux filesystem object context.This is the MLS/MCS attribute, sometimes known as the `range`.When set to `_default`, it will use the `level` portion of the policy if available. |
| **serole** string                                         | The role part of the SELinux filesystem object context.When set to `_default`, it will use the `role` portion of the policy if available. |
| **setype** string                                         | The type part of the SELinux filesystem object context.When set to `_default`, it will use the `type` portion of the policy if available. |
| **seuser** string                                         | The user part of the SELinux filesystem object context.By default it uses the `system` policy, where applicable.When set to `_default`, it will use the `user` portion of the policy if available. |
| **src** path                                              | Path of the file to link to.This applies only to `state=link` and `state=hard`.For `state=link`, this will also accept a non-existing path.Relative paths are relative to the file being created (`path`) which is how the Unix command `ln -s SRC DEST` treats relative paths. |
| **state** string                                          | If `absent`, directories will be recursively deleted, and files or symlinks will be unlinked. In the case of a directory, if `diff` is declared, you will see the files and folders deleted listed under `path_contents`. Note that `absent` will not cause `file` to fail if the `path` does not exist as the state did not change.If `directory`, all intermediate subdirectories will be created if they do not exist. Since Ansible 1.7 they will be created with the supplied permissions.If `file`, with no other options, returns the current state of `path`.If `file`, even with other options (such as `mode`), the file will be modified if it exists but will NOT be created if it does not exist. Set to `touch` or use the [ansible.builtin.copy](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/copy_module.html#ansible-collections-ansible-builtin-copy-module) or [ansible.builtin.template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html#ansible-collections-ansible-builtin-template-module) module if you want to create the file if it does not exist.If `hard`, the hard link will be created or changed.If `link`, the symbolic link will be created or changed.If `touch` (new in 1.4), an empty file will be created if the file does not exist, while an existing file or directory will receive updated file access and modification times (similar to the way `touch` works from the command line).Default is the current state of the file if it exists, `directory` if `recurse=yes`, or `file` otherwise.**Choices:**`"absent"``"directory"``"file"``"hard"``"link"``"touch"` |
| **unsafe_writes** boolean*added in Ansible 2.2*           | Influence when to use atomic operation to prevent data corruption or inconsistent reads from the target filesystem object.By default this module uses atomic operations to prevent data corruption or inconsistent reads from the target filesystem objects, but sometimes systems are configured or just broken in ways that prevent this. One example is docker mounted filesystem objects, which cannot be updated atomically from inside the container and can only be written in an unsafe manner.This option allows Ansible to fall back to unsafe methods of updating filesystem objects when atomic operations fail (however, it doesn’t force Ansible to perform unsafe writes).IMPORTANT! Unsafe writes are subject to race conditions and can lead to data corruption.**Choices:**`false` ← (default)`true` |

**Attributes**

| Attribute      | Support                                                      | Description                                                  |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **check_mode** | **full**                                                     | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**  | **partial**permissions and ownership will be shown but file contents on absent/touch will not. | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform**   | **Platform:** **posix**                                      | Target OS/families that can be operated against              |

**Examples**

```yaml
- name: Change file ownership, group and permissions
  ansible.builtin.file:
    path: /etc/foo.conf
    owner: foo
    group: foo
    mode: '0644'

- name: Give insecure permissions to an existing file
  ansible.builtin.file:
    path: /work
    owner: root
    group: root
    mode: '1777'

- name: Create a symbolic link
  ansible.builtin.file:
    src: /file/to/link/to
    dest: /path/to/symlink
    owner: foo
    group: foo
    state: link

- name: Create two hard links
  ansible.builtin.file:
    src: '/tmp/{{ item.src }}'
    dest: '{{ item.dest }}'
    state: hard
  loop:
    - { src: x, dest: y }
    - { src: z, dest: k }

- name: Touch a file, using symbolic modes to set the permissions (equivalent to 0644)
  ansible.builtin.file:
    path: /etc/foo.conf
    state: touch
    mode: u=rw,g=r,o=r

- name: Touch the same file, but add/remove some permissions
  ansible.builtin.file:
    path: /etc/foo.conf
    state: touch
    mode: u+rw,g-wx,o-rwx

- name: Touch again the same file, but do not change times this makes the task idempotent
  ansible.builtin.file:
    path: /etc/foo.conf
    state: touch
    mode: u+rw,g-wx,o-rwx
    modification_time: preserve
    access_time: preserve

- name: Create a directory if it does not exist
  ansible.builtin.file:
    path: /etc/some_directory
    state: directory
    mode: '0755'

- name: Update modification and access time of given file
  ansible.builtin.file:
    path: /etc/some_file
    state: file
    modification_time: now
    access_time: now

- name: Set access time based on seconds from epoch value
  ansible.builtin.file:
    path: /etc/another_file
    state: file
    access_time: '{{ "%Y%m%d%H%M.%S" | strftime(stat_var.stat.atime) }}'

- name: Recursively change ownership of a directory
  ansible.builtin.file:
    path: /etc/foo
    state: directory
    recurse: yes
    owner: foo
    group: foo

- name: Remove file (delete file)
  ansible.builtin.file:
    path: /etc/foo.txt
    state: absent

- name: Recursively remove directory
  ansible.builtin.file:
    path: /etc/foo
    state: absent
```

**Return Values**

| Key             | Description                                                  |
| --------------- | ------------------------------------------------------------ |
| **dest** string | Destination file/path, equal to the value passed to *path*.**Returned:** state=touch, state=hard, state=link**Sample:** `"/path/to/file.txt"` |
| **path** string | Destination file/path, equal to the value passed to *path*.**Returned:** state=absent, state=directory, state=file**Sample:** `"/path/to/file.txt"` |

## unarchive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html)

**Parameters**

| Parameter                                                    | Comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **attributes** aliases: attrstring*added in Ansible 2.3*     | The attributes the resulting filesystem object should have.To get supported flags look at the man page for *chattr* on the target system.This string should contain the attributes in the same order as the one displayed by *lsattr*.The `=` operator is assumed as default, otherwise `+` or `-` operators need to be included in the string. |
| **copy** boolean                                             | If true, the file is copied from local controller to the managed (remote) node, otherwise, the plugin will look for src archive on the managed machine.This option has been deprecated in favor of `remote_src`.This option is mutually exclusive with `remote_src`.**Choices:**`false``true` ← (default) |
| **creates** path*added in Ansible 1.6*                       | If the specified absolute path (file or directory) already exists, this step will **not** be run.The specified absolute path (file or directory) must be below the base path given with `dest:`. |
| **decrypt** boolean*added in Ansible 2.4*                    | This option controls the autodecryption of source files using vault.**Choices:**`false``true` ← (default) |
| **dest** path / required                                     | Remote absolute path where the archive should be unpacked.The given path must exist. Base directory is not created by this module. |
| **exclude** list / elements=string*added in Ansible 2.1*     | List the directory and file entries that you would like to exclude from the unarchive action.Mutually exclusive with `include`.**Default:** `[]` |
| **extra_opts** list / elements=string*added in Ansible 2.1*  | Specify additional options by passing in an array.Each space-separated command-line option should be a new element of the array. See examples.Command-line options with multiple elements must use multiple lines in the array, one for each element.**Default:** `[""]` |
| **group** string                                             | Name of the group that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current group of the current user unless you are root, in which case it can preserve the previous ownership. |
| **include** list / elements=string*added in ansible-core 2.11* | List of directory and file entries that you would like to extract from the archive. If `include` is not empty, only files listed here will be extracted.Mutually exclusive with `exclude`.**Default:** `[]` |
| **io_buffer_size** integer*added in ansible-core 2.12*       | Size of the volatile memory buffer that is used for extracting files from the archive in bytes.**Default:** `65536` |
| **keep_newer** boolean*added in Ansible 2.1*                 | Do not replace existing files that are newer than files from the archive.**Choices:**`false` ← (default)`true` |
| **list_files** boolean*added in Ansible 2.0*                 | If set to True, return the list of files that are contained in the tarball.**Choices:**`false` ← (default)`true` |
| **mode** any                                                 | The permissions the resulting filesystem object should have.For those used to */usr/bin/chmod* remember that modes are actually octal numbers. You must either add a leading zero so that Ansible’s YAML parser knows it is an octal number (like `0644` or `01777`) or quote it (like `'644'` or `'1777'`) so Ansible receives a string and can do its own conversion from string into number.Giving Ansible a number without following one of these rules will end up with a decimal number which will have unexpected results.As of Ansible 1.8, the mode may be specified as a symbolic mode (for example, `u+rwx` or `u=rw,g=r,o=r`).If `mode` is not specified and the destination filesystem object **does not** exist, the default `umask` on the system will be used when setting the mode for the newly created filesystem object.If `mode` is not specified and the destination filesystem object **does** exist, the mode of the existing filesystem object will be used.Specifying `mode` is the best way to ensure filesystem objects are created with the correct permissions. See CVE-2020-1736 for further details. |
| **owner** string                                             | Name of the user that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current user unless you are root, in which case it can preserve the previous ownership.Specifying a numeric username will be assumed to be a user ID and not a username. Avoid numeric usernames to avoid this confusion. |
| **remote_src** boolean*added in Ansible 2.2*                 | Set to `true` to indicate the archived file is already on the remote system and not local to the Ansible controller.This option is mutually exclusive with `copy`.**Choices:**`false` ← (default)`true` |
| **selevel** string                                           | The level part of the SELinux filesystem object context.This is the MLS/MCS attribute, sometimes known as the `range`.When set to `_default`, it will use the `level` portion of the policy if available. |
| **serole** string                                            | The role part of the SELinux filesystem object context.When set to `_default`, it will use the `role` portion of the policy if available. |
| **setype** string                                            | The type part of the SELinux filesystem object context.When set to `_default`, it will use the `type` portion of the policy if available. |
| **seuser** string                                            | The user part of the SELinux filesystem object context.By default it uses the `system` policy, where applicable.When set to `_default`, it will use the `user` portion of the policy if available. |
| **src** path / required                                      | If `remote_src=no` (default), local path to archive file to copy to the target server; can be absolute or relative. If `remote_src=yes`, path on the target server to existing archive file to unpack.If `remote_src=yes` and `src` contains `://`, the remote machine will download the file from the URL first. (version_added 2.0). This is only for simple cases, for full download support use the [ansible.builtin.get_url](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/get_url_module.html#ansible-collections-ansible-builtin-get-url-module) module. |
| **unsafe_writes** boolean*added in Ansible 2.2*              | Influence when to use atomic operation to prevent data corruption or inconsistent reads from the target filesystem object.By default this module uses atomic operations to prevent data corruption or inconsistent reads from the target filesystem objects, but sometimes systems are configured or just broken in ways that prevent this. One example is docker mounted filesystem objects, which cannot be updated atomically from inside the container and can only be written in an unsafe manner.This option allows Ansible to fall back to unsafe methods of updating filesystem objects when atomic operations fail (however, it doesn’t force Ansible to perform unsafe writes).IMPORTANT! Unsafe writes are subject to race conditions and can lead to data corruption.**Choices:**`false` ← (default)`true` |
| **validate_certs** boolean*added in Ansible 2.2*             | This only applies if using a https URL as the source of the file.This should only set to `false` used on personally controlled sites using self-signed certificate.Prior to 2.2 the code worked as if this was set to `true`.**Choices:**`false``true` ← (default) |

**Attributes**

| Attribute                | Support                                                      | Description                                                  |
| ------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **action**               | **full**                                                     | Indicates this has a corresponding action plugin so some parts of the options can be executed on the controller |
| **async**                | **none**                                                     | Supports being used with the `async` keyword                 |
| **bypass_host_loop**     | **none**                                                     | Forces a ‘global’ task that does not execute per host, this bypasses per host templating and serial, throttle and other loop considerationsConditionals will work as if `run_once` is being used, variables used will be from the first available hostThis action will not work normally outside of lockstep strategies |
| **check_mode**           | **partial**Not supported for gzipped tar files.              | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**            | **partial**Uses gtar’s `--diff` arg to calculate if changed or not. If this `arg` is not supported, it will always unpack the archive. | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform**             | **Platform:** **posix**                                      | Target OS/families that can be operated against              |
| **safe_file_operations** | **none**                                                     | Uses Ansible’s strict file operation functions to ensure proper permissions and avoid data corruption |
| **vault**                | **full**                                                     | Can automatically decrypt Ansible vaulted files              |

**Examples**

```yaml
- name: Extract foo.tgz into /var/lib/foo
  ansible.builtin.unarchive:
    src: foo.tgz
    dest: /var/lib/foo

- name: Unarchive a file that is already on the remote machine
  ansible.builtin.unarchive:
    src: /tmp/foo.zip
    dest: /usr/local/bin
    remote_src: yes

- name: Unarchive a file that needs to be downloaded (added in 2.0)
  ansible.builtin.unarchive:
    src: https://example.com/example.zip
    dest: /usr/local/bin
    remote_src: yes

- name: Unarchive a file with extra options
  ansible.builtin.unarchive:
    src: /tmp/foo.zip
    dest: /usr/local/bin
    extra_opts:
    - --transform
    - s/^xxx/yyy/
```

**Return Values**

| Key                              | Description                                                  |
| -------------------------------- | ------------------------------------------------------------ |
| **dest** string                  | Path to the destination directory. <br>**Returned:** always <br/>**Sample:** `"/opt/software"` |
| **files** list / elements=string | List of all the files in the archive. <br/>**Returned:** When *list_files* is True <br/>**Sample:** `["[\"file1\"", " \"file2\"]"]` |
| **gid** integer                  | Numerical ID of the group that owns the destination directory. <br/>**Returned:** always <br/>**Sample:** `1000` |
| **group** string                 | Name of the group that owns the destination directory. <br/>**Returned:** always <br/>**Sample:** `"librarians"` |
| **handler** string               | Archive software handler used to extract and decompress the archive. <br/>**Returned:** always <br/>**Sample:** `"TgzArchive"` |
| **mode** string                  | String that represents the octal permissions of the destination directory. <br/>**Returned:** always <br/>**Sample:** `"0755"` |
| **owner** string                 | Name of the user that owns the destination directory. <br/>**Returned:** always <br/>**Sample:** `"paul"` |
| **size** integer                 | The size of destination directory in bytes. Does not include the size of files or subdirectories contained within. <br/>**Returned:** always <br/>**Sample:** `36` |
| **src** string                   | The source archive’s path.If *src* was a remote web URL, or from the local ansible controller, this shows the temporary location where the download was stored. <br/>**Returned:** always <br/>**Sample:** `"/home/paul/test.tar.gz"` |
| **state** string                 | State of the destination. Effectively always “directory”. <br/>**Returned:** always <br/>**Sample:** `"directory"` |
| **uid** integer                  | Numerical ID of the user that owns the destination directory. <br/>**Returned:** always <br/>**Sample:** `1000` |

## archive 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/community/general/archive_module.html)

**Parameters**

| Parameter                                                    | Comments                                                     |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| **attributes** aliases: attrstring*added in Ansible 2.3*     | The attributes the resulting filesystem object should have.To get supported flags look at the man page for *chattr* on the target system.This string should contain the attributes in the same order as the one displayed by *lsattr*.The `=` operator is assumed as default, otherwise `+` or `-` operators need to be included in the string. |
| **dest** path                                                | The file name of the destination archive. The parent directory must exists on the remote host.This is required when `path` refers to multiple files by either specifying a glob, a directory or multiple paths in a list.If the destination archive already exists, it will be truncated and overwritten. |
| **exclude_path** list / elements=path                        | Remote absolute path, glob, or list of paths or globs for the file or files to exclude from *path* list and glob expansion.Use *exclusion_patterns* to instead exclude files or subdirectories below any of the paths from the *path* list.**Default:** `[]` |
| **exclusion_patterns** list / elements=path*added in community.general 3.2.0* | Glob style patterns to exclude files or directories from the resulting archive.This differs from *exclude_path* which applies only to the source paths from *path*. |
| **force_archive** boolean                                    | Allows you to force the module to treat this as an archive even if only a single file is specified.By default when a single file is specified it is compressed only (not archived).Enable this if you want to use [ansible.builtin.unarchive](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/unarchive_module.html#ansible-collections-ansible-builtin-unarchive-module) on an archive of a single file created with this module.**Choices:**`false` ← (default)`true` |
| **format** string                                            | The type of compression to use.Support for xz was added in Ansible 2.5.**Choices:**`"bz2"``"gz"` ← (default)`"tar"``"xz"``"zip"` |
| **group** string                                             | Name of the group that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current group of the current user unless you are root, in which case it can preserve the previous ownership. |
| **mode** any                                                 | The permissions the resulting filesystem object should have.For those used to */usr/bin/chmod* remember that modes are actually octal numbers. You must either add a leading zero so that Ansible’s YAML parser knows it is an octal number (like `0644` or `01777`) or quote it (like `'644'` or `'1777'`) so Ansible receives a string and can do its own conversion from string into number.Giving Ansible a number without following one of these rules will end up with a decimal number which will have unexpected results.As of Ansible 1.8, the mode may be specified as a symbolic mode (for example, `u+rwx` or `u=rw,g=r,o=r`).If `mode` is not specified and the destination filesystem object **does not** exist, the default `umask` on the system will be used when setting the mode for the newly created filesystem object.If `mode` is not specified and the destination filesystem object **does** exist, the mode of the existing filesystem object will be used.Specifying `mode` is the best way to ensure filesystem objects are created with the correct permissions. See CVE-2020-1736 for further details. |
| **owner** string                                             | Name of the user that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current user unless you are root, in which case it can preserve the previous ownership.Specifying a numeric username will be assumed to be a user ID and not a username. Avoid numeric usernames to avoid this confusion. |
| **path** list / elements=path / required                     | Remote absolute path, glob, or list of paths or globs for the file or files to compress or archive. |
| **remove** boolean                                           | Remove any added source files and trees after adding to archive.**Choices:**`false` ← (default)`true` |
| **selevel** string                                           | The level part of the SELinux filesystem object context.This is the MLS/MCS attribute, sometimes known as the `range`.When set to `_default`, it will use the `level` portion of the policy if available. |
| **serole** string                                            | The role part of the SELinux filesystem object context.When set to `_default`, it will use the `role` portion of the policy if available. |
| **setype** string                                            | The type part of the SELinux filesystem object context.When set to `_default`, it will use the `type` portion of the policy if available. |
| **seuser** string                                            | The user part of the SELinux filesystem object context.By default it uses the `system` policy, where applicable.When set to `_default`, it will use the `user` portion of the policy if available. |
| **unsafe_writes** boolean*added in Ansible 2.2*              | Influence when to use atomic operation to prevent data corruption or inconsistent reads from the target filesystem object.By default this module uses atomic operations to prevent data corruption or inconsistent reads from the target filesystem objects, but sometimes systems are configured or just broken in ways that prevent this. One example is docker mounted filesystem objects, which cannot be updated atomically from inside the container and can only be written in an unsafe manner.This option allows Ansible to fall back to unsafe methods of updating filesystem objects when atomic operations fail (however, it doesn’t force Ansible to perform unsafe writes).IMPORTANT! Unsafe writes are subject to race conditions and can lead to data corruption.**Choices:**`false` ← (default)`true` |

**Attributes**

| Attribute      | Support  | Description                                                  |
| -------------- | -------- | ------------------------------------------------------------ |
| **check_mode** | **full** | Can run in `check_mode` and return changed status prediction without modifying target. |
| **diff_mode**  | **none** | Will return details on what has changed (or possibly needs changing in `check_mode`), when in diff mode. |

**Examples**

```yaml
- name: Compress directory /path/to/foo/ into /path/to/foo.tgz
  community.general.archive:
    path: /path/to/foo
    dest: /path/to/foo.tgz

- name: Compress regular file /path/to/foo into /path/to/foo.gz and remove it
  community.general.archive:
    path: /path/to/foo
    remove: true

- name: Create a zip archive of /path/to/foo
  community.general.archive:
    path: /path/to/foo
    format: zip

- name: Create a bz2 archive of multiple files, rooted at /path
  community.general.archive:
    path:
    - /path/to/foo
    - /path/wong/foo
    dest: /path/file.tar.bz2
    format: bz2

- name: Create a bz2 archive of a globbed path, while excluding specific dirnames
  community.general.archive:
    path:
    - /path/to/foo/*
    dest: /path/file.tar.bz2
    exclude_path:
    - /path/to/foo/bar
    - /path/to/foo/baz
    format: bz2

- name: Create a bz2 archive of a globbed path, while excluding a glob of dirnames
  community.general.archive:
    path:
    - /path/to/foo/*
    dest: /path/file.tar.bz2
    exclude_path:
    - /path/to/foo/ba*
    format: bz2

- name: Use gzip to compress a single archive (i.e don't archive it first with tar)
  community.general.archive:
    path: /path/to/foo/single.file
    dest: /path/file.gz
    format: gz

- name: Create a tar.gz archive of a single file.
  community.general.archive:
    path: /path/to/foo/single.file
    dest: /path/file.tar.gz
    format: gz
    force_archive: true
```

**Return Values**

| Key                                                     | Description                                                  |
| ------------------------------------------------------- | ------------------------------------------------------------ |
| **archived** list / elements=string                     | Any files that were compressed or added to the archive.**Returned:** success |
| **arcroot** string                                      | The archive root.**Returned:** always                        |
| **dest_state** string*added in community.general 3.4.0* | The state of the *dest* file.`absent` when the file does not exist.`archive` when the file is an archive.`compress` when the file is compressed, but not an archive.`incomplete` when the file is an archive, but some files under *path* were not found.**Returned:** success |
| **expanded_exclude_paths** list / elements=string       | The list of matching exclude paths from the exclude_path argument.**Returned:** always |
| **expanded_paths** list / elements=string               | The list of matching paths from paths argument.**Returned:** always |
| **missing** list / elements=string                      | Any files that were missing from the source.**Returned:** success |
| **state** string                                        | The state of the input `path`.**Returned:** always           |

## hostname 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/hostname_module.html)

**Parameters**

| Parameter                             | Comments                                                     |
| ------------------------------------- | ------------------------------------------------------------ |
| **name** string / required            | Name of the host.If the value is a fully qualified domain name that does not resolve from the given host, this will cause the module to hang for a few seconds while waiting for the name resolution attempt to timeout. |
| **use** string *added in Ansible 2.9* | Which strategy to use to update the hostname.If not set we try to autodetect, but this can be problematic, particularly with containers as they can present misleading information.Note that ‘systemd’ should be specified for RHEL/EL/CentOS 7+. Older distributions should use ‘redhat’.<br>**Choices:**`"alpine"``"debian"``"freebsd"``"generic"``"macos"``"macosx"``"darwin"``"openbsd"``"openrc"``"redhat"``"sles"``"solaris"``"systemd"` |

**Attributes**

| Attribute      | Support                 | Description                                                  |
| -------------- | ----------------------- | ------------------------------------------------------------ |
| **check_mode** | **full**                | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**  | **full**                | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **facts**      | **full**                | Action returns an `ansible_facts` dictionary that will update existing host facts |
| **platform**   | **Platform:** **posix** | Target OS/families that can be operated against              |

**Examples**

```yaml
- name: Set a hostname
  ansible.builtin.hostname:
    name: web01

- name: Set a hostname specifying strategy
  ansible.builtin.hostname:
    name: web01
    use: systemd
```

## cron 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/cron_module.html)

**Parameters**

| Parameter                                     | Comments                                                     |
| --------------------------------------------- | ------------------------------------------------------------ |
| **backup** boolean                            | If set, create a backup of the crontab before it is modified. The location of the backup is returned in the `backup_file` variable by this module.**Choices:**`false` ← (default)`true` |
| **cron_file** path                            | If specified, uses this file instead of an individual user’s crontab. The assumption is that this file is exclusively managed by the module, do not use if the file contains multiple entries, NEVER use for /etc/crontab.If this is a relative path, it is interpreted with respect to */etc/cron.d*.Many linux distros expect (and some require) the filename portion to consist solely of upper- and lower-case letters, digits, underscores, and hyphens.Using this parameter requires you to specify the *user* as well, unless *state* is not *present*.Either this parameter or *name* is required |
| **day** aliases: domstring                    | Day of the month the job should run (`1-31`, `*`, `*/2`, and so on).**Default:** `"*"` |
| **disabled** boolean*added in Ansible 2.0*    | If the job should be disabled (commented out) in the crontab.Only has effect if *state=present*.**Choices:**`false` ← (default)`true` |
| **env** boolean*added in Ansible 2.1*         | If set, manages a crontab’s environment variable.New variables are added on top of crontab.*name* and *value* parameters are the name and the value of environment variable.**Choices:**`false` ← (default)`true` |
| **hour** string                               | Hour when the job should run (`0-23`, `*`, `*/2`, and so on).**Default:** `"*"` |
| **insertafter** string*added in Ansible 2.1*  | Used with *state=present* and *env*.If specified, the environment variable will be inserted after the declaration of specified environment variable. |
| **insertbefore** string*added in Ansible 2.1* | Used with *state=present* and *env*.If specified, the environment variable will be inserted before the declaration of specified environment variable. |
| **job** aliases: valuestring                  | The command to execute or, if env is set, the value of environment variable.The command should not contain line breaks.Required if *state=present*. |
| **minute** string                             | Minute when the job should run (`0-59`, `*`, `*/2`, and so on).**Default:** `"*"` |
| **month** string                              | Month of the year the job should run (`1-12`, `*`, `*/2`, and so on).**Default:** `"*"` |
| **name** string / required                    | Description of a crontab entry or, if env is set, the name of environment variable.This parameter is always required as of ansible-core 2.12. |
| **special_time** string*added in Ansible 1.3* | Special time specification nickname.**Choices:**`"annually"``"daily"``"hourly"``"monthly"``"reboot"``"weekly"``"yearly"` |
| **state** string                              | Whether to ensure the job or environment variable is present or absent.**Choices:**`"absent"``"present"` ← (default) |
| **user** string                               | The specific user whose crontab should be modified.When unset, this parameter defaults to the current user. |
| **weekday** aliases: dowstring                | Day of the week that the job should run (`0-6` for Sunday-Saturday, `*`, and so on).**Default:** `"*"` |

**Attributes**

| Attribute      | Support                 | Description                                                  |
| -------------- | ----------------------- | ------------------------------------------------------------ |
| **check_mode** | **full**                | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**  | **full**                | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform**   | **Platform:** **posix** | Target OS/families that can be operated against              |

**Examples**

```yaml
- name: Ensure a job that runs at 2 and 5 exists. Creates an entry like "0 5,2 * * ls -alh > /dev/null"
  ansible.builtin.cron:
    name: "check dirs"
    minute: "0"
    hour: "5,2"
    job: "ls -alh > /dev/null"

- name: 'Ensure an old job is no longer present. Removes any job that is prefixed by "#Ansible: an old job" from the crontab'
  ansible.builtin.cron:
    name: "an old job"
    state: absent

- name: Creates an entry like "@reboot /some/job.sh"
  ansible.builtin.cron:
    name: "a job for reboot"
    special_time: reboot
    job: "/some/job.sh"

- name: Creates an entry like "PATH=/opt/bin" on top of crontab
  ansible.builtin.cron:
    name: PATH
    env: yes
    job: /opt/bin

- name: Creates an entry like "APP_HOME=/srv/app" and insert it after PATH declaration
  ansible.builtin.cron:
    name: APP_HOME
    env: yes
    job: /srv/app
    insertafter: PATH

- name: Creates a cron file under /etc/cron.d
  ansible.builtin.cron:
    name: yum autoupdate
    weekday: "2"
    minute: "0"
    hour: "12"
    user: root
    job: "YUMINTERACTIVE=0 /usr/sbin/yum-autoupdate"
    cron_file: ansible_yum-autoupdate

- name: Removes a cron file from under /etc/cron.d
  ansible.builtin.cron:
    name: "yum autoupdate"
    cron_file: ansible_yum-autoupdate
    state: absent

- name: Removes "APP_HOME" environment variable from crontab
  ansible.builtin.cron:
    name: APP_HOME
    env: yes
    state: absent
```

## yum_repository 模块

[官网参考文档](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/yum_repository_module.html)

### Parameters

| Parameter                                                | Comments                                                     |
| -------------------------------------------------------- | ------------------------------------------------------------ |
| **async** boolean                                        | If set to `true` Yum will download packages and metadata from this repo in parallel, if possible.In ansible-core 2.11, 2.12, and 2.13 the default value is `true`.This option has been deprecated in RHEL 8. If you’re using one of the versions listed above, you can set this option to None to avoid passing an unknown configuration option.**Choices:**`false``true` |
| **attributes** aliases: attrstring*added in Ansible 2.3* | The attributes the resulting filesystem object should have.To get supported flags look at the man page for *chattr* on the target system.This string should contain the attributes in the same order as the one displayed by *lsattr*.The `=` operator is assumed as default, otherwise `+` or `-` operators need to be included in the string. |
| **bandwidth** string                                     | Maximum available network bandwidth in bytes/second. Used with the *throttle* option.If *throttle* is a percentage and bandwidth is `0` then bandwidth throttling will be disabled. If *throttle* is expressed as a data rate (bytes/sec) then this option is ignored. Default is `0` (no bandwidth throttling).**Default:** `"0"` |
| **baseurl** list / elements=string                       | URL to the directory where the yum repository’s ‘repodata’ directory lives.It can also be a list of multiple URLs.This, the *metalink* or *mirrorlist* parameters are required if *state* is set to `present`. |
| **cost** string                                          | Relative cost of accessing this repository. Useful for weighing one repo’s packages as greater/less than any other.**Default:** `"1000"` |
| **deltarpm_metadata_percentage** string                  | When the relative size of deltarpm metadata vs pkgs is larger than this, deltarpm metadata is not downloaded from the repo. Note that you can give values over `100`, so `200` means that the metadata is required to be half the size of the packages. Use `0` to turn off this check, and always download metadata.**Default:** `"100"` |
| **deltarpm_percentage** string                           | When the relative size of delta vs pkg is larger than this, delta is not used. Use `0` to turn off delta rpm processing. Local repositories (with [file://](file:///) *baseurl*) have delta rpms turned off by default.**Default:** `"75"` |
| **description** string                                   | A human readable string describing the repository. This option corresponds to the “name” property in the repo file.This parameter is only required if *state* is set to `present`. |
| **enabled** boolean                                      | This tells yum whether or not use this repository.Yum default value is `true`.**Choices:**`false``true` |
| **enablegroups** boolean                                 | Determines whether yum will allow the use of package groups for this repository.Yum default value is `true`.**Choices:**`false``true` |
| **exclude** list / elements=string                       | List of packages to exclude from updates or installs. This should be a space separated list. Shell globs using wildcards (eg. `*` and `?`) are allowed.The list can also be a regular YAML array. |
| **failovermethod** string                                | `roundrobin` randomly selects a URL out of the list of URLs to start with and proceeds through each of them as it encounters a failure contacting the host.`priority` starts from the first *baseurl* listed and reads through them sequentially.**Choices:**`"roundrobin"` ← (default)`"priority"` |
| **file** string                                          | File name without the `.repo` extension to save the repo in. Defaults to the value of *name*. |
| **gpgcakey** string                                      | A URL pointing to the ASCII-armored CA key file for the repository. |
| **gpgcheck** boolean                                     | Tells yum whether or not it should perform a GPG signature check on packages.No default setting. If the value is not set, the system setting from `/etc/yum.conf` or system default of `false` will be used.**Choices:**`false``true` |
| **gpgkey** list / elements=string                        | A URL pointing to the ASCII-armored GPG key file for the repository.It can also be a list of multiple URLs. |
| **group** string                                         | Name of the group that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current group of the current user unless you are root, in which case it can preserve the previous ownership. |
| **http_caching** string                                  | Determines how upstream HTTP caches are instructed to handle any HTTP downloads that Yum does.`all` means that all HTTP downloads should be cached.`packages` means that only RPM package downloads should be cached (but not repository metadata downloads).`none` means that no HTTP downloads should be cached.**Choices:**`"all"` ← (default)`"packages"``"none"` |
| **include** string                                       | Include external configuration file. Both, local path and URL is supported. Configuration file will be inserted at the position of the *include=* line. Included files may contain further include lines. Yum will abort with an error if an inclusion loop is detected. |
| **includepkgs** list / elements=string                   | List of packages you want to only use from a repository. This should be a space separated list. Shell globs using wildcards (eg. `*` and `?`) are allowed. Substitution variables (e.g. `$releasever`) are honored here.The list can also be a regular YAML array. |
| **ip_resolve** string                                    | Determines how yum resolves host names.`4` or `IPv4` - resolve to IPv4 addresses only.`6` or `IPv6` - resolve to IPv6 addresses only.**Choices:**`"4"``"6"``"IPv4"``"IPv6"``"whatever"` ← (default) |
| **keepalive** boolean                                    | This tells yum whether or not HTTP/1.1 keepalive should be used with this repository. This can improve transfer speeds by using one connection when downloading multiple files from a repository.**Choices:**`false` ← (default)`true` |
| **keepcache** string                                     | Either `1` or `0`. Determines whether or not yum keeps the cache of headers and packages after successful installation.**Choices:**`"0"``"1"` ← (default) |
| **metadata_expire** string                               | Time (in seconds) after which the metadata will expire.Default value is 6 hours.**Default:** `"21600"` |
| **metadata_expire_filter** string                        | Filter the *metadata_expire* time, allowing a trade of speed for accuracy if a command doesn’t require it. Each yum command can specify that it requires a certain level of timeliness quality from the remote repos. from “I’m about to install/upgrade, so this better be current” to “Anything that’s available is good enough”.`never` - Nothing is filtered, always obey *metadata_expire*.`read-only:past` - Commands that only care about past information are filtered from metadata expiring. Eg. *yum history* info (if history needs to lookup anything about a previous transaction, then by definition the remote package was available in the past).`read-only:present` - Commands that are balanced between past and future. Eg. *yum list yum*.`read-only:future` - Commands that are likely to result in running other commands which will require the latest metadata. Eg. *yum check-update*.Note that this option does not override “yum clean expire-cache”.**Choices:**`"never"``"read-only:past"``"read-only:present"` ← (default)`"read-only:future"` |
| **metalink** string                                      | Specifies a URL to a metalink file for the repomd.xml, a list of mirrors for the entire repository are generated by converting the mirrors for the repomd.xml file to a *baseurl*.This, the *baseurl* or *mirrorlist* parameters are required if *state* is set to `present`. |
| **mirrorlist** string                                    | Specifies a URL to a file containing a list of baseurls.This, the *baseurl* or *metalink* parameters are required if *state* is set to `present`. |
| **mirrorlist_expire** string                             | Time (in seconds) after which the mirrorlist locally cached will expire.Default value is 6 hours.**Default:** `"21600"` |
| **mode** any                                             | The permissions the resulting filesystem object should have.For those used to */usr/bin/chmod* remember that modes are actually octal numbers. You must either add a leading zero so that Ansible’s YAML parser knows it is an octal number (like `0644` or `01777`) or quote it (like `'644'` or `'1777'`) so Ansible receives a string and can do its own conversion from string into number.Giving Ansible a number without following one of these rules will end up with a decimal number which will have unexpected results.As of Ansible 1.8, the mode may be specified as a symbolic mode (for example, `u+rwx` or `u=rw,g=r,o=r`).If `mode` is not specified and the destination filesystem object **does not** exist, the default `umask` on the system will be used when setting the mode for the newly created filesystem object.If `mode` is not specified and the destination filesystem object **does** exist, the mode of the existing filesystem object will be used.Specifying `mode` is the best way to ensure filesystem objects are created with the correct permissions. See CVE-2020-1736 for further details. |
| **module_hotfixes** boolean*added in ansible-core 2.11*  | Disable module RPM filtering and make all RPMs from the repository available. The default is `None`.**Choices:**`false``true` |
| **name** string / required                               | Unique repository ID. This option builds the section name of the repository in the repo file.This parameter is only required if *state* is set to `present` or `absent`. |
| **owner** string                                         | Name of the user that should own the filesystem object, as would be fed to *chown*.When left unspecified, it uses the current user unless you are root, in which case it can preserve the previous ownership.Specifying a numeric username will be assumed to be a user ID and not a username. Avoid numeric usernames to avoid this confusion. |
| **password** string                                      | Password to use with the username for basic authentication.  |
| **priority** string                                      | Enforce ordered protection of repositories. The value is an integer from 1 to 99.This option only works if the YUM Priorities plugin is installed.**Default:** `"99"` |
| **protect** boolean                                      | Protect packages from updates from other repositories.**Choices:**`false` ← (default)`true` |
| **proxy** string                                         | URL to the proxy server that yum should use. Set to `_none_` to disable the global proxy setting. |
| **proxy_password** string                                | Password for this proxy.                                     |
| **proxy_username** string                                | Username to use for proxy.                                   |
| **repo_gpgcheck** boolean                                | This tells yum whether or not it should perform a GPG signature check on the repodata from this repository.**Choices:**`false` ← (default)`true` |
| **reposdir** path                                        | Directory where the `.repo` files will be stored.**Default:** `"/etc/yum.repos.d"` |
| **retries** string                                       | Set the number of times any attempt to retrieve a file should retry before returning an error. Setting this to `0` makes yum try forever.**Default:** `"10"` |
| **s3_enabled** boolean                                   | Enables support for S3 repositories.This option only works if the YUM S3 plugin is installed.**Choices:**`false` ← (default)`true` |
| **selevel** string                                       | The level part of the SELinux filesystem object context.This is the MLS/MCS attribute, sometimes known as the `range`.When set to `_default`, it will use the `level` portion of the policy if available. |
| **serole** string                                        | The role part of the SELinux filesystem object context.When set to `_default`, it will use the `role` portion of the policy if available. |
| **setype** string                                        | The type part of the SELinux filesystem object context.When set to `_default`, it will use the `type` portion of the policy if available. |
| **seuser** string                                        | The user part of the SELinux filesystem object context.By default it uses the `system` policy, where applicable.When set to `_default`, it will use the `user` portion of the policy if available. |
| **skip_if_unavailable** boolean                          | If set to `true` yum will continue running if this repository cannot be contacted for any reason. This should be set carefully as all repos are consulted for any given command.**Choices:**`false` ← (default)`true` |
| **ssl_check_cert_permissions** boolean                   | Whether yum should check the permissions on the paths for the certificates on the repository (both remote and local).If we can’t read any of the files then yum will force *skip_if_unavailable* to be `true`. This is most useful for non-root processes which use yum on repos that have client cert files which are readable only by root.**Choices:**`false` ← (default)`true` |
| **sslcacert** aliases: ca_certstring                     | Path to the directory containing the databases of the certificate authorities yum should use to verify SSL certificates. |
| **sslclientcert** aliases: client_certstring             | Path to the SSL client certificate yum should use to connect to repos/remote sites. |
| **sslclientkey** aliases: client_keystring               | Path to the SSL client key yum should use to connect to repos/remote sites. |
| **sslverify** aliases: validate_certsboolean             | Defines whether yum should verify SSL certificates/hosts at all.**Choices:**`false``true` ← (default) |
| **state** string                                         | State of the repo file.**Choices:**`"absent"``"present"` ← (default) |
| **throttle** string                                      | Enable bandwidth throttling for downloads.This option can be expressed as a absolute data rate in bytes/sec. An SI prefix (k, M or G) may be appended to the bandwidth value. |
| **timeout** string                                       | Number of seconds to wait for a connection before timing out.**Default:** `"30"` |
| **ui_repoid_vars** string                                | When a repository id is displayed, append these yum variables to the string if they are used in the *baseurl*/etc. Variables are appended in the order listed (and found).**Default:** `"releasever basearch"` |
| **unsafe_writes** boolean*added in Ansible 2.2*          | Influence when to use atomic operation to prevent data corruption or inconsistent reads from the target filesystem object.By default this module uses atomic operations to prevent data corruption or inconsistent reads from the target filesystem objects, but sometimes systems are configured or just broken in ways that prevent this. One example is docker mounted filesystem objects, which cannot be updated atomically from inside the container and can only be written in an unsafe manner.This option allows Ansible to fall back to unsafe methods of updating filesystem objects when atomic operations fail (however, it doesn’t force Ansible to perform unsafe writes).IMPORTANT! Unsafe writes are subject to race conditions and can lead to data corruption.**Choices:**`false` ← (default)`true` |
| **username** string                                      | Username to use for basic authentication to a repo or really any url. |

### Attributes

| Attribute      | Support                | Description                                                  |
| -------------- | ---------------------- | ------------------------------------------------------------ |
| **check_mode** | **full**               | Can run in check_mode and return changed status prediction without modifying target |
| **diff_mode**  | **full**               | Will return details on what has changed (or possibly needs changing in check_mode), when in diff mode |
| **platform**   | **Platform:** **rhel** | Target OS/families that can be operated against              |

### Examples

```yaml
- name: Add repository
  ansible.builtin.yum_repository:
    name: epel
    description: EPEL YUM repo
    baseurl: https://download.fedoraproject.org/pub/epel/$releasever/$basearch/

- name: Add multiple repositories into the same file (1/2)
  ansible.builtin.yum_repository:
    name: epel
    description: EPEL YUM repo
    file: external_repos
    baseurl: https://download.fedoraproject.org/pub/epel/$releasever/$basearch/
    gpgcheck: no

- name: Add multiple repositories into the same file (2/2)
  ansible.builtin.yum_repository:
    name: rpmforge
    description: RPMforge YUM repo
    file: external_repos
    baseurl: http://apt.sw.be/redhat/el7/en/$basearch/rpmforge
    mirrorlist: http://mirrorlist.repoforge.org/el7/mirrors-rpmforge
    enabled: no

# Handler showing how to clean yum metadata cache
- name: yum-clean-metadata
  ansible.builtin.command: yum clean metadata

# Example removing a repository and cleaning up metadata cache
- name: Remove repository (and clean up left-over metadata)
  ansible.builtin.yum_repository:
    name: epel
    state: absent
  notify: yum-clean-metadata

- name: Remove repository from a specific repo file
  ansible.builtin.yum_repository:
    name: epel
    file: external_repos
    state: absent
```

### Return Values

| Key              | Description                                                  |
| ---------------- | ------------------------------------------------------------ |
| **repo** string  | repository name<br>**Returned:** success<br>**Sample:** `"epel"` |
| **state** string | state of the target, after execution<br>**Returned:** success<br>**Sample:** `"present"` |

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
