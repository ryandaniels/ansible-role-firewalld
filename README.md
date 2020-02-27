# ansible-role-routes_management

Add local firewall rules on server via firewalld.  
Why another Ansible role to manage firewalld? The role does everything in "offline" mode. Even creating a new service works offline.  
So there should be no issues when firewalld starts, and blocks EVERYTHING by default! (Make sure you test in non-production first, I cannot make any guarantees)
This supports adding custom firewalld "services" into zones with custom ports. Also can add or remove ports for built-in services provided by firewalld.  

Zones: Add IP ranges to Zones.  To remove, change state to `disabled`. If only one IP range needs to be disabled, create a new zone entry for just that IP range.  
Services: `disable` a service will remove it from that zone as specified in the variables. Don't just delete the config to remove it!  
Ports: To remove a port, delete it from the config (under port_protocol).  

Since VMs could have different interfaces (eth0/eth1 etc.), interfaces aren't managed.

Be careful, this will remove and add firewalld rules on the OS. Use with caution.  

## Distros tested

* CentOS: 7.6, 7.7

## Dependencies

* firewalld

Tested with version 0.6.3 (Latest available in CentOS)

## Default Settings

* Enable debug

```yaml
debug_enabled_default: false
```

* Proxy (Needed when installing required packages if behind a proxy)

```yaml
proxy_env: []
```

## Example config file (inventories/dev-env/group_vars/all.yml)

From the example below:  
IP ranges will be added/removed from a zone:  

* `192.168.22.64/26` and `192.168.23.64/26` will be added to the zone `internal`  
* `192.168.32.64/26` and `192.168.33.64/26` will be removed from the zone `internal`  

Custom services will be created with port(s) specified, and added to a zone:  

* `app123-public` service is created with port/protocol `5000/tcp`, and added to zone `public`.  
* `app123-internal` service is created with ports/protocol `8080/tcp`, `9000/tcp`, and added to zone `internal`.  
* `zabbix-agent` service (which is a built-in service in firewalld) is not using the default port (10050) since it's not in the config, `10050/tcp` is removed. Instead using port/protocol `3333/tcp`, and added to zone `public`.  (Note: The port doesn't need to be commented out).
* `openvpn` service (which is a built-in service in firewalld) is using the default port/protocol, and added to zone `public`. Since this service is built-in, you don't need to specify the port. But to be safe, you should still specify it.  

```yaml
---
rhel_firewalld_zone_source:
  - zone: internal
    state: enabled
    source:
      - "192.168.22.64/26"
      - "192.168.23.64/26"
  - zone: internal
    state: disabled
    source:
      - "192.168.32.64/26"
      - "192.168.33.64/26"

rhel_firewalld_custom_service:
  - name: app123-public
    zone: public
    # state: disabled
    state: enabled
    description: app123 firewall rules for public zone
    port_protocol:
      - 5000/tcp
  - name: app123-internal
    zone: internal
    # state: disabled
    state: enabled
    description: app123 firewall rules for internal zone
    port_protocol:
      - 8080/tcp
      - 9000/tcp
  - name: zabbix-agent
    zone: public
    # state: disabled
    state: enabled
    port_protocol:
      # - 10050/tcp
      - 3333/tcp
  - name: openvpn
    zone: public
    state: enabled
```

## Example Playbook routes_management.yml

```yaml
---
- hosts: '{{ inventory }}'
  become: yes
  vars:
    # Use this role firewalld
    rhel_firewalld_managed: true
  roles:
  - firewalld

```

## Usage

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7" -i hosts-dev
```

Skip installing packages (if known already there - speeds up task)

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=all-dev" -i hosts --skip-tags=rhel_firewalld_pkg_install
```

Show more verbose output (debug info)

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 debug_enabled_default=true" -i hosts-dev
```

Start firewalld service at end of role

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 rhel_firewalld_start=true" -i hosts-dev
```

### TODO

* [x] Get working with firewalld started or stopped
* [ ] Confirm this everywhere needed:   notify: Reload firewalld
* [ ] Test this works if firewalld service is running
* [ ] Build travis tests for many scenarios
* [ ] Improve/shorten changing/removing a port
* [ ] --diff doesn't show much since using many commands. Show what's going to happen and add pause when is debug enabled?
