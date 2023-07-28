---
title: Unix Sockets
---

# Podman OpenFile

Demo: Use systemd directive OpenFile= to let Podman inherit an already connected Unix socket.

The systemd directive [`OpenFile=`](https://www.freedesktop.org/software/systemd/man/systemd.service.html#OpenFile=) was introduced in __systemd 253__ (released 15 February 2023).

__Problem__: A container process does not have file permissions to access a UNIX socket.

__Solution__: Start the container via `systemd-run --property OpenFile=... ...`  so that __systemd__ connects to the UNIX socket. The container process inherits the established socket.

For example, `OpenFile=` makes it possible to fetch a web page with libcurl in a container, even when the container user does not have enough file permission to access the UNIX socket of the web server. It works because systemd will connect to the UNIX socket and then let Podman inherit the file descriptor of the established socket. The container process inherits the same file descriptor.

``` mermaid
stateDiagram-v2
    systemd: systemd connects to UNIX socket (specified by OpenFile=)
    systemd --> podman: socket inherited via fork/exec
    state "OCI runtime" as s2
    podman --> conmon: socket inherited via double fork/exec
    conmon --> s2: socket inherited via fork/exec
    s2 --> container: socket inherited via exec
```

__Test 3__ gives a demonstration of this.

### Requirements

* systemd 253

* podman

You could for instance use Fedora 38 or later.
An easy way to try it out is to use Fedora CoreOS (stream = [__next__](https://getfedora.org/coreos?stream=next)) which already contains Fedora 38.

### Run a web server that listens on a Unix socket

1. Create the file _~/Caddyfile_ with the file contents
   ```
   { 
   }

   http://localhost {
     bind unix//sockdir/sock
     respond "Hello world
   "
   }
   ```
2. Run
   ```
   mkdir ~/sockdir
   ```
3. Run
   ```
   podman run \
	--detach \
	--name caddy \
	--replace \
	--rm \
        -v ~/Caddyfile:/etc/caddy/Caddyfile:Z \
        -v ~/sockdir:/sockdir:Z \
	  docker.io/library/caddy
   ```
   The command creates the Unix socket _~/sockdir/sock_ that has the file permissions __755__.

### Test 1: fetch a web page as container user _root_ (Success)

Fetch a web page with curl (running as the container user __root__)

```
podman run \
       --rm \
       -v ~/sockdir:/sockdir:Z \
       registry.fedoraproject.org/fedora \
	 curl \
	   --no-progress-meter \
	   --unix-socket /sockdir/sock \
           http://localhost
```

The command outputs

```
Hello world
```
and exits successfully.

### Test 2: fetch a web page as container user _nobody_ (Failure)

Fetch a web page with curl (running as the container user __nobody__ (UID 65534 and GID 65534).

```
podman run \
       --rm \
       --user 65534:65534 \
       -v ~/sockdir:/sockdir:Z \
	  registry.fedoraproject.org/fedora \
	    curl \
	      --no-progress-meter \
	      --unix-socket /sockdir/sock \
	      http://localhost
```
The command fails with the error message
```
curl: (7) Couldn't connect to server
```

The UNIX socket _~/sockdir/sock_ has file permissions 755. The user __nobody__ inside the container, thus only has Read (4) and Execute (1) permissions.

The error

_curl: (7) Couldn't connect to server_

can be avoided if we set less restrictive file permissions, for example by running `chmod 777 ~/sockdir/sock`.

Is there a way to download the web page as the container user __nobody__ without changing the file permissions of the Unix socket? Yes, by using the systemd directive `OpenFile=` (see __Test 3__).

### Test 3: Use OpenFile= and fetch a web page as container user _nobody_ (Success)

The curl example program [_externalsocket.c_](https://github.com/curl/curl/blob/master/docs/examples/externalsocket.c) demonstrates how libcurl can use an already established socket when accessing a web server.

The file _externalsocket.c_ was adapted to make use of a UNIX socket file descriptor that originates from OpenFile=

1. Build a container from the modified _externalsocket.c_
   ```
   git clone https://github.com/eriksjolund/curl.git
   cd curl
   git switch externalsocket_openfile
   podman build -t demo docs/examples
   ```
2. Fetch the web page as container user _nobody_
   ```
   systemd-run \
     --quiet \
     --property OpenFile=$HOME/sockdir/sock:fdnametest \
     --user \
     --collect \
     --pipe \
     --wait \
     podman \
       run \
       --rm \
       --user 65534:65534 \
       localhost/demo fdnametest http://localhost
   ```
   The command outputs:
   ```
   Hello world
   ```
   and exits successfully.
