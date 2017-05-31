# wrex
Small app for network device testing with [Cisco TRex](https://trex-tgn.cisco.com)
> App is still in active development state now and some features might not work or work improperly

## Requirements
App is written on _python_ and needs **python 3** _(3.5 and above would be better)_. And you should install several **python modules** from requirements.txt _(see [Install](#install) notes below)._

App _works on **NIX_ systems _(due some Windows limitations, see [limits](#operation-system-limits) below)._

**Redis** server is required, because some elements using queue feature. You may _install new_ or _use existing_ redis server.

**Supervisord** for managing app parts startup/shutdown.

**Cisco TRex** installed locally or separately

## Install
Download last [app release](https://github.com/0xck/diploma/releases) and unarchive it into directory you plan to use for app.

### Python
Make sure you have got installed _python 3_, otherwise just use system software manager _(e.g. apt, yum, pkg)_ or download one from [official site](https://www.python.org/downloads/) and install. App tested on 3.5 version and should work on other python 3 releases as well.

###### Python modules
Install all python modules listed in _requirements.txt:_
- Flask
- Flask-SQLAlchemy
- Flask-Script
- Flask-WTF
- rq
- timeout-decorator

For this purpose, you may use any module manager: _pip, easyinstall, pypm._ For example pip manager is used. If module `pip` is not installed on your system just install one using system software manager _(e.g. apt, yum, pkg)._ And go to app directory and execute command:

`python3 -m pip install -r requirements.txt` or if your system uses python3 by default `pip install -r requirements.txt`

---
**Note.** Some systems do not allow you install part modules using module manager because the modules already exist in system software repository in that case you have several ways:
- using sudo/root privileges for force installation
- install modules with system software manager

_Actually there is one more way is using virtual python environment, but it makes additional complication in app usability that is the reason I do not recommend this way for people who do not what is it and how do use it._

### Redis server
>Redis queue feature is used by some components of app for making different tasks in background.

If you have already installed reis server you may skip this paragraph. In future, you need redis URI, base and credentials for setup app _(see [App configuration](#app-configuration) notes)._

If you do not have installed redis server you should install one using system software manager _(e.g. apt, yum, pkg)._ In most cases generic settings which software manager provides for redis is enough for using one.

---
In case you would like to use _supervisor_ for managing redis startup/shutdown you should change redis config file. Copy current redis config file to new file (e.g. `/etc/redis/redis.conf` to `/etc/redis/redis_supervisor.conf`). Open new file with text editor and change string `daemonize yes` to `daemonize no`, write changes. On some system you have to change new config owner and permissions, on `redis:redis` and `600`.

Later you specify new config file as redis config _(see [App configuration](#app-configuration) notes)._

### Supervisord
>Supervisord is used for manage app components startup/shutdown in order to do not start all of them manually at separate (pseudo)terminals.

###### Installation
If you do not have a supervisor just install one using system software manager _(e.g. apt, yum, pkg),_ it is better way because this allows you escape different problems with permissions and supervisor configuration. You may install one as python module, but be careful one uses _python version 2.7_ you should install it first.
###### Configuration
If you installed supervisor using system software manager one has generic config. In case other ways of installation you may use `echo_supervisord_conf` for generate simple config, change it and put into system configuration directory e.g. `/etc/supervisor/` (for Ubuntu Linux). Last step is not necessary you may use any directory and provide path to the config using key `-c`. Use supervisord [documentation](http://supervisord.org/) for details setup and startup.

For example, supervisord config _(from Ubuntu Linux)_ is given below:
```ini
[unix_http_server]
file=/var/run/supervisor.sock
chmod=0700
[inet_http_server]
port=*:9001
username=admin
password=admin
[supervisord]
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisord.pid
childlogdir=/var/log/supervisor
logfile_maxbytes=10MB
logfile_backups=10
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface
[supervisorctl]
serverurl=unix:///var/run/supervisor.sock
[include]
files = /etc/supervisor/conf.d/*.conf
```
Pay attention on `[inet_http_server]` section one allows you manage supervisor via web interface. In example above one listens TCP port 9001 and reachable via all host interfaces. For protection, simple login/password authentication may be used.

Section `[include]` defines place for additional config one will be needed us when we prepared wrex supervisor config _(see [App configuration](#app-configuration) notes)._

### Cisco TRex
>Cisco TRex is used as main network testing software.

###### Setup and configuration
This product is available in [repository](http://trex-tgn.cisco.com/trex/release/). Download latest release and setup it using [official documentation](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_first_time_running). One may be installed on several Linux distro, please check capability [here](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_supported_versions) as bare-metal or [VM instance](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_trex_on_esxi). Please check [hardware recomendations](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_hardware_recommendations) in order to make sure TRex will function properly.

For example:
TRex config for _2x1G interfaces and L3_ is given below:
```yaml
- port_limit      : 2
  version         : 2
  interfaces      : ["03:00.0","0b:00.0"]
  port_bandwidth_gb : 1
  port_info       :
          - ip         : 192.168.1.10
            default_gw : 192.168.1.1
          - ip         : 10.0.1.10
            default_gw : 10.0.1.1
```

Below TRex config for _2x10G interfaces L2_ is presented:
```yaml
- port_limit      : 2
  version         : 2
  interfaces      : ["03:00.0","0b:00.0"]
  port_bandwidth_gb : 10
  port_info       :
          - dest_mac        :   [0x0,0x0,0x0,0x2,0x0,0x0]
            src_mac         :   [0x0,0x0,0x0,0x1,0x0,0x0]
          - dest_mac        :   [0x0,0x0,0x0,0x4,0x0,0x0]
            src_mac         :   [0x0,0x0,0x0,0x3,0x0,0x0]
```
Item `interfaces` has to contain list of proper NICs ID use `dpdk_setup_ports.py -s` with sudo/root privileges for finding your values, see [manual](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_identify_the_ports).

Item `port_info ` must be changed on your values.
###### Daemon operations
After TRex setup was complited go to TRex directory and start a daemon `trex_daemon_server start` with sudo/root privileges. Also you can check status `trex_daemon_server show`, restat `trex_daemon_server restart` and stop daemon `trex_daemon_server stop`, use sudo/root privileges for all of them.

By default daemon listens _TCP port 8090_ make sure this port is available for wrex host. Also for stateless TRex mode _TCP ports 4500 and 4501_ have to be available for wrex host. More about stateful/stateless modes see at [documentation](https://trex-tgn.cisco.com/trex/doc/trex_stateless.html#_stateful_vs_stateless).

## App configuration

## Usage

## Limits
>There are some limits for using app in whole. They are described below.

#### Operation system limits
App works only on *NIX systems, because some app components _(RQ workers)_ use `fork()` that is not implemented on Windows.

#### TRex limits
Current TRex releases have some limits _got from [here](https://communities.cisco.com/community/developer/trex/blog/2017/03/29/how-trex-is-used-by-mellanox)_:
TRex Missing Functionality:

The following items are lacking from TRex in our view:
1. TCP Stack
2. Simulation of packet loss and simulation of retransmission handling.
3. Routing Emulation Support - BGP/OSPF/ARP/VRRP/DHCP/PIM
4. Stateless GUI Gaps:
    - Build/send L2/L3 frames of control protocols, such as STP/LACP/OSPF/BGP.
    - L2 protocol emulation (LACP, STP and IGMP) support.
    - Incremental support on the following fields - VLAN/PRI/TCP Port Number/MTU.
    - Capture capabilities

Also one has [hardware](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_hardware_recommendations) and [software](https://trex-tgn.cisco.com/trex/doc/trex_manual.html#_hardware_recommendations) limits.

#### App limits
App has several limits, some of them will be fixed in future releases. All TRex limits affect app either.

###### General
Are not supported:
- DB engines _(only SQLite)_
- multiuser
- authentication
- items restoring
- online statistic during test
- any management for TRexes or devices
- notification

###### TRex
In whole app is not web interface for TRex, it uses TRex for making tests, gathering information and presenting test result. That means not all TRex features are supported. Are not supported:
- NAT
- latency test
- IPv6
- more than 2 ports
- more than one stream in stateless
- patterns with plugins
- L2 patterns
- manually history size defining for stateful
- multiinstance
- test duration more than 86400 seconds
- any traffic pattern constructors
- more than 1 test per TRex instance
- connection to TRex via IPv6 and DNS
- full TRex port statistic
- work with traffic capture
- changing parameters during test for stateless

###### Device
Are not supported:
- determination realy device status _(only ICMP availability)_