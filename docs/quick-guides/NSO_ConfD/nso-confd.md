# NSO and ConfD interaction 

Version history
     
```
--------------------------------------------------------------------------------
Version:  Date:         Description:                        Author:             
--------------------------------------------------------------------------------
0.0.1     2020-12-22    Initial version                     Michal Novák        
                                                            micnovak@cisco.com
                                                              
0.1.0     2021-01-04    Version for GitHub Pages            Michal Novák           
                                                            micnovak@cisco.com

0.2.0     2021-01-06    Updated document text               Michal Novák        
                                                            micnovak@cisco.com               
--------------------------------------------------------------------------------
```

## Introduction

This note describes how to connect NETCONF compliant device with NSO.
We will use ConfD example application `intro/1-2-3-start-query-model` as the device.

ConfD fully supports NETCONF and we can take advantage of NSO's NETCONF NED (NETCONF Element Driver).
This means out of the box interoperability between ConfD and NSO. We only need to configure
the device (inside NSO's `device` data model) and provide connection information like ports,
credentials and keys.

The steps can be used with ConfD Premium or with ConfD Basic.

ConfD Basic and NSO Free trial can be obtained from following sites:

https://developer.cisco.com/docs/nso - [download link](https://developer.cisco.com/docs/nso/#!getting-nso/getting-nso)  
https://www.tail-f.com/confd-basic - [download link](https://developer.cisco.com/site/confD/downloads/)

### What you will learn

* how to build and start ConfD example application
* how to make, build and link NETCONF NED package for NSO
* how to create and configure ConfD (NETCONF) device in NSO
* how configure the device data model (ConfD example data model ) in NSO

It is recommended to open 4 terminal shells,
ideally in a tiled mode (e.g. using [tmux](https://github.com/tmux/tmux/wiki)).

* _shell 1_ - to build and run ConfD example
* _shell 2_ - to make NSO directory and NED package, to run NSO itself
* _shell 3_ - to run ConfD CLI
* _shell 4_ - to run NSO CLI

### Copy/Paste and Output blocks

In this note you will find script and code examples, that can be directly pasted into shell or CLI terminal. We will use following block style for copy/paste ready
text:

```shell
confd --version
```

NOTE: make sure all commands have executed - confirm last command with _[ENTER]_,
if needed. If viewed on [GitHub](https://github.com), you may find following 
browser [extension](https://github.com/zenorocha/codecopy) useful, to get out of the box copy to clipboard button.

The output of the shell CLI commands or file content will be displayed
with following block style:

_output_
```
$ confd --version
7.4
```

## Prerequisites

* installed ConfD (with `examples.confd` package) - see `README` from ConfD package
* installed NSO - see `README` form NSO package
* build environment (`gcc`, `Makefile`) to build ConfD example (in Ubuntu use `apt-get install build-essential`)
* `xmlstarlet` (optional) for easier modification of `confd.conf` file (in Ubuntu use `apt-get install xmlstarlet`)

We will work in the `/tmp` directory. This is optional. Feel free to use any different directory
with read and write access.

To check ConfD and NSO installation, run following commands and check the output.

NOTE: version numbers may differ, according to your installation

run in the _shell 1_:

```shell
confd --version
```

_output_
```
$ confd --version 
7.4
```

run in the _shell 2_:

```shell
ncs --version
```

_output_
```
$ ncs --version 
5.5
```

NOTE: Many NSO commands start with `ncs` instead of `nso`. This is based on the name of NSO's
predecessor NCS.

## Build and start ConfD and example application

The `1-2-3-start-query-model` example can be found in the ConfD installation directory,
under subdirectory `examples.confd/intro/1-2-3-start-query-model`.

You can build and use example directly inside its directory, or you can copy it
to some other directory. The latter is better idea, since we will modify the `confd.conf` file.

E.g.

```shell
cp -r $CONFD_DIR/examples.confd/intro/1-2-3-start-query-model /tmp
cd /tmp/1-2-3-start-query-model
```

First, we will build the example. Go to the example directory and run
in the _shell 1_:

```shell
make clean all
```

_output_
```
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

NOTE: You can start and investigate example with `Makefile` target commands `make start`, `make cli-c`, etc. You can stop it with `make stop` (see example's `README` for details).

## Build and link NETCONF NED package for NSO

First, we need to set-up NSO directory and enter it. Go to the directory for
NSO project (e.g. `cd /tmp`) and run in the _shell 2_:

```shell
ncs-project create nsotest
cd nsotest
```

NOTE: the older way was to use `ncs-setup --dest nsotest`

Next, we make and build NETCONF NED from the example's YANG file(s) and link it to
the NSO packages. Run in the _shell 2_:

```shell
ncs-make-package \
        --no-java \
        --build \
        --vendor Cisco \
        --netconf-ned $CONFD_DIR/examples.confd/intro/1-2-3-start-query-model \
        dhcpned  #<1>
ncs-setup --package dhcpned --dest . #<2>
```

<1> create NETCONF NED from YANG files (do not use java binding), you can skip `--build`, but then you need to build
the package yourself with `make -C dhcpned/src all`
<2> add (link) NED to NSO packages

To check the package is linked, run in the _shell 2_:

```shell
ls packages
```

_output_
```
dhcpned
```

### Modify ConfD example ports

Before we start the ConfD example, we need to modify its `confd.conf`
to use different CLI and NETCONF SSH ports, so they do not conflict with NSO
CLI and NETCONF SSH ports (which are the same). Open `confd.conf` and:

* add `/confdConfig/cli/ssh/port` -> `13022` (original `2024`)
* modify `/confdConfig/netconf/transport/ssh/port` -> `14022` (original `2022`)

Corresponding `CLI` and NETCONF sections should look like:

_confd.conf_
```xml
<cli>
  <ssh>
    <port>13022</port>
  </ssh>
</cli>
...
<netconf>
  <transport>
    <ssh>
      <enabled>true</enabled>
      <ip>127.0.0.1</ip>
      <port>14022</port>  <1>
    </ssh>
  </transport>
  ...
</netconf>
```

<1> There will be other elements in the `<netconf>` section, only change the port element.

You can use following `xmlstarlet` commands, to modify `confd.conf` automatically.
Run in the _shell 1_ following commands:

```shell
export EXAMPLE_DIR=/tmp/1-2-3-start-query-model  #<1>
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig -t elem -n cli ${EXAMPLE_DIR}//confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig/conf:cli -t elem -n ssh ${EXAMPLE_DIR}/confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -s /conf:confdConfig/conf:cli/conf:ssh -t elem -n port ${EXAMPLE_DIR}//confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -u "/conf:confdConfig/conf:cli/conf:ssh/conf:port" -v 13022 ${EXAMPLE_DIR}/confd.conf
xmlstarlet ed -L -O -N conf="http://tail-f.com/ns/confd_cfg/1.0" -u "/conf:confdConfig/conf:netconf/conf:transport/conf:ssh/conf:port" -v 14022 ${EXAMPLE_DIR}/confd.conf
```

<1> set `EXAMPLE_DIR` as needed

To test the port modification works, start the example (in the _shell 1_) with `make clean all start` and
test NETCONF access. Run in the _shell 3_:

```shell
netconf-console --port 14022 --hello
```

NETCONF hello message should be returned.

To test SSH CLI access, run in the _shell 3_:

```shell
ssh admin@127.0.0.1 -p 13022
```

After the password is entered (default `admin`), ConfD CLI prompt appears.
Use `exit` command to exit the CLI.

## Create and configure ConfD device in NSO

Once we have everything set-up, we can start configuring the ConfD example as NSO device.

If you do not have ConfD example running from previous steps, start it in the _shell 1_:

```shell
make clean all start
```

after that, start NSO in the _shell 2_:

```shell
ncs --with-package-reload
```

next, we can enter NSO CLI and configure the device. In the _shell 3_ run:

```shell
ncs_cli -u admin -C
```

_output_
```
admin connected from 127.0.0.1 using console on pc-test
admin@ncs#
```
we can check our package (`dhcpned`) is correctly loaded, type in the _shell 3_:

```shell
show packages
```

_output_
```
admin@ncs# show packages
packages package dhcpned-nc-1.0
 package-version 1.0
 description     "Generated netconf package"
 ncs-min-version [ 5.5 ]
 directory       ./state/packages-in-use/1/dhcpned
 component dhcpned
  ned netconf ned-id dhcpned-nc-1.0
  ned device vendor Cisco
 oper-status up
```

finally, we enter config mode with command (in the _shell 3_):

```shell
config
```

### Configure `authgroup`

In order the NSO device can connect to the real NETCONF device, we need to
provide authorization details. This is done by linking NSO device with `authgroup`.
We configure `authgroup` in the config mode of NSO CLI. Type in (or paste into) the _shell 3_:

```shell
devices authgroups group devnetconf
default-map remote-name admin
default-map remote-password admin
commit
top
```

you can verify `authgroup` configuration with command

```shell
do show running-config devices authgroups group devnetconf
```

_output_
```
admin@ncs(config)# do show running-config devices authgroups group devnetconf
devices authgroups group devnetconf
 default-map remote-name admin
 default-map remote-password $9$zKHJM0RX2pfYCs6KL8pN2ZleIAQBt+wAJsuOwW+LRMY=
!
```

### Configure and synchronize NETCONF device

We have everything ready to configure NSO device and connect to ConfD (NETCONF) running
the example. Type in (paste into) the _shell 3_:

```shell
devices device EX_NETCONF
address 127.0.0.1
port 14022
authgroup devnetconf
device-type netconf ned-id dhcpned
state admin-state unlocked
commit
```

Once device is configured, we can try to synchronize it, so we know connection to
the ConfD NETCONF device is correctly established. Type in the _shell 3_:

```shell
ssh fetch-host-keys
sync-from
```

_output_
```
admin@ncs(config-device-EX_NETCONF)# ssh fetch-host-keys
result updated
fingerprint {
    algorithm ssh-rsa
    value 61:46:3d:74:9d:3c:0f:26:30:2b:2a:1a:0f:c6:3d:3e
}
admin@ncs(config-device-EX_NETCONF)# sync-from
result true
```

to see how NSO device is configured, type in the _shell 3_:

```shell
top
do show running-config devices device EX_NETCONF
```

_output_
```
admin@ncs(config)# do show running-config devices device EX_NETCONF 
devices device EX_NETCONF
 address   127.0.0.1
 port      14022
 ssh host-key ssh-rsa
  key-data "AAAAB3NzaC1yc2EAAAADAQABAAABgQDnUZtw+eyGJkhJIrMAEjDlUkQ2rlHbe5F22uFzZOB9\nM01m7CqSag+cL0vOHnnaHwPSTscoVYn+ygVcJEtCRy+mbqEnbDzTy9PA0i8/HX6tGOOhOhGF\n/DeFNTsVE9/Yd3a+piS4ZiIHPItiVHs181JkXEiLT3JK+5787GQ/0AxRnOwFDG4YbznlD6v5\npUzxkLqSf2ZND8HtsguCzbYM5O2kzChYll9Dzk5Q2CrSC3rGS3Wh4ZkdBNw5/4M0UR0KoVVV\nPFVdv9kEKT+9TiFsf/WtGaOCnxgWwhc4iXztz8PYg7uFTUBvYj+W/bJEaoUvHgsud6OlexXF\nDpMCWynW4Ky2FobsN7VLTsDWGpQwcP+rF2BD1zbaEZnVZZ86FMT+WUwoccaqFU9B2eyIfkAM\nMf5JM2207bbtxTs7EcGXwWz5lJTJ9Ywa9UBTRq9vHa1m3Kcp7Bwtt3kupV07oHIgoXH+F/P5\nETfMIz3kSsCkiCTB/+wsNt4sV1+I5fA5ih4L2TE="
 !
 authgroup devnetconf
 device-type netconf ned-id dhcpned-nc-1.0
 state admin-state unlocked
!
```

## Configure ConfD example in NSO

We have ConfD example attached to the NSO as device (name `EX_NETCONF`).
Now, we can configure example data model (directly in the NSO).
In the _shell 3_ type (make sure you are still in config mode):

```shell
top
devices device EX_NETCONF
config dhcp default-lease-time 700s
commit
```

If everything goes well, `Commit complete` message appears.

To verify the configuration was performed on the ConfD, open example CLI and check it.
In the _shell 4_ go to the example directory (e.g. `cd /tmp/1-2-3-start-query-model`) and
run following command to enter CLI:

```shell
make cli-c
```

_ConfD CLI is entered_
```
admin connected from 127.0.0.1 using console on pc-test
pc-test#
```

once in the ConfD example CLI, type (in _shell 4_):

```shell
show running-config dhcp
```

we can see the `default-lease-time` value configured in the NSO CLI is
applied and visible in the running configuration of the ConfD example device:

```shell
pc-test# show running-config  dhcp
dhcp default-lease-time 700s
```

In the similar way we can display the same data in the NSO CLI. Type in the _shell 3_:

```shell
top
show full-configuration devices device EX_NETCONF config
```

_output:_
```
admin@ncs(config)# show full-configuration devices device EX_NETCONF config
devices device EX_NETCONF
 config
  dhcp default-lease-time 700s
 !
!
```

## Stop and delete

To stop NSO, type in the _shell 2_:

```shell
ncs --stop
```

To stop ConfD example application, press in the _shell 1_ _[CTRL-C]_ or type in
the example directory (`/tmp/1-2-3-start-query-model`):

```shell
make stop
```

If needed, you can delete the example directory (`rm /tmp/1-2-3-start-query-model`) and the NSO directory (`rm /tmp/nsotest`).

## Conclusion

In this note we have learnt how to connect and configure NETCONF device in NSO.
To connect NETCONF device, we have to configure it in the NSO `device` data model.
No adaptation, filtering or bridging application is needed.
This is advantage of NETCONF standard.

We have used ConfD and ConfD example application (`intro/1-2-3-start-query-model`)
as a NETCONF device.
ConfD is NETCONF compliant and NSO is tested with ConfD. The steps described
in this note can be used with any device, which is NETCONF compliant.

NOTE: We have shown how to make NETCONF NED with commandline command `ncs-make-package`.
There are also tools that can be used for this, like [Pioneer](https://github.com/NSO-developer/pioneer) and
NETCONF NED Builder (successor to Pioneer)
<!-- Todo link to NED Builder>
