# Privileged containers vs commands - Podman

Some commands of podman - exec, create and run have an option called `privileged`.
If we look at documentation of the `--privileged` option in podman run and exec commands, they say the same thing -

```
--privileged
       Give extended privileges to this container. The default is false.

       By default, Podman containers are unprivileged (=false) and cannot, for example, modify parts of the operating system. This is
       because by default a container is only allowed limited access to devices. A "privileged" container is given the same access to
       devices as the user launching the container, with the exception of virtual consoles (/dev/tty\d+) when running in systemd mode
       (--systemd=always).

       A privileged container turns off the security features that isolate the container from the host. Dropped Capabilities, limited
       devices, read-only mount points, Apparmor/SELinux separation, and Seccomp filters are all disabled.  Due to the disabled secu‐
       rity features, the privileged field should almost never be set as containers can easily break out of confinement.

       Containers  running  in  a  user namespace (e.g., rootless containers) cannot have more privileges than the user that launched
       them.
```

However, podman exec can only execute a command in an existing container. It does not start a new container.
So what happens when we use privileged option with podman exec?
Here, we will focus on two features to highlight differences - process capabilities and dropped devices.

## Privileged containers

If we start a new container with podman run using the `--privileged` option -
```
$ podman run -it --privileged --name priv1 alpine
```
we can verify that the container is in fact privileged by inspecting it -
```
$ podman container inspect priv1 -f '{{.HostConfig.Privileged }}'
true
```
If we check the effective capabilities of the sh process running in the above container from the host -
```
$ podman top priv1 pid hpid args
PID   HPID    COMMAND
1     19337   /bin/sh

$ getpcaps 19337
19337: =ep
```
Here we first get the host PID (HPID field) of the sh process in the container, and then use **getpcaps** (Get process capabilities) utility with that PID to list the capabilities of the process.
Process capabilties can also be seen with the same podman top command, using capeff (effective capabilities), etc.
If we run `sleep 100` inside the container, and check the capabilities of the sleep process from the host, we get -
```
$ podman top priv1 pid hpid args
PID   HPID    COMMAND
1     19337   /bin/sh
4     20424   /bin/sh

$ getpcaps $(podman top priv1 hpid | tail -n +2)
19337: =ep
20424: =ep
```
which shows that both the sh process and the sleep process have same capabilities.
Now what does `=ep` means for capabilities?

If we look at [cap_from_text](https://www.man7.org/linux/man-pages/man3/cap_from_text.3.html) manual page, it says -
>      In the case that the leading operator is `=', and no list of
>      capabilities is provided, the action-list is assumed to refer to
>      `all' capabilities.  For example, the following three clauses are
>      equivalent to each other (and indicate a completely empty
>      capability set): "all="; "="; "cap_chown,<every-other-
>      capability>=".

which means that both above processes have full capabilities on the host. Since entire container is privileged, all process have full capabilities by default.
Note that if this is a rootless container, they will not have more capabilties on the host then the rootless user though.

Even if we run a command via exec without using the privileged option, the command still has full privileges.
```
$ podman exec -it priv1 /bin/sh
```
```
$ podman top priv1 pid hpid args
PID   HPID    COMMAND
1     19337   /bin/sh
7     20756   /bin/sh

$ getpcaps $(podman top priv1 hpid | tail -n +2)
19337: =ep
20756: =ep
```
If we check the devices available inside the container, we see that there are a lot of devices available.
```
# ls /dev/             ## This is being run inside the container
autofs           kmsg             pts              stderr           ttyS23           usbmon0          vcsu1
bsg              kvm              random           stdin            ttyS24           usbmon1          vcsu2
btrfs-control    loop-control     rfkill           stdout           ttyS25           usbmon2          vcsu3
bus              lp0              rtc0             tty              ttyS26           usbmon3          vcsu4
console          lp1              sda              ttyS0            ttyS27           userfaultfd      vcsu5
core             lp2              sda1             ttyS1            ttyS28           vcs              vcsu6
cpu              lp3              sda2             ttyS10           ttyS29           vcs1             vfio
cpu_dma_latency  mapper           sda3             ttyS11           ttyS3            vcs2             vga_arbiter
...
```

## Privileged commands

If we start a non-privileged container, and check its capabilities on host -
```
$ podman run -it --name notpriv1 alpine
```
```
$ podman container inspect notpriv1 -f '{{.HostConfig.Privileged }}'
false
```
```
$ podman top notpriv1 pid hpid args
PID   HPID    COMMAND
1     30042   /bin/sh

$ getpcaps $(podman top notpriv1 hpid | tail -n +2)
30042: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap=ep
```
We can see that the process in the container has specific capabilities, instead of full capabilities.

If we check the devices available to the container, we see that only a few devices are available.
```
# ls /dev/             ## This is being run inside the container
console  fd       mqueue   ptmx     random   stderr   stdout   urandom
core     full     null     pts      shm      stdin    tty      zero
```

However, if we run a command using exec with privileged option in this non-privileged container, we see that the new command again has full capabilities.
```
$ podman exec --privileged -it notpriv1 /bin/sh
```
```
$ podman top notpriv1 pid hpid args
PID   HPID    COMMAND
1     30042   /bin/sh
13    31052   /bin/sh

$ getpcaps $(podman top notpriv1 hpid | tail -n +2)
30042: cap_chown,cap_dac_override,cap_fowner,cap_fsetid,cap_kill,cap_setgid,cap_setuid,cap_setpcap,cap_net_bind_service,cap_sys_chroot,cap_setfcap=ep
31052: =ep
```

But even with privileged command, we still have access to limited devices only.
```
## Similar to running a new priviledged shell in the container and executing ls /dev/ in it.
$ podman exec --privileged -it notpriv1 ls /dev/
console  fd       mqueue   ptmx     random   stderr   stdout   urandom
core     full     null     pts      shm      stdin    tty      zero
```

This is because visible devices are shared across the container, whereas the capabilities are per process.
Moreover, SELinux labels are applied to the new process based on the container SELinux labels, i.e., if we exec with privileged option into
a non-privileged container, the new process will have same non-privileged SELinux label as container, instead of a privileged one.
We have not discussed SELinux labels here, but they can be viewed with podman top command with `label` field, or `/proc/<pid>/attr/current` from host process directly.

## Conclusion

We can conclude that privileged option means different things for commands like run, which are used to create a new container, 
vs exec command, which is used to run only a new process in an existing container.
