# Simple NSO and ConfD interaction 

This note is a start-up guide, 
that shows how to use ConfD example `intro/1-2-3-start-query-model` with NSO.
Since ConfD supports NETCONF, we will take advantage of NETCONF NED,
which means out of the box interoperability between ConfD example and NSO.
We only need to configure connection information, like ports, credentials and keys.

It can be used with ConfD Premium or with ConfD Basic. 

ConfD Basic and NSO Free trial can be obtained from following sites:

https://developer.cisco.com/docs/nso - [download link](https://developer.cisco.com/docs/nso/#!getting-nso/getting-nso)  
https://www.tail-f.com/confd-basic - [download link](https://developer.cisco.com/site/confD/downloads/)

## What you will learn
                                           
* build and start ConfD and example application
* build and link NETCONF NED package for NSO
* create and configure ConfD device in NSO 
* configure ConfD example in NSO

Each steps will be explained and followed by 
copy/paste commands that you can directly paste
into your (linux) terminal.

It is recommended to open 4 shells.
It is recommended to use tiled mode (e.g. using [tmux](https://github.com/tmux/tmux/wiki)).

* *shell 1* - to build and run ConfD example 
* *shell 2* - to build NSO directory and NED package, to run NSO itself
* *shell 3* - to run ConfD CLI
* *shell 4* - to run NSO CLI

## Prerequisites

* installed ConfD (with `examples.confd` package) - see `README` from ConfD package
* installed NSO - see `README` form NSO package
* docker environment (optional)
* `xmlstarlet` (optional) for easier modification of `confd.conf` file
* build environment (`gcc`, `Makefile`) for ConfD examples (on Ubuntu use `apt-get install build-essential`) 
   
To check ConfD and NSO installation, run following commands and check the output.

NOTE:: _version numbers may differ, according to your installation_ 

run in *shell 1*:
```shell
confd --version
```
expected output:
```shell
$ confd --version 
7.4
```
run in *shell 2*:
```shell
ncs --version
```
expected output:
```shell
$ ncs --version 
5.5
```

NOTE:: Many NSO commands starts with `ncs` instead of `nso`. This based on the name of NSO's
predecessor NCS.

## Build and start ConfD and example application

In ConfD installation directory you can find examples under `examples.confd`
subdirectory. We will use `examples.confd/intro/1-2-3-start-query-model` example.

You can build and use example directly in the example directory, or you can copy it
to some other directory. The latter is probably better idea, since we will modify
`confd.conf` file 
(e.g. `cp -r $CONFD_DIR/examples.confd/intro/1-2-3-start-query-model /tmp; cd /tmp//1-2-3-start-query-model`). 

First, we will build the example. Go to the example directory and run
in *shell 1*:

```shell
make clean all 
```
expected output:
```shell
$ make clean all 
rm -rf \
	*.o *.a *.xso *.fxs *.xsd *.ccl \
	*_proto.h \
	./confd-cdb *.db aaa_cdb.* \
	rollback*/rollback{0..999} rollback{0..999} \
	cli-history \
	host.key host.cert ssh-keydir \
	*.log confderr.log.* \
	etc *.access \
	running.invalid global.data _tmp* local.data
rm -rf dhcpd.h dhcpd_conf dhcpd.conf 2> /dev/null || true
rm -rf *log *trace cli-history 2> /dev/null || true
/confd-7.4.x86_64/bin/confdc --fail-on-warnings  -c -o dhcpd.fxs  dhcpd.yang
/confd-7.4.x86_64/bin/confdc -c commands-j.cli
/confd-7.4.x86_64/bin/confdc -c commands-c.cli
mkdir -p ./confd-cdb
cp /confd-7.4.x86_64/var/confd/cdb/aaa_init.xml ./confd-cdb
ln -s /confd-7.4.x86_64/etc/confd/ssh ssh-keydir
/confd-7.4.x86_64/bin/confdc --emit-h dhcpd.h dhcpd.fxs
cc -c -o dhcpd_conf.o dhcpd_conf.c -Wall -g -I/confd-7.4.x86_64/include -DCONFD_C_PRODUCT_CONFD
cc -o dhcpd_conf dhcpd_conf.o /confd-7.4.x86_64/lib/libconfd.a -lpthread -lm
C build complete
Build complete
```
NOTE::
You can start and investigate example with Makefile target commands `make start`, `make cli-c`, etc.
(See example README for details) and stop it with `make stop`.

## Build and link NETCONF NED package for NSO

First, we need to set-up NSO directory and enter it. Run in `shell 2`:

```shell
ncs-project create nsotest
cd nsotest
```
NOTE:: Older way was to use `ncs-setup --dest nsotest`

Next, we build NETCONF NED from the example YANG file(s) and link it to 
NSO packages. Run in `shell 2`:

```shell
ncs-make-package --no-java \
        --netconf-ned $CONFD_DIR/examples.confd/intro/1-2-3-start-query-model \
        dhcpned  #<1>
$ ncs-setup --package dhcpned --dest . #<2>
```
<1> create NETCONF NED from YANG files (do not use java binding)  
<2> add(link) NED to NSO packages
 
### Modify ConfD example ports

Before we start ConfD example, we need to modify `confd.conf` of the example, 
to use different CLI and NETCONF ports, so they do not conflict with NSO
CLI and NETCONF ports (which are same). Open `confd.conf` and add or modify:

* add `/confdConfig/cli/ssh/port` --> `13022` (original `2022`)
* modify `/confdConfig/netconf/transport/ssh/port` --> `14022` (oroginal `)

Corresponding `CLI` and NETCONF sections should look like:

```xml
<cli>
  <ssh>
    <port>13022</port>
  </ssh>
</cli>
```

```xml
 <netconf>   <1>
    <transport>
      <ssh>
        <enabled>true</enabled>
        <ip>127.0.0.1</ip>
        <port>14022</port>
      </ssh>
    </transport>
    ...
  </netconf>
```         
<1> There will be other elements in `<netconf>` section, only changed part is displayed here.
 
You can also use following `xmlstarlet` commands, to make modification automatically. Run in the *shell 1*:
 
```shell
export EXAMPLE_DIR=/tmp/1-2-3-start-query-model  #<1>
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig -t elem -n cli ${EXAMPLE_DIR}//confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig/conf:cli -t elem -n ssh ${EXAMPLE_DIR}/confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig/conf:cli/conf:ssh -t elem -n port ${EXAMPLE_DIR}//confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -u "/conf:confdConfig/conf:cli/conf:ssh/conf:port" -v 13022 ${EXAMPLE_DIR}/confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -u "/conf:confdConfig/conf:netconf/conf:transport/conf:ssh/conf:port" -v 14022 ${EXAMPLE_DIR}/confd.conf
```
<1> set `EXAMPLE_DIR` as needed
 
To test the modification works, start the example (in *shell 1*) with `make clean all start` and
test NETCONF access. Run *shell 3*:

```shell
netconf-console --port 14022 --hello
```

To test SSH CLI access, run in *shell 3*:

```shell
ssh admin@127.0.0.1 -p 13022
```
NOTE:: Use `exit` command to exit example CLI

## Create and configure ConfD device in NSO

Once we have everything set-up, we can start configuring the  ConfD example as NSO device. 

If you do not have ConfD example running from previous steps, start it in the *shell 1*:

```shell
make clean all start
```

after that, start  NSO in *shell 2*:

```shell
ncs --with-package-reload
```

NOTE:: Starting NSO can take some time.

next, we can enter NSO CLI and configure the device. In *shell 3* run:

```shell
ncs_cli -u admin -C
```
finally we enter config mode with command (in *shell 3*):

```shell
config
```

### Configure `authgroup`

In order NSO device can connect to real NETCONF device, we need to 
provide authorization details. This is done by linking it with `authgroup`.
We configure `authgroup` in config mode of NSO CLI. Type in *shell 3*:

```shell
devices authgroups group devnetconf
default-map remote-name admin
default-map remote-password admin
commit
top
```

### Configure and synchronize NETCONF device

Now we have everything ready, to configure NETCONF device and connect running 
ConfD example with NSO. Type in *shell 3*:

```shell
devices device EX_NETCONF
address 127.0.0.1
port 14022
authgroup devnetconf
device-type netconf ned-id dhcpned
state admin-state unlocked
commit
```
  
Once device is configured, we can try to synchronize it, so we know connection to device is
correctly established. Type in *shell 3*:

```shell
ssh fetch-host-keys
sync-from
```


## Configure ConfD example in NSO

We have ConfD example attached to NSO as device (name `EX_NETCONF`). We can configure it.
In  *shell 3* type (make sure you are still in config mode):

```shell
top
devices device EX_NETCONF
config dhcp default-lease-time 700s
commit
```

To verify the configuration was performed in ConfD, open example CLI and check it.
In *shell 4* got o example directory (e.g. cd `/tmp//1-2-3-start-query-model`) and
run following command to enter CLI:

```shell
make cli-c
```

once in ConfD example CLI, type (in *shell 4*):

```shell
show full-configuration dhcp
```
 

In similar way we can display the same data in NSO CLI. Tyep in *shell 3*:

```shell
top
show full-configuration devices device EX_NETCONF
```



## Conclusion


   
 

