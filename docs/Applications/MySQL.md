---
title: MySQL
---

# Podman MySQL

Run MySQL on macOS with Podman.

:::caution
status: draft documentation, right now untested
:::

Documentation of how to run __docker.io/library/mysql__ with Podman on macOS and store the mysql data directory on the
macOS host.

### Examine the container image on a Linux computer

On a Fedora 38 (Linux) computer find out which UID and GID
in the container that should be mapped to your regular user.

1. `sudo dnf install podman`
2. `podman pull docker.io/library/mysql`
3. run the command
   ```
   $ podman image inspect \
	     --format "user: {{.User}}" \
	     docker.io/library/mysql
   user:
   $
   ```
   The User field in the image is not set. This is often a hint
   that the container image should be started as the container user __root__ (which is the default).
4. `mkdir datadir`
5. run the mysql container
   ```
   podman run \
       --rm \
       -d \
       --name mysql \
       -e MYSQL_ROOT_PASSWORD=foobar \
       -v ./datadir:/var/lib/mysql:Z \
       docker.io/library/mysql
   ab27226071714d0e15fe4f08014fd42b2a2fd78813dfde609d933cec62292cac
   ```
6. Check which UID and GID ownership the files under the directory _datadir_ have.
   Run the command
   ```
   $ podman unshare ls -ln datadir
   total 90556
   -rw-r-----. 1 999 999       56 May  3 07:43  auto.cnf
   -rw-r-----. 1 999 999        0 May  3 07:44  binlog.index
   -rw-------. 1 999 999     1680 May  3 07:44  ca-key.pem
   -rw-r--r--. 1 999 999     1112 May  3 07:44  ca.pem
   -rw-r--r--. 1 999 999     1112 May  3 07:44  client-cert.pem
   -rw-------. 1 999 999     1680 May  3 07:44  client-key.pem
   -rw-r-----. 1 999 999   196608 May  3 07:44 '#ib_16384_0.dblwr'
   -rw-r-----. 1 999 999  8585216 May  3 07:43 '#ib_16384_1.dblwr'
   -rw-r-----. 1 999 999     5554 May  3 07:44  ib_buffer_pool
   -rw-r-----. 1 999 999 12582912 May  3 07:44  ibdata1
   -rw-r-----. 1 999 999 12582912 May  3 07:44  ibtmp1
   drwxr-x---. 2 999 999      173 May  3 07:44 '#innodb_redo'
   drwxr-x---. 2 999 999      187 May  3 07:44 '#innodb_temp'
   drwxr-x---. 2 999 999      143 May  3 07:44  mysql
   -rw-r-----. 1 999 999 25165824 May  3 07:44  mysql.ibd
   drwxr-x---. 2 999 999     8192 May  3 07:44  performance_schema
   -rw-------. 1 999 999     1676 May  3 07:44  private_key.pem
   -rw-r--r--. 1 999 999      452 May  3 07:44  public_key.pem
   -rw-r--r--. 1 999 999     1112 May  3 07:44  server-cert.pem
   -rw-------. 1 999 999     1680 May  3 07:44  server-key.pem
   drwxr-x---. 2 999 999       28 May  3 07:44  sys
   -rw-r-----. 1 999 999 16777216 May  3 07:44  undo_001
   -rw-r-----. 1 999 999 16777216 May  3 07:44  undo_002
   ```
   All the files are owned by the container UID 999 and GID 999.
   This is good to know when running under macOS
   because there we need to map UIDs and GIDs so that these files
   will be stored with the same ownership as the regular user on the host.
   To do that we will use the command-line option `--userns keep-id:uid=999,gid=999`

### Run mysql on macOS with Podman

Install Podman

1. `brew install podman`
2. `podman machine init`
3. `podman machine start`

Run mysql

1. `mkdir datadir`
2. run the mysql container
   ```
   $ podman -c podman-machine-default \
       run \
       --rm \
       --userns keep-id:uid=999,gid=999 \
       -d \
       -e MYSQL_ROOT_PASSWORD=foobar \
       --security-opt label=disable \
       --name mysql \
       -v ./datadir:/var/lib/mysql \
       docker.io/library/mysql 
   ed8b69089a3d813a44d40f85df9044d0efca6f0720bc1c0dc371edbcba386ccf
   ```
3. optional: verify that the files in the directory _datadir_ are owned by your user and group

```
$ find datadir | wc -l
186
$ find datadir -not -user $(id -un)
$ find datadir -not -group $(id -gn)
```
