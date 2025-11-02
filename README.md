# Proxmox (PVE)
Complete PVE cluster management.

Manage bare-metal PVE cluster turnup, LXC, and KVM containers and their
complete lifecycle.

## Requirements
No additional technical requirements. Previous PVE experience is recommended
for setting up role.

Role version matches PVE release version with an extra specifier for role
revisions. Applying role that is mis-matched to PVE version is **NOT**
supported:

PVE: `7.4` -> r_pufky.pve: `7.4.X`

## Role Variables
Read **all** default files and the **examples** docs. Settings have been throughly documented for
usage.
[defaults/main/main.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/main.yml).

[defaults/main/datacenter.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/datacenter.yml).

[defaults/main/firewall.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/firewall.yml).

[defaults/main/cluster.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/cluster.yml).

[defaults/main/lxc.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/lxc.yml).

[defaults/main/kvm.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/kvm.yml).

[defaults/main/cloud_init.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/cloud_init.yml).

[defaults/main/ports.yml](https://github.com/r-pufky/ansible_pve/blob/main/defaults/main/ports.yml).

## Dependencies
No external dependencies. A custom python interface has been created and
unittested for interfacing with Proxmox.

# PVE Cluster Example
See [included example vars](https://github.com/r-pufky/ansible_pve/blob/main/docs/example)
configuration for a full working example.

## PVE Install (ISO)
Manually install PVE to the bare-metal machine and prep for provisioning:

1. ISO Install may be done with a weak password (auto-changed) during
   provisioning.
2. Ensure network adapters (physical adapter names) are correct in host_vars.
3. Ensure ROUTER has server added to DNAT override group if using DNAT.
4. If removing and re-adding a cluster node see:
    https://pve.proxmox.com/wiki/Cluster_Manager

See [Hosts](https://github.com/r-pufky/ansible_pve/blob/main/docs/example/hosts) for defining hosts for role use.

## Cold Start (Ansible Deployment Machine)
Cold starting a PVE cluster mean turning up a PVE cluster with no pre-existing
inafrastructure (e.g. DNS) from ansible deployment machine.

1. Set static hostnames on router for cluster nodes / infra
2. Statically define hosts on machine executing the ansible roles:

     /etc/hosts
       192.168.0.5   pm1.example.com    pm1
       192.168.0.10  pm2.example.com    pm2
       192.168.0.15  pm3.example.com    pm3
       192.168.0.250 pihole.example.com pihole # Use your own DNS resolver.
3. If using DNAT, make sure to add machine excuting ansible role as an
   exception to correctly resolve.
4. Update `inventory` ansible file:

    ```
    # Define **ALL** PVE cluster nodes under pve_nodes.
    [pve_nodes]
    pm1.example.com
    pm2.example.com
    pm3.example.com

    # Define your primary PVE cluster node.
    [pve_masters]
    pm1.example.com

    # Convience to filter all LXC hosts. (--tags lxc)
    [pve_lxc]
    xplex.example.com

    # Convience to filter all KVM hosts. (--tags kvm)
    [pve_kvm]
    vtest.example.com

    # Convience to filter all containers (--tags lxc,kvm)
    [pve_containers:children]
    pve_lxc
    pve_kvm
    ```
5. Configure `group_vars` and `host_vars`. See example files for an example
   configuration.
4. [Configure Ansible on PVE nodes](https://github.com/r-pufky/ansible_pve/blob/main/README.md#pve-ansible-configuration).

## PVE Ansible Configuration
Unconfigured PVE installs need to be setup for ansible connections. This is
done outside of the PVE role as it can be very specific for environments. PVE
must be connected through 'root' (`remote_user: 'root'`) to do this initial
setup. Be sure to:

1. Add your ansible account with your preferred methods. Be sure to add to sudo
   users for all commands.
2. Update SSHD configuration with your preferred configuration for connections.
3. Reboot the PVE node if needed.

Future connections should be able to be done through the ansible account with
the ability to execute 'root' commands.

site.yml
```yaml
- name: 'PVE Ansible Configuration'
  hosts: 'all'
  remote_user: 'root'
  tasks:
    - name: 'PVE Ansible | init bare host with PVE role'
      ansible.builtin.include_tasks: 'roles/your/custom/init.yml'
  tags:
    - 'pve_cold_start'
    - 'never'
```

Add ansible configure to all PVE nodes.
```bash
ansible-playbook site.yml --tags pve_cold_start --limit {CLUSTER HOSTS}*.example.com --ask-pass
```

## Provision
Once the PVE nodes can be connected via an ansible account, provision the PVE
cluster. This will prep the bare-metal nodes with basic configuration to be
added to the PVE cluster, including:

* Apply subscription keys OR non-subscription repo
* Ensure packages and system is completely up to date
* Apply root password changes, if required
* Enable hardware virtualization
* Apply network configuration
* Apply DNS resolver configuration
* Create/add node to defined PVE cluster
* Install fail2ban filters/rules specifically for PVE WebUI (service is NOT
  installed)
* Changes are applied serially

site.yml
```yaml
- name: 'provision pve cluster nodes'
  hosts:  'pve_nodes'
  become: true
  serial: 1
  roles:
    # Commented roles are suggestions for additional things to do during
    # provisioning.
    #- 'users' # Your own custom users role.
    #- 'sshd' # Additional SSH customization
    #- 'wireguard' # Let's manage wireguard too.
    #- 'zfs' # ZFS as well.
    #- 'nfs' # NFS.
    #- 'fail2ban' # and install fail2ban role.
    - 'r_pufky.pve/provision'
  tags:
    - 'pve_provision'
```

```bash
ansible-playbook site.yml --tags pve_provision --limit pve_nodes -v
```
This can be re-applied at any time to re-provision the bare-metal nodes for
changes. See https://pve.proxmox.com/wiki/Cluster_Manager for removing and
re-adding cluster nodes.

## Cluster
Once the PVE cluster is created, (re)configure the cluster for use. This is
done via the master cluster node, including:

* Apply subscription keys OR non-subscription repo
* Apply datacenter configuration
* Apply DNS resolver configuration
* Apply root email configuration
* Apply datacenter firewall configuration
* Enable nvidia passthru on specified cluster node
* Install cronjobs on specified cluster nodes

```yaml
- name: 'pve configure cluster'
  hosts:  'pve_masters'
  become: true
  roles:
    # Commented roles are suggestions for additional things to do during
    # cluster configuration.
    #- 'users' # Your own custom users role.
    - 'r_pufky.pve/cluster'
  tags:
    - 'pve_cluster'
```

```bash
ansible-playbook site.yml --tags pve_cluster --limit pve_masters -v
```

## Containers
Once the cluster has been configured, containers can than be added using the
LXC and KVM subroles. These are executed through the pve master node.

Container configurations are autoamtically searched through `host_vars`;
applying either of these roles **without** limiting to specific hosts and
`pve_masters` will (re)configure **all** hosts that define LXC/KVM containers
in `host_vars`.

See [host_vars/xplex.example.com/lxc.yml](https://github.com/r-pufky/ansible_pve/blob/main/docs/example/host_vars/xplex.example.com/vars/lxc.yml).
```yaml
- name: 'pve create lxc containers'
  hosts:  'pve_masters'
  become: true
  roles:
    # Commented roles are suggestions for additional things to do during
    # cluster configuration.
    #- 'users' # Your own custom users role.
    - 'r_pufky.pve/lxc'
  tags:
    - 'lxc'
```

See [host_vars/vtest.example.com/kvm.yml](https://github.com/r-pufky/ansible_pve/blob/main/docs/example/host_vars/vtest.example.com/vars/kvm.yml).
```yaml
- name: 'pve create kvm containers'
  hosts:  'pve_masters'
  become: true
  roles:
    # Commented roles are suggestions for additional things to do during
    # cluster configuration.
    #- 'users' # Your own custom users role.
    - 'r_pufky.pve/kvm'
  tags:
    - 'kvm'
```

```bash
ansible-playbook site.yml --tags lxc,kvm -v
```

Containers maybe deleted before creating them with these roles by enabling
optional destroy flags. This is a single host or a comma separated list of
hosts:
```bash
-e 'pve_destroy_hosts="xtest.example.com,xtest2.example.com"'
```

## Putting It All Together
Once the basic subroles are defined above, containers may be turned up in a single
command:

Turn up a LXC container, initalize it with your ansible role, apply a db role:
```bash
ansible-playbook site.yml --tags lxc,init,db --limit xdb.example.com,pve_masters -v
```

Rebuild a container by destroying it and re-creating it from scratch:
```bash
ansible-playbook site.yml --tags lxc,init,db --limit xdb.example.com,pve_masters -v -e 'pve_destroy_hosts="xdb.example.com"'
```

Build a PVE cluster and deploy base containers.
```bash
ansible-playbook site.yml --tags pve_cold_start --limit {CLUSTER HOSTS}*.example.com --ask-pass
ansible-playbook site.yml --tags pve_provision,pve_cluster --limit pve_nodes -v
ansible-playbook site.yml --tags lxc,kvm --limit pve_masters -v
```

## Issues
Create a bug and provide as much information as possible.
Associate pull requests with a submitted bug.

## License
[AGPL-3.0 License](https://github.com/r-pufky/ansible_pve/blob/main/LICENSE)

## Author Information
https://keybase.io/rpufky
