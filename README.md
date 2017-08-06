# iocage

iocage host install/config and jails installation

Uses the `iocage` ansible module from https://github.com/criecm/ansible-iocage
(embedded in the role's library/)

## Requirements

FreeBSD host system

## Role variables

(in defaults/main.yml)

* `iocage_zpool` (zroot): ZFS pool for iocage

* `fetch_properties` ({}): properties to be passed to `iocage fetch`
  example: { ftpfiles="base.txz", ftphost="my.own.freebsd.mirror" }

* `jail_list` ([]): list of jails dicts to be created on host, see below

### per-jail variables

(in vars/jail.yml)

* `tag` (no default, mandatory): human identifier, unique on host

* `hostname` (''): generated UUID if empty

* `ip4` (''): IPv4 addresse(s), same format as iocage: [ifaceN|]192.0.2.1[/24][,[ifaceN|]192.0.2.1[/24][,…]]
  * if 'iface|' is prepended, the IP will be added to the interface at jail boot
  * if no mask is given, IP will be /32

* `ip6` (''): IPv6 … same as above. (but default mask is /128, not /32 ;-P )

* `resolver` ('auto'): resolv.conf's content for the jail, with ';' instead of newlines
  (iocage will copy the host's one at jail boot if empty)

### resolver=auto logic

`resolver` will be auto-populated according to variables `search_domains` and `resolvers`
(here we have them in `group_vars/all.yml`). This will select search domain(s) and resolvers
 depending on jail's IP addresses.

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

## example playbook for host and one jail:

```
- hosts: realmachine
  roles:
    - iocage
  vars:
    jail_list:
      - { tag: myjail, hostname: myjail.example.org, ip4: 'bge0|198.51.100.0' }
```

## ansible-iocage module
update from source:

`git subtree pull -P roles/criecm.iocage/library/src/iocage https://github.com/criecm/ansible-iocage.git master`
