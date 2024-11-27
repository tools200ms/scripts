# NFT List

NFT List provides the functionality for blocking and allowing traffic based on: 
* Domain names
* IPv4 and IPv6 hosts and networks
* Mac addresses

It extends [Linux NFT (Nftables)](https://en.wikipedia.org/wiki/Nftables) firewall and uses [NFT Sets](https://wiki.nftables.org/wiki-nftables/index.php/Sets) to "attach" a specific list. 

It implements the "available-enabled pattern" configuration (refer to the [Configuration section](#configuration)), commonly used in Apache or Nginx, allowing administrators to manage system conveniently.

**This tool enables administrators to separate the NFT firewall configuration logic (defined in `/etc/nftables.*`) from its data - that is access/deny lists.**

`NFT List` resolves domain names by default via CloudFlare's `1.1.1.1` DOH (DNS over HTTP(s)). The tool is implemented in Python.

**NOTICE: As the firewall is a critical security component, it is essential to review this documentation thoroughly. It has been designed to be concise, with important details emphasized in bold for clarity.**

## Table of contents

- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
  - ['avaliable' and 'enabled' locations](#avaliable-and-enabled-locations)
  - ['included' location](#included-location)
- [Firewall side (NFT sets)](#firewall-side-NFT-sets_)
  - ["Non-atomic" issue](#non-atomic-issue)
  - [Set Corks convention](#set-corks-convention)
  - [Allow/deny awareness](#allowdeny-awareness)
  - [Supported NFT types](#supported-nft-types)
  - [DNS quering](#dns-quering)
  - [File format for '.list'](#file-format-for-list)
    - [Section marks](#section-marks)
    - [Directives](#directives)
      - [@include](#include)
      - [@query](#query)
      - [@onpanic](#onpanic)
      - [@timeout](#timeout)
    - [Comments](#comments)
- [Nftables set flags and 'NFT List' behaviour](##nftables-set-flags-and-nft-list-behaviour)
  - [Flag timeout](#flag-timeout)
  - [Flag interval](#flag-interval)
- [Refreshing lists](#refreshing-lists)
- [Troubleshooting](#troubleshooting)
- [Summary](#summary)
- [Useful links](#useful-links)


## Requirements

The `NFT List` requires Python 3 and a configured NFT firewall. Administrators define NFT sets within the firewall, which are populated with the appropriate lists by `NFT List`.

**Short NOTICE: In this document network resources handled by `NFT List` (domain names, IPv4/IPv6 and macs) will be referred simply as 'resources'.**

## Installation

Project is available in PyPI, to install it use the following commands (root is required): 
```commandline
# Install Python package from repository: 
pip install nftlist
# initialise configuration: 
nftlist init --download
```
Option `--download` downloads pre-defined lists that can be used for restricting firewall to traffic to a specific popular servers, such as GitHub or Cloudflare. The complete list can be seen 
[here](https://github.com/tools200ms/nftlist/tree/release/lists).

## Configuration
`Nft List` configuration is in `/etc/nftlists/`. The file-directory structure is as follows: 
```bash
# tree /etc/nftlists/
/etc/nftlists/
├── available
│   ├── inbound_filter.list
│   └── outbound_filter.list
├── enabled
│   └── inbound_filter.list -> ../available/inbound_filter.list
└── included
    ├── _local
    │   ├── broadcast-ipv4.list
    │   ├── localnet-ipv4.list
    │   └── localnet-ipv6.list
    ├── _user
    ├── bitbucket
    │   ├── bitbucket-all-inet.list
    │   ├── bitbucket-out-inet.list
    │   └── bitbucket-out_email-inet.list
    ├── cloudflare
    │   ├── cloudflare-ipv4.list
    │   └── cloudflare-ipv6.list
    ├── github
# .... GitHub IP's *.list files
    └── nitropack
        └── nitropack-ipv4.list
```
### `avaliable` and `enabled` locations
**Directory `available` is designated for user defined access/deny lists.** 
Directory `enabled` can only be used for linking to lists in `available`. **As the pattern suggests, `NFT List` loads the lists solely from the `enabled` directory**.

List files are required to have a `.list` extension, and link files in `enabled` must share the same basename as their corresponding source files. `NFT List` does not perform recursive searches; therefore, paths like *enabled/directory/link_to_file_in_available.list* are ignored with a warning.

Files `*.list` contain defined resources, potential comments, but also `section marks` and `directives` (details in section ["File Format"](#file-format)). One of the directives is `@include` that allows on inclusion of pre-defined lists from services such as GitHub. These lists are located under `included` directory.

### `included` location

**Location `included/_user` is intended for dropping a large 'user-defined' lists'. That are later 'included' with `@include` directive (in `enabled/` defined list). To activate a list, it must be enabled by creating a symbolic link in the enabled directory.**

**User defined large and dynamic lists located under 'included/_user' can have following extensions: .txt, .bz2, .gz, .xz.**

**`NFT List` accepts compressed lists, but extension must fit to used compression algorithm.**

## Firewall side (NFT sets)
On NFT side (by default in `/etc/nftables.nft` and `/etc/nftables.d/`) administrator defines firewall configuration. Configuration shall include sets, where `NFT List` attaches appropriate resources. See examples in [Examples section](#examples).

`NFT List`  **does not modify the firewall configuration;** it strictly **manages ONLY elements** within NFT sets.


### "Non-atomic" issue
Advantage of NFT over IPtables is 'atomic' operation. The command `nft -f <filename>` loads configuration 'on a side'. When configfile is successfully validated NFT switches in an atomic manner to a new setup. This means there is no a single moment when firewall is partially configured.
As `NFT List` acts after `nft` command, this might bring following security issues: 
- very short time of security violation (such as open access from denied IPs)
- permanent time of security violation

The first case is difficult to exploit (but still possible), it is due to a short time that is after NFT setup, but before lists are loaded. The second that is a very serious is the result of no-loading `NFT List` due to some kind of system error. 

### Set Corks convention
The solution is to *cork* a sets in NFT configuration with `0.0.0.0/0` and `::/128` masks. Hence, Nftables treats all IP's as blacklisted. **After wards, while `NFT List` loads the list, also removes `0.0.0.0/0` and `::/128` corks**. 

**`NFT List` at launch traverses the list and rules to issue warnings about potentially 'opened' rules.**

**Notice**, there are only **deny 'lists' that must be 'corked'** in oppose to **access lists that can be empty**.

This has been called **'cork' convention**, below there is a snippet demonstrating idea: 
```
table inet filtering_machine {
    set allow_ip_list {
        type ipv4_addr;
        flags timeout;
    }

    set ban_ip_list {
        type ipv4_addr;
        elements = { 0.0.0.0/0 }
    }
    
    table inet machine {
      .......
      chain input {
        type filter hook input priority 0; policy drop;
        ........
        # for 'allow' list ampty set is OK
         iifname eth0 tcp dport 22 \
                        ip  saddr @allow_ip_list ct state new accept
    }
    .......
    chain forward {
      type filter hook forward priority 0; policy allow;
        ........
        # more secure is to "ban all" with 'daddr 0.0.0.0/0 drop'
        ip daddr eth1 oifname eth0 \
              ip daddr @ban_ip_list drop
        ip saddr $LAN_NET iifname $LAN_NIC accept
    }
}
```
In example above, by default we tread all forwarded traffic as banned. And also, by default we don't let any IP establishing connection at input.

Notice, administrators can add extra rules to open access from administrative IPs, that is fixed it Nftables configuration. It ensures access for them in the case if NFT List is not loaded.

### Allow/deny distinction
Policy of using cork addresses let `NFT List` to distinct which list is allow, which deny. This is useful with 'panic' option described in ["NFT List options"](#nft_list_options)z.

### Supported NFT types
`NFT List` accepts a following element types: 
- `ipv4_addr`
- `ipv6_addr`
- `ether_addr`

**NOTICE: In NFT there is no type that would match both: `ipv4_addr` and `ipv6_addr`. Therefore, one set can not hold mixed IPv4 and IPv6 elements.**

Details about Set types can be found in ["Named sets specifications (nftables.org site)"](https://wiki.nftables.org/wiki-nftables/index.php/Sets).

## File format for `.list`
File format of `.list` is as follows: 
```
# /etc/nftlists/avaliable/blacklisted_domain.list

=set inet filter blacklist

bad-domain.xxx.com # don't go there
your-bank.trustme-login.space # no comment
# aha ...
specialoffernow.info

@include _user/large_blacklist
=end
```

Details on the syntax and structure are provided below.

### Section marks
`Section mark` can be thought of as a procedure, or function, while Nftables set as function's prototype. `Section mark` declaration refers unambiguously to a specific set with a following syntax:

> **=set \<family\> \<table name\> \<set name\>**

Section must be ended with the following mark: 

> **=end**

In between **=set** and **=end** user defines resource list, but also `directives` specialising list properties.

### Directives
Directives start with **\@** and are used to specify properties (overwrite defaults) and extend capabilities. Directives are defined inside section. Directives must be defined right after `=set`, except `@include` that can be located anywhere (mixed with resource list).

#### @include
**\@include \<file name\>**

This directive extends resource definition by external list. It is used to include well known internet resources, such as Github IP's, but also large or being the subject of regular updates user defined lists.

**\<file name\>** is a path to file relative to `/etc/nftlists/included`. See details about [configuration directories](#configuration).

#### @query
Query directive instructs `NFT List` to also query domain subdomains, the syntax is as follows: 
> **@query \<subdomain 1\> \<subdomain 2\>**

Example usage is: 
**@query www mail**

#### @onpanic
Directive `onpanic` overwrites default onpanic behaviour that is determinated by `NFT List` basing on [Cork convention](#set-corks-convention) and rule policies where set has been used. Syntax is as follows: 

**\@onpanic keep|drop|rise**

**keep** - makes `NFT List` to 'keep' the list if 'panic' signal is risen.
**drop** - makes `NFT List` to 'discard' the list if 'panic' signal is risen.
**rise** - makes `NFT List` to 'add' the list to the set if 'panic' signal is risen. Important notice is that the list will not be loaded onder 'normal' circumstances. List with value 'rise' of 'onpanic' is loaded only in the case of 'panic' signal.

`panic` signal is rised with: **nftlist panic** command.

#### @timeout
Directive 'timeout' can be used only if Nftables set has 'timeout' flag defined. `@timeout` overwrites a default timeout. the syntax is as follows: 
> **@timeout <time in format: ?h?m?s>**

Example usage: 
> **@timeout 24h15m, @timeout 30s, @timeout 1h**

### Comments
Comments in `.list` are marked with `#` symgol. Comments can takeentire line, as well as be placed at the end of command: 
```
# This is the comment
=set inet filter allow

# ranges
192.168.0.100-192.168.0.200
192.168.3.50-192.168.3.100

10.0.2.2  # allow this IP
#end
```

## `NFT List` Characteristics

### DNS querying
If set has been defined of a `type ipv4_addr` NFT-List will resolve `A` DNS records to acquire IP.
If set is of `type ipv6_addr` `AAAA` DNS records are queried.


### Nftables set flags

Nftables sets can be featured by flags that specifies more precisely behaviour and format of an elements. Below sub-sections describe how `NFT List` behaves if certain flags are set.

#### Flag timeout
Specifies timeout when a set element is set for expiration. 

**Note** that various elements within one set can have a different timeouts.

If `timeout` flag is defined `NFT List` sets default timeout that is 24h15m, or the time that has been defined by `@timeout` directive.

* *timeout* - `NFT List` resolves IP addresses and adds them to 'timeout set' setting up a default timeout that is 3 days. If the list is refreshed, timeout is updated. If given domain name does not resolve to a certain IP anymore, it will simpli expire and eventually disappear from the set.

#### Flag interval
Set elements can be intervals, that is IP addresses with network prefixes or address ranges in format: *<first addr>-<last addr>*, e.g. `192.168.100-192.168.199`.

`NFT List` extends accepted format by network prefixes and address ranges.

#### Flag auto-merge
`NFT List` acknowledges `auto-merge` set flag. `auto-merge` comes together with an `interval` flag. If flag is on, IP addresses or networks will be merged into intervals if suitable. For instance, adding networks `10.2.0.0/16` and `11.3.0.0/16` into `auto-merge` set results in one entry that is: `10.2.0.0/15`. 
If there is the chance that IP addresses might 'stick' or IP ranges might have a common part, `auto-merge` would improve filtering efficiency after all.

## Usage
Once when `nftlist` Python package is installed systemd or OpenRC service is added to init system and enabled. To star service use: 
```bash
service nftlist start
```
It can be used from commandline, for details check with: 
```bash
nftlist --help
```

### Refreshing lists
One of the aims for `NFT List` is keeping resource list upto date. In the case of domain names, resolved IP's should reflect actual state of DNS system. As the IP set in `A` and `AAAA` records can change, `NFT List` checks DNS periodically for changes.

See ["'Refresh' option in Usage section"](#periodic_refresh).

Algorithm for refreshing lists vary depanding if timeout flag is set, or not.
#### Refreshing 'no-timeout' sets
If Set is a 'typical', 'no time-out' set, refresh operation will synchronise actual list with set elements. After all there will be 1-to-1 mapping. IP's that has been deleted from list, will be deleted from set, new IP's will be added.

In the case of domain names IP's that has no reference from DNS will be removed, new IP's will be added.

#### Refreshing 'timeout' sets
If time-out is set, refreshing is much faster as `NFT List` adds a new resources with a default or `@timeout` defined timeout. If the IP already exists its timeout is reset to default or `@timeout` defined.
Elements that does not exist anymore eventually simply expire. This is more suitable for a large lists, as there is no need for exact comparison to find expired element.


### Periodic refresh
`pip install nftlist` also adds script for daily cron tasks. 
`A` or `AAAA` records can change over time, therefore firewall shall be updated periodically.
It is advised to add to daily cron `service nftlist reload` so configuration shall be keep in
sync with DNS entries.


### Panic signal
In the case of signal that securit branch had occured NFT List can aplly 'panic' policy. This will drop elements from allow lists, replace deny with 'any host'(0.0.0.0/0 and ::/128) addresses and apply `@onpanic` directives.

### Manual run
By default, configuration from `/etc/nftlists/available/` is loaded, however it can be overwritten:
```bash
nftlist update /etc/my_list.list
# or
nftlist update /etc/my_list_directory/
# additionally include directory can be indicated:
nftlist update /etc/my_list.list --includedir /etc/incl_lists/
# or shortly:
nftlist update /etc/my_list.list -D /etc/incl_lists/
```

## Troubleshooting
In the case of troubles you run:
```
nftlist --verbose
```
this will issue verbose messages. If it does not help analyze what exactly is under the hood:

```bash
# Stop firewall
service nftlist stop
service nftables stop

# checkout defined rules
# it should be empty
nft list ruleset

# start firewall
service nftables stop



# start nftlist
service nftlist start

# and verify if sets has been updated
nft list ruleset
```
also you can:
```bash
# analyze configuration
nft list ruleset

# List all sets
nft list sets

# Add set elements manually
nft add element inet my_table allowed_hosts { 172.22.0.2 }
```

NFT provides [counters](https://wiki.nftables.org/wiki-nftables/index.php/Counters) and [log capabilities](https://wiki.nftables.org/wiki-nftables/index.php/Logging_traffic) it can give you some information if traffic is passing through a certain firewall rule or hook.
In tha case of troubles tools such as `tcpdump`, `WireShark`, `conntrack` come in handy.

## Examples
Simple example can be found in: [example1.1-workstation.nft](examples/example1.1-workstation.nft) and `NFT List`
[example1.1-workstation.list](examples/example1.1-workstation.list).

Once when NFT with this configuration is loaded (note `flush ruleset` - that purges all previous settings) `allowed_hosts` is empty.
Therefore, no other host is allowed to let in, access can be open from command line:
```bash
nft add element inet my_table allowed_hosts { 172.22.0.2 }
```
or `NFT List` can do this for you. Define file such as:
```
# /etc/nftdef/allowed_hosts.list
# host that are allowed for connecting this box

=set inet my_table allowed_hosts # load bellow addresses into 'allowed_hosts' set
172.22.0.2 # Peer 2
=end
```

and load:
```bash
nftlist.sh update /etc/nftdef/allowed_hosts.list
```
`NFT List` comes with OpenRC init script to load settings a boot time, and cron script for periodic reloads. More in [Part II: Installation and Configuration](#part-ii-installation-and-configuration).


#### NFT sets

NFT comes with [NFT sets](https://wiki.nftables.org/wiki-nftables/index.php/Sets) that allow on grouping elements of the same type together.
Example definition of NFT set looks as follows:
```
table inet user_defined_table_name {
       set user_defined_set_name {
               type ipv4_addr;
               flags timeout, interval;
               auto-merge;

               elements = { 10.0.0.0/8 }
       }
}
```

### White List example

One of the example of whitelisting is to allow outgoing connections to be only possible with a
very limited number of servers. This can be a list of a domain names that are used for system updates.
The configuration might look as bellow:
```
# file: /etc/nftlists/available/repo_devuan.list

# Devuan repositories
deb.devuan.org
deb.debian.org
```
```
# file: /etc/nftlists/enabled/

@set inet filter allow_outgoing
@include repo_devuan.list
```
We can whitelist more resources:
```
# file: /etc/nftlists/enabled/

@set inet filter allow_outgoing
@include repo_devuan.list

10.2.0.100
example.com
```
Note that by splitting configuration to a set of files it become more manageable.
Also note, that you can mix domain names with IP address.

NFT rule bound to `allow_outgoing` set would be defined in forward chain. The default policy for that chain might be `drop`, the
rule might be as follows:
```
iifname br0 oifname eth0 ip daddr @allow_outgoing tcp dport { 80, 443 } accept
```

# TODO
* Move `/etc/nftlist/included` to more aproperiate location such as `/var/lib/nftlist/`
* Compress "include" resources, think about checksum chek
* Implement support for `constant` flag
* Extend `NFT List` by NFT maps, and vmaps
* Replace `includes/_local/*` path with a keyword that does include appropriate RFC standard.
* Add an interactive mode that would work as follows: 
```bash
# nftlist
\# nftlist update -i
Following updates has been found
Id:    action:
 1     add 4 IPv4 elements
        to 'blacklist' set of 'filter' inet table

proceed with update?
      yes    / no / details / help or (y    /n/d/h) - if yes,     update all
  or  yes Ids/ no / details / help or (y Ids/n/d/h) - if yes Ids, update chosen set
 :
```

# Summary

`NFT List` can be used in: 
- virtualize environments with containerization/virtualization technics where running VMs can be limited to exact network resources that is needed
- IoT devices to protect from un-authorized access
- routers that can be extended with advanced filtering capabilities

This tool has been developed as a lite, robust and straightforward solution for a various firewall configuration cases.

# Useful links

**HowTo:**

* Nftables Wiki [here](https://wiki.nftables.org/)
* Gentoo Nftables guide (nice and compact) [here](https://wiki.gentoo.org/wiki/Nftables)
* and Arch Linux NFT (alike Gentoo's wiki, but nothing about howto compile kernel :) [wiki](https://wiki.archlinux.org/title/Nftables)

**Theory:**

* Netfilter framework [at Wikipedia](https://en.wikipedia.org/wiki/Netfilter)

**Administration**

* Using iptables-nft: a hybrid Linux firewall [at Redhat.com](https://www.redhat.com/en/blog/using-iptables-nft-hybrid-linux-firewall)
* [serverfault: nftables: referencing a set from another table](https://serverfault.com/questions/1145318/nftables-referencing-a-set-from-another-table)

**Bad addresses databases**

* Phishing Database: [github: mitchellkrogza/Phishing.Database](https://github.com/mitchellkrogza/Phishing.Database/)

**Programming**

* fw4 Filtering traffic with IP sets by DNS [Openwrt.org](https://openwrt.org/docs/guide-user/firewall/filtering_traffic_at_ip_addresses_by_dns)
* Python Nftables tutorial: [GitHub](https://github.com/aborrero/python-nftables-tutorial)

**Othe links**

* nftlb - nftables load balancer [GitHub](https://github.com/zevenet/nftlb)

