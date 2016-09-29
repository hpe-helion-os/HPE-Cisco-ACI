# Cisco ACI Integration with HOS 3.0

[Cisco Application Centric Infrastructure (Cisco ACI)](http://www.cisco.com/c/en/us/solutions/data-center-virtualization/application-centric-infrastructure/index.html) is a software-defined networking (SDN) architecture.  Helion OpenStack 3.0 (HOS 3.0) supports the routing of Neutron traffic through a Cisco ACI fabric by integrating with a [Cisco Application Policy Infrastructure Controller (APIC)](http://www.cisco.com/c/en/us/products/cloud-systems-management/application-policy-infrastructure-controller-apic/index.html).

----------

## Concepts

Integrating with Cisco ACI depends on many different components, so it is important to get at least a basic overview of what they do.

### Helion Lifecycle Manager

The Helion Lifecycle Manager (HLM) is used to orchestrate the deployment.  The automatic deployment playbook uses the hosts in the cloud found in `~/scratch/ansible/next/hos/ansible/hosts/verb_hosts`.

### Neutron Server

Neutron controller nodes handle the interaction between Neutron and APIC.

#### APIC API & APIC ML2 Driver

The APIC API is a Python package that the APIC ML2 driver uses to talk to APIC.

The APIC ML2 driver is a driver for the Neutron servers that lets Neutron interface with APIC.

#### APIC Service Agent

The APIC service agent updates host links in APIC as discovered from the APIC host agent.  This informs APIC about what compute nodes are connected and automatically configurable.

### Nova Compute

Nova compute nodes, which use Open vSwitch, directly interface with APIC under the guidance of the Neutron servers.  A VLAN tagged interface on the physical port connected to APIC is created for communications over OpFlex.

#### OpFlex

OpFlex is the protocol that handles the network traffic through the infrastructure ("infra") VLAN.

#### Agent OVS

`agent-ovs` is what runs on the Nova compute nodes as an OpFlex endpoint.

#### APIC Host Agent

The APIC host agent runs on Nova compute nodes to detect the connection to APIC and lets the APIC host agent running on the Neutron controller nodes know.

#### APIC API & APIC ML2 Driver

Unlike on the Neutron controller nodes, the APIC API and APIC ML2 driver packages are used to support the Neutron OpFlex agent on Nova compute nodes because some features (like the Neutron metadata agent) are offloaded from the Neutron controller nodes and onto the Nova compute nodes.

#### Neutron OpFlex Agent

This agent, identified as `neutron-opflex-agent.service` on the Nova compute nodes, monitors OpFlex and starts multiple other processes, described below:

 - [Supervisor](http://supervisord.org/) ― Functioning as Cisco's spinoff process monitoring system, Supervisor branches off more processes:
   - Neutron metadata agent (`neutron-metadata-agent`) ― Provides the metadata service for virtual machines, proxied through the APIC ML2 driver's metadata namespace proxy
   - OpFlex EP watcher (`opflex-ep-watcher`) ― Monitors OpFlex endpoints
   - OpFlex state watcher (`opflex-state-watcher`) ― Configures metadata routing
     - OpFlex namespace proxy (`opflex-ns-proxy`) ― Performs the proxying for optimized metadata

----------

## Installation

This integration must be run after fully deploying Helion OpenStack without Cisco ACI.  If you reconfigure Neutron after running the integration, you must run the integration again because reconfiguring Neutron will override most of the steps in this integration.

### Caveats

 - While testing an integration, Cisco discovered that APIC version 1.1(4e) has a bug that prevents virtual machines from communicating across the fabric.  This bug is not present in APIC version 2.0(1m).
 - The automatic deployment playbook supports OpFlex VXLAN or VLAN tunneling on only the first physical port discovered to be connected to APIC.  The information about which port goes to APIC comes from `lldpctl`.
 - Compute nodes are discovered by `neutron-cisco-apic-host-agent`, which is installed by the automatic deployment playbook.  Consult with your Cisco ACI specialist for advanced configurations.
 - External networks must be configured separately in APIC before attempting to create them in Neutron.  Consult with your Cisco ACI specialist for assistance.  The `[apic_external_network:<EXT_NET_NAME>]` Neutron configuration section must be added to the bottom of `/opt/stack/service/neutron/etc/ml2_conf.ini` and `neutron-server` needs to be restarted on every Neutron controller node.

### Prerequisites

#### Helion OpenStack

Helion OpenStack 3.0 must be deployed fully first before performing the integration of Cisco ACI.  The instructions for deploying HOS 3.0 can be found [here](http://docs.hpcloud.com/#3.x/helion/installation/installing_kvm.html).

On the HLM, Apache needs to be configured to listen on port 79 HTTP with `/opt/hlm_packager` as the document root.  If you did not modify the Apache configuration, this should already be configured by the HOS deployment.

Every Neutron Open vSwitch server (Neutron controllers and Nova compute nodes) needs the following apt sources configured:

 - `deb [arch=amd64] http://HLM_IP:79/hlm/hlinux cattleprod main contrib non-free`
 - `deb [arch=i386] http://HLM_IP:79/hlm/cmc32 cattleprod main`

… where `HLM_IP` is the IP address of the HLM.  If you did not edit the files in `/etc/apt/sources.list.d`, this should already be configured by the HOS deployment.

#### Network

In addition to the [Helion OpenStack network configuration](http://docs.hpcloud.com/#3.x/helion/installation/preinstall_checklist.html), a physical port on each Nova compute node must be plugged into the Cisco ACI fabric.  Running `sudo lldpcli show neighbors` should reveal a router/leaf from the Cisco ACI fabric.  (Note that `lldpd` needs to be installed from `apt` first in order to run `lldpcli`.)

All of the Neutron controller nodes must be connected so that APIC is reachable from them.  If your APIC IP address is `10.105.1.10`, you should be able to fetch the APIC login interface from the Neutron controller nodes using this command: `curl -k https://10.105.1.10`

Because of how the Cisco third-party software is supplied, some software need to be provided offline (not connected to the Internet) while other software are fetched online (over the Internet) from official Noiro Networks (Cisco) sources.  Internet access on the Helion Lifecycle Manager and the Neutron controller nodes is required to run the automatic deployment playbook.  The playbook can make use of an HTTP/HTTPS proxy.

To summarize:

 - Ensure that each Nova compute node has a dedicated port connected to a Cisco ACI leaf.
 - Ensure that each Neutron controller node can reach APIC.
 - Ensure that the HLM and all Neutron controller nodes have Internet connectivity.

### Integration

 1. On the HLM, write `ciscoaci-deploy.yml` to a location of your choice.  This guide assumes that you have written the code into `~/ciscoaci-deploy.yml`.
 2. Copy the file `ciscoaci-deploy-resources.tar.gz` to the same folder where you put `ciscoaci-deploy.yml`.  In this guide, the path would be `~/ciscoaci-deploy-resources.tar.gz`.
 3. Run the Cisco ACI integration playbook:

    ```
    ansible-playbook -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts ~/ciscoaci-deploy.yml [OPTIONS]
    ```

    Replace `[OPTIONS]` with whichever of these switches that are applicable to your integration:

    | Switch | Requirement | Default | Description
    | --- | --- | --- | ---
    | `-e apic_ip='<string>'` | Recommended | 10.105.1.10 | `<string>` is the IP address of APIC as seen by the Neutron controller nodes.
    | `-e apic_username='<string>'` | Recommended | admin | `<string>` is the username used to authenticate with APIC.
    | `-e apic_password='<string>'` | Recommended | cisco123 | `<string>` is the password used to authenticate with APIC.  The password must not contain single quotes ("`'`").
    | `-e apic_system_id='<string>'` | Suggested | helion-openstack | `<string>` is a string up to 16 characters long identifying your cloud to APIC.
    | `-e proxy='<proxy_address:port>'`   | If necessary | *(blank)* | If your environment requires a proxy to reach the Internet, specify the proxy information in `<proxy_address:port>`.  This applies to `git` on the HLM and to `pip` on the Neutron controller nodes and the Nova compute nodes.
    | `-e opflex_encapsulation_mode='<string>'` | If necessary | vxlan | `<string>` is either `vxlan` or `vlan`.  This sets the OpFlex port encapsulation mode to VXLAN or VLAN, respectively.  If any other string is specified, deployment will fail at the OpFlex port encapsulation configuration.
    | `-e ml2_vlan_physnet='<string>'` | If necessary | physnet1 | `<string>` is the name of the physical network for the `ml2_type_vlan.network_vlan_ranges` configuration option.  Only change this if you know what you are doing.  This option only applies to the VLAN OpFlex encapsulation mode.
    | `-e ml2_vlan_lower=<int>` | If necessary | 1000 | `<int>` is the lower bound of the VLAN for the `ml2_type_vlan.network_vlan_ranges` configuration option.  This option only applies to the VLAN OpFlex encapsulation mode.
    | `-e ml2_vlan_upper=<int>` | If necessary | 2000 | `<int>` is the upper bound of the VLAN for the `ml2_type_vlan.network_vlan_ranges` configuration option.  This option only applies to the VLAN OpFlex encapsulation mode.
    | `-e opflex_peer_ip='<string>'` | If necessary | 10.0.0.30 | `<string>` is the IP address of the Cisco ACI OpFlex peer.
    | `-e opflex_peer_port=<int>` | If necessary | 8009 | `<int>` is the port of the Cisco ACI OpFlex peer.
    | `-e vxlan_remote_ip='<string>'` | If necessary | 10.0.0.32 | `<string>` is the IP address of the VXLAN tunnel endpoint in the Cisco ACI fabric.  This option only applies to the VXLAN OpFlex encapsulation mode.
    | `-e vxlan_remote_port=<int>` | If necessary | 8472 | `<int>` is the port of the VXLAN tunnel endpoint in the Cisco ACI fabric.  This option only applies to the VXLAN OpFlex encapsulation mode.<br />**Warning:** If you change this value, you must first run `sudo ovs-vsctl del-port br-int br-int_vxlan0` manually on every Nova compute node.
    | `-e version_agent='<string>'` | Optional | stable/liberty | `<string>` is a full 40-character SHA-1 hash, the literal string "HEAD", a branch name, or a tag name from the Noiro Networks Git repository <https://github.com/noironetworks/python-opflex-agent>.  Specify this if you wish to use a specific version of the OpFlex agent.
    | `-e version_ml2='<string>'` | Optional | liberty | `<string>` is a full 40-character SHA-1 hash, the literal string "HEAD", a branch name, or a tag name from the Noiro Networks Git repository <https://github.com/noironetworks/apic-ml2-driver>.  Specify this if you wish to use a specific version of the APIC ML2 driver for Neutron.
    | `-e version_api='<string>'` | Optional | HEAD | `<string>` is a full 40-character SHA-1 hash, the literal string "HEAD", a branch name, or a tag name from the Noiro Networks Git repository <https://github.com/noironetworks/apicapi>.  Specify this if you wish to use a specific version of the Python APIC API.
    | `-e keep_scratch=<bool>` | Optional | False | By default, this playbook deletes its temporary files located in `/tmp/scratch` after running.  Set `<bool>` to `True` if you want to preserve the temporary files.
    | `--tags setup` | Optional | *(N/A)* | Specify this to have the playbook create the offline apt repository, fetch the online resources, and do nothing else.
    | `--tags setup-offline` | Optional | *(N/A)* | Specify this to have the playbook create the offline apt repository and do nothing else.
    | `--tags setup-online` | Optional | *(N/A)* | Specify this to have the playbook fetch the online resources and do nothing else.  Note that these online resources may have additional online dependencies which are only downloaded at install time.
    | `--tags install` | Optional | *(N/A)* | Specify this to have the playbook install the third-party software and do nothing else.
    | `--tags install-offline` | Optional | *(N/A)* | Specify this to have the playbook install the third-party software from `~/ciscoaci-deploy-resources.tar.gz` and do nothing else.
    | `--tags install-offline` | Optional | *(N/A)* | Specify this to have the playbook install the third-party software from `/tmp/scratch` and do nothing else.
    | `--tags configure` | Optional | *(N/A)* | Specify this to have the playbook perform the configuration of the cloud components for the integration and do nothing else.
    | `--tags configure-3rdparty` | Optional | *(N/A)* | Specify this to have the playbook perform the configuration of the installed third-party software and do nothing else.
    | `--tags configure-os` | Optional | *(N/A)* | Specify this to have the playbook modify Helion OpenStack configuration files for the integration and do nothing else.
    | `--tags services` | Optional | *(N/A)* | Specify this to have the playbook disable conflicting services, stop conflicting services, enable installed third-party services, restart installed third-party services, restart `neutron-server`, and do nothing else.
    | `--tags services-removeconflicts` | Optional | *(N/A)* | Specify this to have the playbook disable conflicting services, stop conflicting services, and do nothing else.
    | `--tags services-restart` | Optional | *(N/A)* | Specify this to have the playbook enable installed third-party services, restart installed third-party services, restart `neutron-server`, and do nothing else.

----------

## Verification

### Practical Scenarios

#### Two Instances on a Private Network

These instructions create two instances on their own private network and check to make sure that they can reach each other by ICMP.

 1. If you do not already have a CirrOS image in Glance called "cirros-0.3.4-x86_64-disk", run these commands to set up a CirrOS image:

    ```
    source ~/service.osrc
    cd
    # Export a proxy if necessary (e.g. `export http_proxy="http://proxy.atlanta.hp.com:8080"`)
    wget 'http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img'
    glance image-create --progress --name cirros-0.3.4-x86_64-disk --disk-format qcow2 --container-format bare --visibility public --file ~/cirros-0.3.4-x86_64-disk.img
    ```
    
    Sample output:

    ```
    stack@helion-cp1-c0-m1-mgmt:/tmp$ source ~/service.osrc
    stack@helion-cp1-c0-m1-mgmt:~$ cd
    stack@helion-cp1-c0-m1-mgmt:~$ wget 'http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img'
    --2016-08-23 15:16:56--  http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
    Resolving download.cirros-cloud.net (download.cirros-cloud.net)... 64.90.42.85
    Connecting to download.cirros-cloud.net (download.cirros-cloud.net)|64.90.42.85|:80... connected.
    HTTP request sent, awaiting response... 200 OK
    Length: 13287936 (13M) [text/plain]
    Saving to: ‘cirros-0.3.4-x86_64-disk.img’
    
    cirros-0.3.4-x86_64-disk.img                                           100%[=============================>]  12.67M  17.3MB/s   in 0.7s   
    
    2016-08-23 15:16:56 (17.3 MB/s) - ‘cirros-0.3.4-x86_64-disk.img’ saved [13287936/13287936]
    
    stack@helion-cp1-c0-m1-mgmt:~$ glance image-create --progress --name cirros-0.3.4-x86_64-disk --disk-format qcow2 --container-format bare --visibility public --file ~/cirros-0.3.4-x86_64-disk.img
    [=============================>] 100%
    +------------------+--------------------------------------+
    | Property         | Value                                |
    +------------------+--------------------------------------+
    | checksum         | ee1eca47dc88f4879d8a229cc70a07c6     |
    | container_format | bare                                 |
    | created_at       | 2016-08-23T15:17:09Z                 |
    | disk_format      | qcow2                                |
    | id               | f33a2066-d623-4db4-bea5-004b772fa00a |
    | min_disk         | 0                                    |
    | min_ram          | 0                                    |
    | name             | cirros-0.3.4-x86_64-disk             |
    | owner            | 688c22afba0c4a9f8b9b7be50cd62376     |
    | protected        | False                                |
    | size             | 13287936                             |
    | status           | active                               |
    | tags             | []                                   |
    | updated_at       | 2016-08-23T15:17:17Z                 |
    | virtual_size     | None                                 |
    | visibility       | public                               |
    +------------------+--------------------------------------+
    ```

 2. Create a private network that the instances will later use:

    ```
    neutron net-create Net100
    ```

    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron net-create Net100
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | d58b49af-c4ad-4288-92b7-9e792a393562 |
    | mtu                       | 0                                    |
    | name                      | Net100                               |
    | provider:network_type     | opflex                               |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  |                                      |
    | router:external           | False                                |
    | shared                    | False                                |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | 688c22afba0c4a9f8b9b7be50cd62376     |
    +---------------------------+--------------------------------------+
    ```

 3. Create a subnet for that network:

    ```
    neutron subnet-create --name SubNet100 Net100 192.168.100.0/24 --gateway 192.168.100.1
    ```

    Sample output:

    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron subnet-create --name SubNet100 Net100 192.168.100.0/24 --gateway 192.168.100.1
    Created a new subnet:
    +-------------------+------------------------------------------------------+
    | Field             | Value                                                |
    +-------------------+------------------------------------------------------+
    | allocation_pools  | {"start": "192.168.100.2", "end": "192.168.100.254"} |
    | cidr              | 192.168.100.0/24                                     |
    | dns_nameservers   |                                                      |
    | enable_dhcp       | True                                                 |
    | gateway_ip        | 192.168.100.1                                        |
    | host_routes       |                                                      |
    | id                | e09ec631-4eab-4818-9560-d18c2407df43                 |
    | ip_version        | 4                                                    |
    | ipv6_address_mode |                                                      |
    | ipv6_ra_mode      |                                                      |
    | name              | SubNet100                                            |
    | network_id        | d58b49af-c4ad-4288-92b7-9e792a393562                 |
    | subnetpool_id     |                                                      |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376                     |
    +-------------------+------------------------------------------------------+
    ```

 4. Set up a security group with rules that allow all TCP, UDP, and ICMP communications:

    ```
    neutron security-group-create allow_all
    neutron security-group-rule-create allow_all --direction ingress --protocol icmp
    neutron security-group-rule-create allow_all --direction egress --protocol icmp
    neutron security-group-rule-create allow_all --direction ingress --protocol tcp
    neutron security-group-rule-create allow_all --direction egress --protocol tcp
    neutron security-group-rule-create allow_all --direction ingress --protocol udp
    neutron security-group-rule-create allow_all --direction egress --protocol udp
    ```

    Sample output:

    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-create allow_all
    Created a new security_group:
    +----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | Field                | Value                                                                                                                                                                                                                                                           |
    +----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    | description          |                                                                                                                                                                                                                                                                 |
    | id                   | 4121c835-85f2-4e01-8cfe-846638834c66                                                                                                                                                                                                                            |
    | name                 | allow_all                                                                                                                                                                                                                                                       |
    | security_group_rules | {"remote_group_id": null, "direction": "egress", "remote_ip_prefix": null, "protocol": null, "tenant_id": "688c22afba0c4a9f8b9b7be50cd62376", "port_range_max": null, "security_group_id": "4121c835-85f2-4e01-8cfe-846638834c66", "port_range_min": null,      |
    |                      | "ethertype": "IPv4", "id": "e7b6a1e9-1ea8-4b60-9cd5-68777ab26c8b"}                                                                                                                                                                                              |
    |                      | {"remote_group_id": null, "direction": "egress", "remote_ip_prefix": null, "protocol": null, "tenant_id": "688c22afba0c4a9f8b9b7be50cd62376", "port_range_max": null, "security_group_id": "4121c835-85f2-4e01-8cfe-846638834c66", "port_range_min": null,      |
    |                      | "ethertype": "IPv6", "id": "348af9dc-deb5-4d69-b0f3-0d99a8dae5b5"}                                                                                                                                                                                              |
    | tenant_id            | 688c22afba0c4a9f8b9b7be50cd62376                                                                                                                                                                                                                                |
    +----------------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction ingress --protocol icmp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | ingress                              |
    | ethertype         | IPv4                                 |
    | id                | 0ae259c6-dda6-4276-9e58-6a6da88c1a52 |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | icmp                                 |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction egress --protocol icmp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | egress                               |
    | ethertype         | IPv4                                 |
    | id                | bbfcfde7-0e13-4a53-877f-170589f3a0ba |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | icmp                                 |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction ingress --protocol tcp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | ingress                              |
    | ethertype         | IPv4                                 |
    | id                | 3c3452bb-9b97-4eff-b3d9-ffb5e646ac71 |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | tcp                                  |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction egress --protocol tcp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | egress                               |
    | ethertype         | IPv4                                 |
    | id                | 4e5f1336-385e-47dc-b105-cc77e0035b54 |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | tcp                                  |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction ingress --protocol udp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | ingress                              |
    | ethertype         | IPv4                                 |
    | id                | 448f5465-7d7a-4abe-a456-4c41a140b3ae |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | udp                                  |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron security-group-rule-create allow_all --direction egress --protocol udp
    Created a new security_group_rule:
    +-------------------+--------------------------------------+
    | Field             | Value                                |
    +-------------------+--------------------------------------+
    | direction         | egress                               |
    | ethertype         | IPv4                                 |
    | id                | 939721cb-dbfd-48b7-af59-724392077d2b |
    | port_range_max    |                                      |
    | port_range_min    |                                      |
    | protocol          | udp                                  |
    | remote_group_id   |                                      |
    | remote_ip_prefix  |                                      |
    | security_group_id | 4121c835-85f2-4e01-8cfe-846638834c66 |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-------------------+--------------------------------------+
    ```

 5. Create two virtual machine instances:

    ```
    nova boot --flavor 1 --image "cirros-0.3.4-x86_64-disk" --nic net-id="$(neutron net-list | awk '/Net100/ {print $2}')",v4-fixed-ip="192.168.100.11" --security-groups allow_all cisco-test-vm1
    nova boot --flavor 1 --image "cirros-0.3.4-x86_64-disk" --nic net-id="$(neutron net-list | awk '/Net100/ {print $2}')",v4-fixed-ip="192.168.100.12" --security-groups allow_all cisco-test-vm2
    ```

    Sample output:

    ```
    stack@helion-cp1-c0-m1-mgmt:~$ nova boot --flavor 1 --image "cirros-0.3.4-x86_64-disk" --nic net-id="$(neutron net-list | awk '/Net100/ {print $2}')",v4-fixed-ip="192.168.100.11" --security-groups allow_all cisco-test-vm1
    +--------------------------------------+-----------------------------------------------------------------+
    | Property                             | Value                                                           |
    +--------------------------------------+-----------------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                          |
    | OS-EXT-AZ:availability_zone          |                                                                 |
    | OS-EXT-SRV-ATTR:host                 | -                                                               |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                               |
    | OS-EXT-SRV-ATTR:instance_name        | instance-000001df                                               |
    | OS-EXT-STS:power_state               | 0                                                               |
    | OS-EXT-STS:task_state                | scheduling                                                      |
    | OS-EXT-STS:vm_state                  | building                                                        |
    | OS-SRV-USG:launched_at               | -                                                               |
    | OS-SRV-USG:terminated_at             | -                                                               |
    | accessIPv4                           |                                                                 |
    | accessIPv6                           |                                                                 |
    | adminPass                            | ue44DTfYviSx                                                    |
    | config_drive                         |                                                                 |
    | created                              | 2016-08-23T15:26:48Z                                            |
    | flavor                               | m1.tiny (1)                                                     |
    | hostId                               |                                                                 |
    | id                                   | 4197f2a5-fa69-4a25-9a13-b059f731e2bf                            |
    | image                                | cirros-0.3.4-x86_64-disk (f33a2066-d623-4db4-bea5-004b772fa00a) |
    | key_name                             | -                                                               |
    | metadata                             | {}                                                              |
    | name                                 | cisco-test-vm1                                                  |
    | os-extended-volumes:volumes_attached | []                                                              |
    | progress                             | 0                                                               |
    | security_groups                      | allow_all                                                       |
    | status                               | BUILD                                                           |
    | tenant_id                            | 688c22afba0c4a9f8b9b7be50cd62376                                |
    | updated                              | 2016-08-23T15:26:48Z                                            |
    | user_id                              | 4700ca14593c4224a5ad9a73c16ac374                                |
    +--------------------------------------+-----------------------------------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ nova boot --flavor 1 --image "cirros-0.3.4-x86_64-disk" --nic net-id="$(neutron net-list | awk '/Net100/ {print $2}')",v4-fixed-ip="192.168.100.12" --security-groups allow_all cisco-test-vm2
    +--------------------------------------+-----------------------------------------------------------------+
    | Property                             | Value                                                           |
    +--------------------------------------+-----------------------------------------------------------------+
    | OS-DCF:diskConfig                    | MANUAL                                                          |
    | OS-EXT-AZ:availability_zone          |                                                                 |
    | OS-EXT-SRV-ATTR:host                 | -                                                               |
    | OS-EXT-SRV-ATTR:hypervisor_hostname  | -                                                               |
    | OS-EXT-SRV-ATTR:instance_name        | instance-000001e2                                               |
    | OS-EXT-STS:power_state               | 0                                                               |
    | OS-EXT-STS:task_state                | scheduling                                                      |
    | OS-EXT-STS:vm_state                  | building                                                        |
    | OS-SRV-USG:launched_at               | -                                                               |
    | OS-SRV-USG:terminated_at             | -                                                               |
    | accessIPv4                           |                                                                 |
    | accessIPv6                           |                                                                 |
    | adminPass                            | skQbnD2eDb9r                                                    |
    | config_drive                         |                                                                 |
    | created                              | 2016-08-23T15:27:00Z                                            |
    | flavor                               | m1.tiny (1)                                                     |
    | hostId                               |                                                                 |
    | id                                   | 0b20479b-2be2-4557-84f0-f5698d4ff7ec                            |
    | image                                | cirros-0.3.4-x86_64-disk (f33a2066-d623-4db4-bea5-004b772fa00a) |
    | key_name                             | -                                                               |
    | metadata                             | {}                                                              |
    | name                                 | cisco-test-vm2                                                  |
    | os-extended-volumes:volumes_attached | []                                                              |
    | progress                             | 0                                                               |
    | security_groups                      | allow_all                                                       |
    | status                               | BUILD                                                           |
    | tenant_id                            | 688c22afba0c4a9f8b9b7be50cd62376                                |
    | updated                              | 2016-08-23T15:27:00Z                                            |
    | user_id                              | 4700ca14593c4224a5ad9a73c16ac374                                |
    +--------------------------------------+-----------------------------------------------------------------+
    ```

 6. Get the VNC console of one of the instances:

    ```
    nova get-vnc-console cisco-test-vm1 novnc
    ```

    Sample output:

    ```
    stack@helion-cp1-c0-m1-mgmt:~$ nova get-vnc-console cisco-test-vm1 novnc
    +-------+----------------------------------------------------------------------------------+
    | Type  | Url                                                                              |
    +-------+----------------------------------------------------------------------------------+
    | novnc | https://10.105.2.8:6080/vnc_auto.html?token=be773ae5-06c7-453b-b26c-9d51323e7952 |
    +-------+----------------------------------------------------------------------------------+
    ```

 7. Connect to the noVNC console by visiting in your web browser the outputted URL from the previous step.
    
    Sample output:

    ![CirrOS login screen](docs/images/cirros-login.png)

 8. Log in to CirrOS using the default username `cirros` and the default password `cubswin:)`.
 9. `ping` the other instance that you started using this command:

    ```
    ping -c4 192.168.100.12
    ```

    Sample output:

    ![CirrOS ping other instance](docs/images/cirros-ping-other-instance.png)

 10. Visually verify in APIC that the tenant has been created along with the network `Net100` and the subnet `192.168.100.0/24`.
    
    Sample output:
    
    ![APIC verify subnet](docs/images/apic-verify-subnet.png)

#### External Network

This is a continuation of the previous scenario.  These instructions connect the two instances from before to the external network by means of floating IP addresses.

 1. Configure an external routed network in APIC.  For assistance, please contact your Cisco ACI specialist.  This guide assumes that your layer 3 domain is called `ext-net`, your external endpoint group (external EPG) is called `ext-net-EPG`, your host pool CIDR is `10.105.11.1/24`, and your external IP address range is `10.105.12.0/24`.
 2. On every Neutron controller node, at the bottom of the file `/opt/stack/service/neutron/etc/ml2_conf.ini`, add the following section and options:
    
    ```
    [apic_external_network:ext-net]
    external_epg = ext-net-EPG
    host_pool_cidr = 10.105.11.1/24
    preexisting = True
    ```
    
 3. Restart `neutron-server` on every Neutron controller node:
    
    ```
    ansible --sudo -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR -m service -a "name=neutron-server state=restarted"
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ ansible --sudo -i ~/scratch/ansible/next/hos/ansible/hosts/verb_hosts NEU-SVR -m service -a "name=neutron-server state=restarted"
    helion-cp1-c1-m2-mgmt | success >> {
        "changed": true, 
        "name": "neutron-server", 
        "state": "started"
    }
    
    helion-cp1-c1-m3-mgmt | success >> {
        "changed": true, 
        "name": "neutron-server", 
        "state": "started"
    }
    
    helion-cp1-c1-m1-mgmt | success >> {
        "changed": true, 
        "name": "neutron-server", 
        "state": "started"
    }
    ```

 4. Create the external network in Neutron:

    ```
    neutron net-create ext-net --router:external --shared
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron net-create ext-net --router:external --shared
    Created a new network:
    +---------------------------+--------------------------------------+
    | Field                     | Value                                |
    +---------------------------+--------------------------------------+
    | admin_state_up            | True                                 |
    | id                        | 61dfcd28-7f36-4db2-8dae-ff86b3e64c04 |
    | mtu                       | 0                                    |
    | name                      | ext-net                              |
    | provider:network_type     | opflex                               |
    | provider:physical_network | physnet1                             |
    | provider:segmentation_id  |                                      |
    | router:external           | True                                 |
    | shared                    | True                                 |
    | status                    | ACTIVE                               |
    | subnets                   |                                      |
    | tenant_id                 | 688c22afba0c4a9f8b9b7be50cd62376     |
    +---------------------------+--------------------------------------+
    ```
       
       **Note:** If you get a message reading `create_network_postcommit failed.` when running this command, then you did not configure the same external network name in APIC that you just tried to create in Neutron.
    
 5. Create the public subnet `10.105.12.0/24`:

    ```
    neutron subnet-create ext-net 10.105.12.0/24 --name ext-subnet --gateway 10.105.12.1
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron subnet-create ext-net 10.105.12.0/24 --name ext-subnet --gateway 10.105.12.1
    Created a new subnet:
    +-------------------+--------------------------------------------------+
    | Field             | Value                                            |
    +-------------------+--------------------------------------------------+
    | allocation_pools  | {"start": "10.105.12.2", "end": "10.105.12.254"} |
    | cidr              | 10.105.12.0/24                                   |
    | dns_nameservers   |                                                  |
    | enable_dhcp       | True                                             |
    | gateway_ip        | 10.105.12.1                                      |
    | host_routes       |                                                  |
    | id                | 9ba2d15a-100e-4dda-9dd8-4e8383905901             |
    | ip_version        | 4                                                |
    | ipv6_address_mode |                                                  |
    | ipv6_ra_mode      |                                                  |
    | name              | ext-subnet                                       |
    | network_id        | 61dfcd28-7f36-4db2-8dae-ff86b3e64c04             |
    | subnetpool_id     |                                                  |
    | tenant_id         | 688c22afba0c4a9f8b9b7be50cd62376                 |
    +-------------------+--------------------------------------------------+
    ```

 6. Confirm that a network with a name beginning with `host-snat-network-for-internal-use-` has been created:

    ```
    neutron net-list
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron net-list
    +--------------------------------------+-------------------------------------------------------------------------+-------------------------------------------------------+
    | id                                   | name                                                                    | subnets                                               |
    +--------------------------------------+-------------------------------------------------------------------------+-------------------------------------------------------+
    | 699a35b6-ff60-4ce3-ab1e-1f9911a19b11 | OCTAVIA-MGMT-NET                                                        | b266ffd3-75fb-457c-a68d-3934b0a9170e 10.105.6.0/24    |
    | a1a38e64-0132-4e85-b3a9-38f20df7e050 | host-snat-network-for-internal-use-61dfcd28-7f36-4db2-8dae-ff86b3e64c04 | 52b03b85-9508-4a97-9e4a-ec163db2bbb8 10.105.11.0/24   |
    | 61dfcd28-7f36-4db2-8dae-ff86b3e64c04 | ext-net                                                                 | 9ba2d15a-100e-4dda-9dd8-4e8383905901 10.105.12.0/24   |
    | d58b49af-c4ad-4288-92b7-9e792a393562 | Net100                                                                  | e09ec631-4eab-4818-9560-d18c2407df43 192.168.100.0/24 |
    +--------------------------------------+-------------------------------------------------------------------------+-------------------------------------------------------+
    ```

 7. Create a router to connect the private network to the external network:

    ```
    neutron router-create Net100-router
    neutron router-interface-add Net100-router SubNet100
    neutron router-gateway-set Net100-router ext-net
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron router-create Net100-router
    Created a new router:
    +-----------------------+--------------------------------------+
    | Field                 | Value                                |
    +-----------------------+--------------------------------------+
    | admin_state_up        | True                                 |
    | external_gateway_info |                                      |
    | id                    | f3667713-408c-46e2-98a8-1795fae73458 |
    | name                  | Net100-router                        |
    | routes                |                                      |
    | status                | ACTIVE                               |
    | tenant_id             | 688c22afba0c4a9f8b9b7be50cd62376     |
    +-----------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron router-interface-add Net100-router SubNet100
    Added interface dcb034e1-fd00-4ceb-a7de-a25a16d11d07 to router Net100-router.
    stack@helion-cp1-c0-m1-mgmt:~$ neutron router-gateway-set Net100-router ext-net
    Set gateway for router Net100-router
    ```

 8. Create floating IP addresses for the instances:

    ```
    neutron floatingip-create ext-net --floating-ip-address 10.105.12.11
    neutron floatingip-create ext-net --floating-ip-address 10.105.12.12
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ neutron floatingip-create ext-net --floating-ip-address 10.105.12.11
    Created a new floatingip:
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    |                                      |
    | floating_ip_address | 10.105.12.11                         |
    | floating_network_id | 61dfcd28-7f36-4db2-8dae-ff86b3e64c04 |
    | id                  | b188f515-d5c5-4b73-822c-96229ba0cf5d |
    | port_id             |                                      |
    | router_id           |                                      |
    | status              | DOWN                                 |
    | tenant_id           | 688c22afba0c4a9f8b9b7be50cd62376     |
    +---------------------+--------------------------------------+
    stack@helion-cp1-c0-m1-mgmt:~$ neutron floatingip-create ext-net --floating-ip-address 10.105.12.12
    Created a new floatingip:
    +---------------------+--------------------------------------+
    | Field               | Value                                |
    +---------------------+--------------------------------------+
    | fixed_ip_address    |                                      |
    | floating_ip_address | 10.105.12.12                         |
    | floating_network_id | 61dfcd28-7f36-4db2-8dae-ff86b3e64c04 |
    | id                  | 498b3515-e148-4c8e-8ed9-6e070cf6fcff |
    | port_id             |                                      |
    | router_id           |                                      |
    | status              | DOWN                                 |
    | tenant_id           | 688c22afba0c4a9f8b9b7be50cd62376     |
    +---------------------+--------------------------------------+
    ```

 9. Connect floating IP addresses to each instance:

    ```
    nova floating-ip-associate cisco-test-vm1 10.105.12.11
    nova floating-ip-associate cisco-test-vm2 10.105.12.12
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ nova floating-ip-associate cisco-test-vm1 10.105.12.11
    stack@helion-cp1-c0-m1-mgmt:~$ nova floating-ip-associate cisco-test-vm2 10.105.12.12
    ```

 10. Confirm that you can connect to the instances using their public IP addresses:

    ```
    ssh cirros@10.105.12.11
    ssh cirros@10.105.12.12
    ```
    
    Sample output:
    
    ```
    stack@helion-cp1-c0-m1-mgmt:~$ ssh cirros@10.105.12.11
    Warning: Permanently added '10.105.12.11' (RSA) to the list of known hosts.
    cirros@10.105.12.11's password: 
    $ ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue 
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
        link/ether fa:16:3e:21:0f:b1 brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.11/24 brd 192.168.100.255 scope global eth0
        inet6 fe80::f816:3eff:fe21:fb1/64 scope link 
           valid_lft forever preferred_lft forever
    $ Connection to 10.105.12.11 closed.
    stack@helion-cp1-c0-m1-mgmt:~$ ssh cirros@10.105.12.12
    Warning: Permanently added '10.105.12.12' (RSA) to the list of known hosts.
    cirros@10.105.12.12's password: 
    $ ip addr
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 16436 qdisc noqueue 
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast qlen 1000
        link/ether fa:16:3e:86:2e:c6 brd ff:ff:ff:ff:ff:ff
        inet 192.168.100.12/24 brd 192.168.100.255 scope global eth0
        inet6 fe80::f816:3eff:fe86:2ec6/64 scope link 
           valid_lft forever preferred_lft forever
    $ Connection to 10.105.12.12 closed.
    ```

