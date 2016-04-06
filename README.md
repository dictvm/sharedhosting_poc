# Control Groups

We can now confine users to only use specific resources.

Users created using `site.yml` will be created with a limit of 512CPU shares (out of 1024, which equals to 50% of all cores) and 1GB of RAM (cumulated virtual memory of the user).

### Verify
Login is as our test user. Defaults to `anotheruser`:

`ssh anotheruser@127.0.0.1 -p 2222 -i .vagrant/machines/selinux/virtualbox/private_key`

Run the following command as `anotheruser`:
`$ stress --vm 10 --vm-bytes 120M --timeout 60s`

Run the following command as user `vagrant`. Login via `vagrant ssh`:

`systemd-cgtop`

Pay close attention to the `%CPU` and `Memory` columns showing the Slice `user-1001`:

`/user.slice/user-1001.slice                     14  327.5   276.3M        -        -`

The CPU percentage should never grow beyond 512 while the Memory usage should never grow beyond 1GB of RAM.

You can also cross-check the configured limits using the following commands:

```
$ systemctl show -p CPUShares user-1001.slice
MemoryLimit=1073741824
$ systemctl show -p MemoryLimit user-1001.slice
CPUShares=512
```

# SELinux Basic Usage

# Random information
* New files and directories inherent the parent directory's SELinux type

## Naming
Identity: An identity is not the traditional UNIX UID. An SELinux identity can be assigned to one or more specific UNIX UIDs. The identity can be switched

id -Z
- Shows current user's SELinux context

ls -Z
- Show SELinux context of files or directories

useradd -Z
usermod -Z selinux_policy user

## Trivia

The following SELinux User Capabilities are available:
- User
- Role
- Domain

An overview of all SELinux-context associations with Linux accounts can be found in `/etc/selinux/targeted/modules/active/seusers`.

## Basics
Labeling
- Files, processes, ports
- user:role:type:level(optional)
Type Enforcement

## Tools

setenforce
- set Enforce/Permissive

getenforce
- get current mode

semanage login -l
- show User/SELinux User configuration

semanage fcontext -l
- list available contexts, use grep to filter

sesearch
- search policy types

seinfo
- verbose information on policies

restorecon
- reset security context of files

chcon
- change security context of files temporarily
- will be reset during a relabel of the filesystem (restorecon)

semodule
- import compiled policy module
- list active policy modules

matchpathcon
- compare current security context with default security context

sepolicy
- New tool introduced in RHEL7. Allows for easier policy creation.
