---
title: Systemd run as another user
---

# Podman Systemd Run Docs

On a Linux computer the command `systemd-run` can be used to run Podman as another user.

:::caution
:warning: Work-in-progress, experimental

status: Work-in-progress. Experimental draft. Fact-checking is needed regarding the diagrams
and the explanation of the steps in the diagrams.
The diagrams and the explanation of the steps were written with a bit of guessing of how it works.
:::

## Run podman as another user (using `--property User=`)

Using `sudo systemd-run --user --property User=username ...`

Run the commands

```
sudo useradd test
uid=$(id -u test)
sudo systemd-run \
  --property User=test \
  --property Requires=user@${uid}.service \
  --property After=user@${uid}.service \
  --property Environment=XDG_RUNTIME_DIR=/run/user/${uid} \
  --collect \
  --pipe \
  --quiet \
  --wait \
  podman run --quiet --rm alpine echo hello
```

The text `hello` is written to stdout.

``` mermaid
graph TB

    a1[systemd-run] -.->|1. dbus| a2[systemd]
    a2 -->|"2. (if missing) fork/exec"| a3["systemd --user<br>(user@1000.service)"]
    a2 -->|3. fork/exec| a4[podman]
    a4 -->|4. fork/exec| a5[conmon]
    a5 -->|5. fork/exec| a6[OCI runtime]
    a6 -->|6. exec| a7[container]

    classDef white fill:#fff,stroke:#333;
    class a1,a2 white;
```

The white boxes are processes running as root and the colored boxes are processes
running as the user _test_.
The steps explained in more detail (here assuming _1000_ is UID for the user _test_):

1. __systemd-run__ requests a new transient service unit from the __systemd system manager__
   using dbus. In the request __systemd-run__ also passes the file descriptors
   for stdin, stdout, and stderr to the __systemd system manager__. To learn more about
   the technology used for passing file descriptors over a Unix socket see `SCM_RIGHTS`
   and `sendmsg()` in `man 7 unix`.
2. __systemd system manager__  makes sure that the __systemd user manager__ instance
   user@1000.service is in the _active_ state. If needed, __systemd system manager__
   will start the user@1000.service, which means `systemd --user` is executed.
3. __systemd system manager__ starts __podman__ with a fork/exec.
4. __podman__ starts __conmon__ with a fork/exec.
5. __conmon__ starts __OCI runtime__ with a fork/exec.
6. __OCI runtime__ starts __container__ with an exec.

## Run podman as another user (using `--machine=`)

Using `sudo systemd-run --user --machine=username@ ...`

Run the commands

```
sudo useradd test
sudo systemd-run \
  --collect \
  --machine=test@ \
  --pipe \
  --quiet \
  --user \
  --wait \
  podman run \
    --quiet \
    --rm \
    alpine \
      echo hello
```

The text `hello` is written to stdout.

``` mermaid
graph TB

    a1[systemd-run] -->|"1. fork/exec"| a2[systemd-run]
    a2 -.->|"2. dbus"| a3[systemd]
    a3 -->|"3. fork/exec"| a4[systemd-stdio-bridge]    
    a3 -->|"4. (if missing) fork/exec"| a5["systemd --user<br>(user@1000.service)"]
    a4 -.->|"5. dbus"| a5
    a1 -.->|"6. dbus"| a5
    a5 -->|7. fork/exec| a6[podman]
    a6 -->|8. fork/exec| a7[conmon]
    a7 -->|9. fork/exec| a8[OCI runtime]
    a8 -->|10. exec| a9[container]

    classDef white fill:#fff,stroke:#333;
    class a1,a2,a3 white;
```

The white boxes are processes running as root and the colored boxes
are processes running as the user _test_.
The steps explained in more detail (here assuming _1000_ is UID for the user _test_):

1. __systemd-run__ starts __systemd-run__ (a second instance) with fork/exec
2. __systemd-run__ (second instance) sends a dbus request to __systemd__ requesting that systemd-stdio-bridge
   should be started.
   In the request __systemd-run__ also passes the file descriptors
   for stdin, stdout, and stderr to the __systemd system manager__. To learn more about
   the technology used for passing file descriptors over a Unix socket see `SCM_RIGHTS`
   and `sendmsg()` in `man 7 unix`.
3. __systemd system manager__ starts __systemd-stdio-bridge__ running as user UID _1000_ and let the stdin, stdout,
   stderr
   be the file descriptors it was passed from __step 2__.
4. The __systemd system manager__ makes sure that the __systemd user manager__ instance
   __user@1000.service__ is in the _active_ state. If needed, __systemd system manager__
   will start  __user@1000.service__, which means `systemd --user` is executed.
5. __systemd-stdio-bridge__  announces its precesence to __systemd user manager__ using dbus.
6. __systemd-run__ (first instance) requests a new transient service unit from the
   __systemd user manager__ using dbus.
7. __systemd user manager__ starts __podman__ with a fork/exec.
8. __podman__ starts __conmon__ with a fork/exec.
9. __conmon__ starts __OCI runtime__ with a fork/exec.
10. __OCI runtime__ starts the __container__ with an exec.

## Using `--property OpenFile=`

The systemd directive [`OpenFile=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#OpenFile=)
was introduced in __systemd 253__ (released 15 February 2023) and is available in for example Fedora 38.

The `OpenFile=` directive instructs systemd to open a file before starting the service. The file descriptor
will be passed to the started service as an inherited file descriptor.

#### Example: systemd system manager opens the file

_/root/secretfile_ in read-only mode and the container reads the file descriptor

Run the commands

```
sudo bash -c 'echo "hello from secret" > /etc/secretfile'
sudo chmod 700 /etc/secretfile
sudo useradd test
uid=$(id -u test)
sudo systemd-run \
  --collect \
  --pipe \
  --property OpenFile=/etc/secretfile:myfdname:read-only \
  --property User=test \
  --property Requires=user@${uid}.service \
  --property After=user@${uid}.service \
  --property Environment=XDG_RUNTIME_DIR=/run/user/${uid} \
  --quiet \
  --wait \
  podman run --quiet --rm alpine sh -c "cat <&3"
```

The text `hello from secret` is written to stdout.

#### Example: systemd user manager opens the file

_/home/test/secretfile_ in read-only mode and the container reads the file descriptor

Run the commands

```
sudo useradd test
sudo bash -c 'echo "hello from secret" > /home/test/secretfile'
sudo chmod 700 /home/test/secretfile
sudo chown test:test /home/test/secretfile
uid=$(id -u test)
sudo systemd-run \
  --user \
  --machine=test@ \
  --property OpenFile=/home/test/secretfile:myfdname:read-only \
  --collect \
  --pipe \
  --quiet \
  --wait \
  podman run -q --rm --user 65534:65534 alpine sh -c "cat <&3"
```

The text `hello from secret` is written to stdout. (When I tested
on Fedora CoreOS 38.20230430.1.0 (with container-selinux-2.209.0-1.fc38.noarch)
I first had to run `sudo setenforce 0` to get this example to work)

__audit2allow__ showed that these rules are necessary

```
#============= container_t ==============
allow container_t user_home_t:file read;

#============= systemd_logind_t ==============

#!!!! This avc can be allowed using the boolean 'domain_can_mmap_files'
allow systemd_logind_t etc_t:file map;
```

Note, in the example the option `--user 65534:65534` was added to highlight
the fact that the container user does not need to be mapped
to the regular user of the host (i.e. the user _test_).

#### Example: systemd system manager opens the file

_/root/secretfile_ in read-only mode and a container running in a container reads the file descriptor

Run the commands

```
sudo bash -c 'echo "hello from secret" > /etc/secretfile'
sudo chmod 700 /etc/secretfile
sudo useradd test
uid=$(id -u test)
sudo systemd-run \
  --collect \
  --pipe \
  --property OpenFile=/etc/secretfile:myfdname:read-only \
  --property User=test \
  --property Requires=user@${uid}.service \
  --property After=user@${uid}.service \
  --property Environment=XDG_RUNTIME_DIR=/run/user/${uid} \
  --quiet \
  --wait \
  podman run \
    --device /dev/fuse \
    --quiet \
    --rm \
    --security-opt label=disable \
    --user podman \
    quay.io/podman/stable \
      podman run \
        --quiet \
        --rm \
        alpine sh -c "cat <&3"
```

The text `hello from secret` is written to stdout.

The command took about 1 minute to run. To see more progress remove the `--quiet` flags.
