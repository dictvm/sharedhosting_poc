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
