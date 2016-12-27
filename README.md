# What this is and isn't

This is a prototype for a shared hosting environment where users are allowed to execute arbitrary
code and bind their apps to all non-privileged ports. During the development of a new  
version of (https://uberspace.de)[Uberspace] I was trying to come up with a solution to the
following problems:

## Resource limits
Current Uberspace systems are persistently checking for processes that use more than 600MB of
memory for a longer period to ensure that there are no excessive noisy neighbors on the system that
could limit the experience of other users on the system. These processes are then sent a SIGTERM and
the owner of the process is informed via mail about the reasons for doing so. This approach doesn't
work for I/O and CPU shares though.

This prototype demonstrates how to create unprivileged users with specified resource limitations by
creating a Control Group (this is why it's called cgroups) with pre-defined values for `CPUShares` and `MemoryLimit`. 

## Port assignment
Uberspace currently allows all users to bind to any unpriviled ports. This is a security risk as
another user on the system can try to bind to a previously used port in case of a reboot or a crash
of the service and collect all incoming traffic to this port or even impersonate the service, if it
isn't properly secured, e.g. by using selfsigned certificates or no TLS at all.

I tried to leverage SELinux to assign ports to SELinux-users so that all users are still able to
connect to assigned ports without being able to bind their own apps to any port that isn't assigned
to them. According to my tests I was successful.

**Note:** To the best of my knowledge the scenario I described never happened in all of my 4 years at Uberspace.

## Increased Security
Uberspace currently disables SELinux. Everyone is scared of the complexity of SELinux and therefore
just disables SELinux. According to my own experience, documentation of SELinux is also pretty
overwhelming, outdated and often distribution-vendor-§specific. 

My approach is based on (https://major.io/2013/07/05/confine-untrusted-users-including-your-children-with-selinux/)[a blog post by Major Hayden]. 

This attempt should allow for anything that unprivileged users are supposed to be doing on a Linux
system while limiting the scope of the impact of a local root exploit on a shared hosting system
like Uberspace, as long as the attack isn't targeted against SELinux in the Linux kernel.

## Why not use containers?
To the best of my knowledge, there is no way to allow unprivileged users to create containers without
either adding them to a privileged group that is root-equivalent while also exposing control over
all containers to any user on the system. Due to the fact that Uberspace is still primarily shared
hosting and not a PAAS-solution, containers where not an option for the version of Uberspace that I was working on.

## Will this be used for Uberspace?

I do not know if any of this will end up in the next version of Uberspace because I am no longer
working for Uberspace. Please direct all questions regarding the new version of Uberspace to either
(mailto:hallo@uberspace.de)[hallo@uberspace.de] or to
(https://twitter.com/ubernauten)[@ubernauten]. 

Please consider this as a proof of concept.

# Control Groups
We can confine users to only use specified resources.

Users created using `site.yml` will be created with a limit of 512CPU shares (out of 1024, which equals to 50% of all cores) and 1GB of RAM (cumulated virtual memory of the user). The user variable defaults to `poc_testuser`.

# Test & Verify
Login is as our test user. Defaults to `poc_testuser`:  
`$ ssh poc_testuser@127.0.0.1 -p 2222 -i .vagrant/machines/selinux/virtualbox/private_key`

Run the following command as `poc_testuser`:  
`$ stress --vm 10 --vm-bytes 120M --timeout 60s`

Run the following command as user `vagrant`. Login via `vagrant ssh`:  
`$ systemd-cgtop`

Pay close attention to the `%CPU` and `Memory` columns showing the Slice `user-1001`:

`/user.slice/user-1001.slice                     14  327.5   276.3M        -        -`

The CPU percentage should never grow beyond 512 while the Memory usage should never grow beyond 1GB of RAM.  

**Note:** Both values can vary during verification.

You can also cross-check the configured limits using the following commands:

```
$ systemctl show -p CPUShares user-1001.slice
CPUShares=512
$ systemctl show -p MemoryLimit user-1001.slice
MemoryLimit=1073741824
```

# SELinux Basic Usage

## Uberspace-related

### Ports
Our current default policy per user allows connections to all outgoing ports. The policy is defined in the template files `shelluser_u.j2` and `shelluser.te.j2`. A detailed explanation of both template files will follow.

A user cannot bind to a local port, without first adding it to her user-specific `uberspace_poc_testuser_port_t`-label.

To ensure that a user cannot elevate their privileges by creating a Uberspace account like `system` or `unconfined`, we add a prefix called `uberspace_`. This also allows for easier management of Uberspace-user's ports.

`semanage` can modify user context modules and port types. Each modification of an existing user context is being merged into a SELinux-managed diff-table that is being applied to each user's existing policy. This means that we do not need to edit individual policies to add or remove ports. We can also modify a user's context without having to manually re-apply port modifications of a user's context.

In simpler words: `semanage` keeps track of changes to individual user's security contexts outside of each policy template. Each change is applied to the base context, even if the context-module of the user is changed.

Add a port:  
`semanage port -a -t uberspace_poc_testuser_port_t -p $protocol $port`

Delete a port:  
`semanage port -d -t uberspace_poc_testuser_port_t -p $protocol $port`

**Note:** If a deleted port is still in use, the user will be able to use the port as long as her process is bound to the port.

List all ports enabled for a user:  
`semanage port --list | grep uberspace_poc_testuser_port_t`

List all ports enabled for all Uberspace users:  
`semanage port --list | grep uberspace_`

#### Test & Verify

Create a second user by replacing the user-variable in `site.yml`. In this example, the user is `ubertest`.

Allow `ubertest` to bind to port 31337/UDP, which is the default port of `netcat`.
```
[vagrant@selinux ~]$ sudo semanage port -a -t uberspace_ubertest_port_t -p udp 31337
```

Login as `ubertest`:
```
$ ssh ubertest@127.0.0.1 -p 2222 -i .vagrant/machines/selinux/virtualbox/private_key  
$ nc -u -l
```

Now log in as `poc_testuser`:
```
[poc_testuser@selinux ~]$ nc -u -l
Ncat: bind to :::31337: Permission denied. QUITTING.
```

As the example demonstrates, the user `poc_testuser` cannot bind to the port that has just been
assigned to `ubertest`.

Now we send data via netcat from `poc_testuser` to `ubertest`:
```
[poc_testuser@selinux ~]$ echo "isthisevenworking?" > foo.txt && nc -u localhost 31337 < foo.txt
[uberspace@selinux ~]$ nc -u -l
isthisevenworking?
```

**Note:** As you can see, it is still possible to locally connect to a port that has been assigned to a user! If a user wants to ensure that access to her process is restricted to her account only, she must use a UNIX socket. Applications still need authentication when binding to a TCP/UDP-port, even if the port is not opened in iptables!

### Sockets
Binding to sockets is possible without any limitations.

# Useful information about SELinux and its tools
**NOTE:** I haven't checked all of this in months. Please feel free to correct me wherever I am wrong!

New files and directories inherent the parent directory's SELinux type... tbc.

## Naming
Identity: An identity is not the traditional UNIX UID. An SELinux identity can be assigned to one or more specific UNIX UIDs. The identity can be switched. tbc.

`id -Z`
- Shows current user's SELinux context

`ls -Z`
- Show SELinux context of files or directories

`useradd -Z`
`zusermod -Z selinux_policy user`

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
tbc.

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
