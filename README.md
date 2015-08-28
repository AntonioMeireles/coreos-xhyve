# CoreOS + [xhyve](https://github.com/mist64/xhyve)

**CAVEATS**
-----------
 > - xhyve is a young project, expect bugs! You must be running OS X 10.10.3
 >   Yosemite or later and 2010 or later Mac (i.e. with a CPU that supports EPT)
 >   for this to work.
 > - if you use **any** version of VirtualBox prior to VirtualBox 4.3.30 then
 >   xhyve will crash your system either if VirtualBox is running, or had been
 >   run previously after the last reboot (see xhyve's issues
 >   [#5](mist64/xhyve#5) and [#9](mist64/xhyve#9) for the full context). So,
 >   if you are unable to update VirtualBox to the latest, either of the 4.x or
 >   5.x streams, and were using it in your current session please do restart
 >   your Mac before attempting to run xhyve.

## Step by Step Instructions

### Install xhyve
#### from [homebrew](http://brew.sh) (recommended)
```
❯❯❯ brew install xhyve
```
#### or from [source](https://github.com/mist64/xhyve)
```
❯❯❯ git clone https://github.com/mist64/xhyve
❯❯❯ cd xhyve
❯❯❯ make
❯❯❯ sudo cp build/xhyve /usr/local/bin/
```
#### check it is working...
```
❯❯❯  xhyve -h
Usage: xhyve [-behuwxACHPWY] [-c vcpus] [-g <gdb port>] [-l <lpc>] ...
```

### Install coreos-xhyve
```
❯❯❯ git clone git@github.com:coreos/coreos-xhyve.git
❯❯❯ cd coreos-xhyve
❯❯❯ ./build.sh
```
### kickstart a CoreOS VM
the following command will fetch the `latest` CoreOS Alpha image
available, verify it (if you have gpg installed in your system) with the build
public key, add an OEM xhyve personality and then load it over xhyve.

```
❯❯❯ sudo coreos-xhyve run

```

In your terminal you should see something along this:

```
(...)
This is localhost (Linux x86_64 4.1.5-coreos) 13:23:20
SSH host key: d0:b7:8e:5a:ef:c3:ef:f5:4d:69:c0:ba:35:62:28:3c (DSA)
SSH host key: 2d:11:6f:7c:84:f7:36:34:e7:b9:a8:73:f9:1d:ae:72 (ED25519)
SSH host key: 31:02:9b:95:99:60:d8:5f:74:36:44:30:be:aa:65:ef (RSA)
eth0: 192.168.64.220 fe80::7817:77ff:fe6f:cf32

localhost login: core (automatic login)

CoreOS stable (779.0.0)
Update Strategy: No Reboots
Last login: Tue Aug 25 13:23:20 +0000 2015 on /dev/tty1.
core@localhost ~ $
```
you 'll find out that `/Users` is available (via NFS) already inside your VM.
that will come handy when you come to play with docker volumes later...

### Usage
```
0.0.1

Usage:
  coreos [flags]
  coreos [command]

Available Commands:
  rm          deletes CoreOS image locally
  halt        halts a running CoreOS instance
  ls          lists locally available CoreOS images
  ps          lists running CoreOS instances
  pull        pull a CoreOS image from upstream
  run         pull a CoreOS image from upstream
  version     displays the version number
  help        Help about any command

Flags:
      --debug[=false]: adds extra verbosity for debugging purposes
  -h, --help[=false]: help for coreos
      --json[=false]: outputs in JSON for easy 3rd party integration


Use "coreos [command] --help" for more information about a command.
```
> read [here](documentation/markdown/coreos.md) the full
> auto-generated documentation. 

### simple usage recipe - a docker and rkt playground.
- create a volume to store your persistent data. (will be
  `/var/lib/{docker|rkt}`)
  ```
  ❯❯❯ dd if=/dev/zero of=persistent.img bs=1G count=16
  ```
  in this case we created it with 16GB.

- *label* it
  > requires [homebrew's](http://brew.sh) e2fsprogs package installed.
  >
  > `❯❯❯ brew install e2fsprogs`

  ```
  ❯❯❯ /usr/local/Cellar/e2fsprogs/1.42.12/sbin/e2label var_lib_docker.img rkthdd
  ```
  here, we labeled our volume `rkthdd` which is the *signature* that our
  [*recipe*](cloud-init/docker-only-with-persistent-storage.txt) expects.

  >by relying in *labels* for volume identification we get around the issues we'd
  >have otherwise if we were depending on the actual volume name (/dev/vd...) as
  >those would have to be hardcoded (an issue, if one is mix and matching
  >multiple recipes all dealing with different volumes...)

- start your docker and rkt playground.
  ```
  ❯❯❯ sudo  SSHKEY="ssh-rsa AAAAB3...== x@y.z" \
    UUID=deadbeef-dead-dead-dead-deaddeafbeef \
    ./coreos-xhyve run --volume vda@/path/to/persistent.img --cloud_config \
    cloud-init/docker-only-with-persistent-storage.txt --cpus 2 --memory 2048
  ```
 this will start a VM with the volume we created previously feeded,
 2 virtual cores and 2GB of RAM. The provided
 [cloud-config](cloud-init/docker-only-with-persistent-storage.txt) will format
 the given volume (if it wasn't yet) and bind mount both /var/lib/rkt and
 /var/lib/docker on top of it. docker will also become available through socket
 activation. above we passed arguments to the VM both via
 environment variables and command flags. both ways work, just use whatever
 suits your taste best.

 > Regarding docker, as of 15 August, CoreOS is still in the 1.7 stream so, in
 > order talk to CoreOS' one you'll need on your Mac a matching client,as
 > Homebrew is already at 1.8.x. So if you'll need to ...
 > ```
 > ❯❯❯ brew remove docker
 > ❯❯❯ brew tap homebrew/versions
 > ❯❯❯ brew install docker171
 > ```
- now, from another *shell* in your mac...
```
❯❯❯ docker -H 192.168.64.59:2375 images -a
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
busybox             latest              8c2e06607696        4 months ago        2.43 MB
<none>              <none>              6ce2e90b0bc7        4 months ago        2.43 MB
<none>              <none>
```
or ...

```
❯❯❯ ssh core@192.168.64.59 sudo rkt list
UUID	ACI	STATE	NETWORKS
```
- have fun!

## Contributing
coreos-xhyve is an [open source](http://opensource.org/osd) project under the
[Apache License, Version 2.0](http://opensource.org/licenses/Apache-2.0), ence
contributions are gladly welcomed!

See [CONTRIBUTING](./CONTRIBUTING) for details on submitting patches and the
contribution workflow.
