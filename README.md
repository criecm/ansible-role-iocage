# iocage

iocage host install/config and jails installation *on FreeBSD*
(this won't work as-is on FreeNAS, patches welcome ;)

Uses the `iocage` ansible module from
https://github.com/fractalcells/ansible-iocage
(embedded in the role's library/)

Adds created hosts in inventory (`add_host`) with a `iocage_host` variable
filled with host's `inventory_hostname` and an `iojails` group

## Role variables

(in defaults/main.yml)

* `iocage_zpool (zroot)`:
   ZFS pool for iocage

* `iocage_fetch_args ({}):`
   arguments to be passed to `iocage fetch`
   example: "-s ftp.local -d pub/FreeBSD/releases"

* `jail_list ([])`
   list of jails dicts to be created on host, see below

* `myjail ('')`
   if defined, run only this jail from `jail_list` (none if not found)

* `myjails([])`
   same as myjail, but fore more than one :)

*  `iocage_components (none)` - list
   if defined, only install these components

*  `iocage_enable_ssh (True)`
   Enable ssh in new jails

*  `iocage_release (uname -r)`
   The release you need

* `jail_init_role ()`
  Role to be imported to initialize new jail

* `iocage_use_pkg (True)`
  Will install iocage from packages, or from git if False

### per-jail variables

(in vars/jail.yml)

* `name` (no default, mandatory): human identifier, unique on host

* `hostname` (''): generated UUID if empty

* `ip4` (''): IPv4 addresse(s), same format as iocage: [ifaceN|]192.0.2.1[/24][,[ifaceN|]192.0.2.1[/24][,…]]
  * if 'iface|' is prepended, the IP will be added to the interface at jail boot
  * if no mask is given, IP will be /32

* `ip6` (''): IPv6 … same as above. (but default mask is /128, not /32 ;-P )

* `resolver` ('auto'): resolv.conf's content for the jail, with ';' instead of newlines
  (iocage will copy the host's one at jail boot if empty)

* `properties` ({}):
   Dict for any iocage jail properties available

* `authkeys (/root/.ssh/authorized_keys)`
  File to copy as /root/.ssh/authorized_keys in jail

### resolver=auto logic

`resolver` will be auto-populated according to variables `search_domains` and `resolvers`
(here we have them in `group_vars/all.yml`). This will select search domain(s) and resolvers
 depending on jail's IP addresses.
if `dns64_resolvers` is a list and jail has no ip4 addresses, those are the resolvers used.

```
# if ip is in 'network', 'domain' is added
search_domains:
  - { network: '192.0.2.0/24', domain: 'our.example.net' }
  - { network: '198.51.100.0/24', domain: 'ryd.example.org' }
  - { network: '2001:0DB8:fe43::/32', domain: 'ipv6.example.org' }
  - { network: '0.0.0.0/0', domain: 'example.com' }

# if ip is in 'network', 'ip' is added to resolvers
resolvers:
  - { network: '192.0.2.0/24', ip: 192.0.2.1 }
  - { network: '198.51.100.0/24', ip: 192.0.2.1 }
  - { network: '2001:0DB8:fe43::/56', ip: 2001:0DB8::1 }
  - { network: '0.0.0.0/0', ip: 8.8.8.8 }
  - { network: '::/0', ip: 2620:0:ccc::2 }
```

## example playbooks:

### A jailhost with two jails:

```
- hosts: realmachine
  roles:
    - criecm.iocage
  vars:
    jail_list:
      - { name: myfirstjail, hostname: myfirstjail.example.org, ip4_addr: 'bge0|198.51.100.0' }
      - { name: mysecjail, hostname: mysecjail.example.org, ip4_addr: 'bge0|198.51.100.8' }
```

### a playbook snippet to create/register the jail before working on it

```
- hosts: realmachine
  roles:
    - criecm.iocage
  vars:
    # here jail_list can be in the inventory/host_vars/realmachine.yml
    myjail: myfirstjail

- hosts: myfirstjail
  roles:
    - criecm.apache
  […]
```

## ansible-iocage module
update from source:

`git subtree pull -P roles/criecm.iocage/library/src/iocage https://github.com/criecm/ansible-iocage.git master`
