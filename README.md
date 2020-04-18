# ansible-role-firewalld

Add local firewall rules on server via firewalld.  
Why another Ansible role to manage firewalld? The role does everything in "offline" mode. Even creating a new service works offline.  
So there should be no issues when firewalld starts, and blocks EVERYTHING by default! (Make sure you test in non-production first, I cannot make any guarantees)
This supports adding custom firewalld "services" into zones with custom ports. Also can add or remove ports for built-in services provided by firewalld.  
Removes unwanted default services from public/internal zone.  

Zones: Add IP ranges to Zones.  To remove, change state to `disabled`. If only one IP range needs to be disabled, create a new zone entry for just that IP range.  
Services: `disable` a service will remove it from that zone as specified in the variables. Don't just delete the config to remove it!  
Ports: To remove a port, delete it from the config (under port_protocol).  

Since VMs could have different interfaces (eth0/eth1 etc.), interfaces aren't managed.

Be careful, this will remove and add firewalld rules on the OS. Use with caution.  

firewalld manual: <https://firewalld.org/documentation/>

**Note**: Docker and firewalld do not get along. This role has a check enabled to fail this role if the docker service is running or enabled.  
For more information about firewalld and Docker:  
<https://success.docker.com/article/why-am-i-having-network-problems-after-firewalld-is-restarted>  
<https://www.tripwire.com/state-of-security/devops/psa-beware-exposing-ports-docker/>  
<https://docs.docker.com/network/iptables/>  

## Distros tested

* CentOS: 7.6, 7.7, 8.1
* Fedora 30, 31
* Arch Linux (firewalld version 0.8.1) - But who would run a bunch of Arch servers..

## Dependencies

* firewalld

Tested with version 0.6.3 (Latest available in CentOS 7)  
Tested with version 0.6.6 (Latest available in Fedora 30)  
Tested with version 0.7.0 (Latest available in CentOS 8)  
Tested with version 0.7.3 (Latest available in Fedora 31)  
Tested with version 0.8.1 (Latest available in Arch Linux)  

## Default Settings

* Enable debug

```yaml
debug_enabled_default: false
```

* Proxy (Needed when installing required packages if behind a proxy)

```yaml
proxy_env: []
```

* Role disabled by default. Change to true in group_vars or playbook etc

```yaml
firewalld_managed: false
```

* Show configuration from variables

```yaml
firewalld_show_config: false
```

* Start firewalld service

```yaml
firewalld_start: false
```

* Install firewalld package

```yaml
firewalld_managed_pkg: true
```

## User Settings

* Remove unwanted default services from public zone

```yaml
firewalld_public_remove_default:
  - mdns
  - samba-client
```

* Remove unwanted default services from internal zone

```yaml
firewalld_internal_remove_default:
  - mdns
  - samba-client
```

* Zone Config

Full list of firewalld predefined zones: <https://firewalld.org/documentation/zone/predefined-zones.html>

```yaml
firewalld_zone_source:
  - zone: name of predefined zone (ex. internal|public) (Required)
    state: enabled|disabled (Required)
    source:
      - "IP/Subnet" (Required)
```

* Service and Port Config

```yaml
firewalld_custom_service:
  - name: service name (Required)
    zone: public|internal etc (See zone list above) (Required)
    state: enabled (Required)
    description: Description of service (Optional)
    port_protocol: (Required, unless a built-in service except when changing a default port for built-in service)
      - port/protocol (Required, unless a built-in service)
```

See example for more details.  
Get list of build-in services:  

```bash
firewall-offline-cmd --get-services
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
firewalld_zone_source:
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

firewalld_custom_service:
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

## Example Playbook firewalld.yml

```yaml
---
- hosts: '{{ inventory }}'
  become: yes
  vars:
    # Use this role #RTFM
    # firewalld_managed: true
  roles:
  - firewalld
```

## Usage

By default no tasks will run unless you set `firewalld_managed=true`. This is by design to prevent accidents by people who don't RTFM.

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 firewalld_managed=true" -i hosts-dev
```

Skip installing packages (if known already there - speeds up task)

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 firewalld_managed=true" -i hosts --skip-tags=firewalld_pkg_install
```

Show more verbose output (debug info)

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 firewalld_managed=true debug_enabled_default=true" -i hosts-dev
```

Start firewalld service at end of role

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 firewalld_managed=true firewalld_start=true" -i hosts-dev
```

Only show configuration (from variables)

```bash
ansible-playbook firewalld.yml --extra-vars "inventory=centos7 firewalld_managed=true firewalld_show_config=true" -i hosts --tags "firewalld_show_config"
```

## firewalld Command Reference

More commands can be found in firewalld documentation: <https://firewalld.org/documentation/man-pages/firewall-offline-cmd>

Add IPs to a Zone:

```bash
firewall-offline-cmd --zone=public --list-all

firewall-offline-cmd --zone=internal --add-source=192.168.22.64/26
firewall-offline-cmd --zone=internal --add-source=192.168.23.64/26
```

Add custom service to public zone:  

```bash
firewall-offline-cmd --new-service=app123-public
firewall-offline-cmd --service=app123-public --set-short=app123-public
firewall-offline-cmd --service=app123-public --set-description='app123 fw rules for public zone'
firewall-offline-cmd --service=app123-public --add-port=5000/tcp
firewall-offline-cmd --zone=public --add-service=app123-public
```

Add custom service to internal zone:  

```bash
firewall-offline-cmd --new-service=app123-internal
firewall-offline-cmd --service=app123-internal --set-short=app123-internal
firewall-offline-cmd --service=app123-internal --set-description='app123 fw rules for internal zone'
firewall-offline-cmd --service=app123-internal --add-port=8080/tcp
firewall-offline-cmd --service=app123-internal --add-port=9000/tcp
firewall-offline-cmd --zone=internal --add-service=app123-internal

firewall-offline-cmd --zone=internal --list-all
```

Get list of build-in services:  

```bash
firewall-offline-cmd --get-services
```

Misc useful commands:

```bash
firewall-cmd --state
firewall-cmd --get-active-zones

firewall-offline-cmd --zone=public --list-all
firewall-offline-cmd --zone=internal --list-all

firewall-offline-cmd --zone=public --list-services
firewall-offline-cmd --zone=internal --list-services

firewall-offline-cmd --info-service=app123-public
firewall-offline-cmd --info-service=app123-internal

firewall-cmd --reload
```

## iptables Command Reference

List iptables that are active:

```bash
iptables -nL
```

## nftables Command Reference

List nftables that are active:

```bash
nft list tables
nft list ruleset
nft list table inet firewalld
nft list chain inet firewalld filter_IN_public_allow
```

## TODO

* [x] Get working with firewalld started or stopped
* [x] Confirm reload everywhere needed: "notify: Reload firewalld"
* [ ] Add more tags to tasks
* [x] Test this actually works if firewalld service is running and doesn't block traffic to running app
* [x] Build travis tests for many scenarios
* [ ] Improve/shorten changing/removing a port
* [ ] --diff doesn't show much since using many commands. Show what's going to happen and add pause when is debug enabled?
