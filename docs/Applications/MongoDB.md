---
title: MongoDB
---

# Podman MongoDB

Run Mongodb on macOS with Podman.

:::caution
status: draft documentation, right now untested
:::

Documentation of how to run __docker.io/library/mongodb__ with Podman on macOS and store the MongoDB data directory on
the macOS host.

### Examine the container image on a Linux computer

On a Fedora 38 (Linux) computer find out which UID and GID
in the container that should be mapped to your regular user.

1. `sudo dnf install podman`
2. `podman pull docker.io/library/mongo`
3. run the command
   ```
   $ podman image inspect \
	     --format "user: {{.User}}" \
	     docker.io/library/mongo
   user:
   $
   ```
   The User field in the image is not set. This is often a hint
   that the container image should be started as the container user __root__ (which is the default).
4. `mkdir datadir`
5. run the mongodb container
   ```
   podman run \
       --rm \
       -d \
       --name mongo \
       -p 27017:27017 \
       -v ./datadir:/data/db:Z \
       docker.io/library/mongo 
   ab27226071714d0e15fe4f08014fd42b2a2fd78813dfde609d933cec62292cac
   ```
6. Check which UID and GID ownership the files under the directory _datadir_ have.
   Run the command
   ```
   $ podman unshare ls -ln datadir
   total 64
   -rw-------. 1 999 999 4096 May  3 06:39 collection-0-7725828959745876180.wt
   -rw-------. 1 999 999 4096 May  3 06:39 collection-2-7725828959745876180.wt
   -rw-------. 1 999 999 4096 May  3 06:39 collection-4-7725828959745876180.wt
   drwx------. 2 999 999   71 May  3 06:39 diagnostic.data
   -rw-------. 1 999 999 4096 May  3 06:39 index-1-7725828959745876180.wt
   -rw-------. 1 999 999 4096 May  3 06:39 index-3-7725828959745876180.wt
   -rw-------. 1 999 999 4096 May  3 06:39 index-5-7725828959745876180.wt
   -rw-------. 1 999 999 4096 May  3 06:39 index-6-7725828959745876180.wt
   drwx------. 2 999 999  110 May  3 06:39 journal
   -rw-------. 1 999 999 4096 May  3 06:39 _mdb_catalog.wt
   -rw-------. 1 999 999    2 May  3 06:39 mongod.lock
   -rw-------. 1 999 999 4096 May  3 06:39 sizeStorer.wt
   -rw-------. 1 999 999  114 May  3 06:39 storage.bson
   -rw-------. 1 999 999   50 May  3 06:39 WiredTiger
   -rw-------. 1 999 999 4096 May  3 06:39 WiredTigerHS.wt
   -rw-------. 1 999 999   21 May  3 06:39 WiredTiger.lock
   -rw-------. 1 999 999 1165 May  3 06:39 WiredTiger.turtle
   -rw-------. 1 999 999 4096 May  3 06:39 WiredTiger.wt
   ```
   All the files are owned by the container UID 999 and GID 999.
   This is good to know when running under macOS
   because there we need to map UIDs and GIDs so that these files
   will be stored with the same ownership as the regular user on the host.
   To do that we will use the command-line option `--userns keep-id:uid=999,gid=999`

### Run Mongodb on macOS with Podman

Install Podman

1. `brew install podman`
2. `podman machine init`
3. `podman machine start`

Run MongoDB

1. `mkdir datadir`
2. run the mongo container

```
podman -c podman-machine-default \
    run \
    --rm \
    --userns keep-id:uid=999,gid=999 \
    -d \
    --security-opt label=disable \
    --name mongo \
    -p 27017:27017 \
    -v ./datadir:/data/db \
    docker.io/library/mongo 
ed8b69089a3d813a44d40f85df9044d0efca6f0720bc1c0dc371edbcba386ccf
```

3. optional: verify that the files in the directory _datadir_ are owned by your user and group

```
$ find datadir | wc -l
23
$ find datadir -not -user $(id -un)
$ find datadir -not -group $(id -gn)
```

There is no _chown_ error

```
$ podman -c podman-machine-default logs mongo | grep -i chown
$
```
