## closet-switch-design-l3

### Summary:

This is an Ansible demo which configures two Cumulus VX switches with BGP and NetQ using J2 / Jinja2 templates

### Network Diagram:

![Network Diagram](https://github.com/chronot1995/closet-switch-design-l3/blob/master/documentation/closet-switch-design-l3.png)

### Initializing the demo environment:

First, make sure that the following is currently running on your machine:

1. Vagrant > version 2.1.5

    https://www.vagrantup.com/

2. Virtualbox > version 5.2.16

    https://www.virtualbox.org

3. Copy the Git repo to your local machine:

    ```git clone https://github.com/chronot1995/closet-switch-design-l3```

4. Change directories to the following

    ```closet-switch-design-l3```

6. Run the following:

    ```./start-vagrant-poc.sh```

### Running the Ansible Playbook

1. SSH into the oob-mgmt-server:

    ```cd vx-simulation```   
    ```vagrant ssh oob-mgmt-server```

2. Copy the Git repo unto the oob-mgmt-server:

    ```git clone https://github.com/chronot1995/closet-switch-design-l3```

3. Change directories to the following

    ```closet-switch-design-l3/automation```

4. Run the following:

    ```./provision.sh```

This will bring run the automation script and configure the two switches with BGP.

### Troubleshooting

Helpful NCLU troubleshooting commands:

- net show route
- net show bgp summary
- net show interface | grep -i UP
- net show lldp

Helpful Linux troubleshooting commands:

- ip route
- ip link show
- ip address <interface>

The BGP Summary command will show if each switch had formed a neighbor relationship:

```
cumulus@switch01:mgmt-vrf:~$ net show bgp summary

show bgp ipv4 unicast summary
=============================
BGP router identifier 10.1.1.1, local AS number 65111 vrf-id 0
BGP table version 3
RIB entries 5, using 760 bytes of memory
Peers 2, using 39 KiB of memory

Neighbor        V         AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd
switch02(swp1)  4      65222       8       8        0    0    0 00:00:09            2
switch02(swp2)  4      65222       8       9        0    0    0 00:00:09            2

Total number of neighbors 2

```

One should see that the corresponding loopback route is installed with two next hops / ECMP:

```
cumulus@switch01:mgmt-vrf:~$ net show route

show ip route
=============
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, P - PIM, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel,
       > - selected route, * - FIB route

K>* 0.0.0.0/0 [0/0] via 10.0.2.2, vagrant, 00:00:29
C>* 10.0.2.0/24 is directly connected, vagrant, 00:00:29
C>* 10.1.1.1/32 is directly connected, lo, 00:00:29
B>* 10.2.2.2/32 [20/0] via fe80::4638:39ff:fe00:2, swp1, 00:00:25
  *                    via fe80::4638:39ff:fe00:4, swp2, 00:00:25
```

NetQ allows us to see all of the BGP connections:

```
cumulus@switch01:mgmt-vrf:~$ netq show bgp
Matching bgp records:
Hostname         Neighbor                         VRF              ASN        Peer ASN   PfxRx        Last Changed
---------------- -------------------------------- ---------------- ---------- ---------- ------------ ----------------
switch01         swp1(switch02)                   default          65111      65222      2/-/-        51.110938s
switch01         swp2(switch02)                   default          65111      65222      2/-/-        51.110954s
switch02         swp1(switch01)                   default          65222      65111      2/-/-        51.110959s
switch02         swp2(switch01)                   default          65222      65111      2/-/-        51.110964s
```

NetQ also allows us to see how the demo is cabled up:

```
cumulus@switch01:mgmt-vrf:~$ netq show lldp
Matching lldp records:
Hostname    Interface    Peer Hostname    Peer Interface    Last Changed
----------  -----------  ---------------  ----------------  --------------
switch01    eth0         oob-mgmt-switch  swp3              47.961817s
switch01    swp1         switch02         swp1              47.961724s
switch01    swp2         switch02         swp2              47.961605s
switch02    eth0         oob-mgmt-switch  swp4              49.970120s
switch02    swp1         switch01         swp1              49.969953s
switch02    swp2         switch01         swp2              49.969830s
```


### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

    ```vagrant destroy -f```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the vx-simulation directory:

    ```python2 topology_converter.py closet-switch-design-l3.dot -c```

    After the above command is executed, the following configuration changes are necessary:

4. Within ```vx-simulation/helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh```

The following stanza:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.3.1.0

Will be replaced with the following:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.6.2

The following stanza will replace the install_ansible function:

```
install_ansible(){
echo " ### Installing Ansible... ###"
apt-get install -qy ansible sshpass libssh-dev python-dev libssl-dev libffi-dev
sudo pip install pip --upgrade
sudo pip install setuptools --upgrade
sudo pip install ansible==$ansible_version --upgrade
}```

Add the following ```echo``` right before the end of the file.

    echo " ### Adding .bash_profile to auto login as cumulus user"
    echo "sudo su - cumulus" >> /home/vagrant/.bash_profile
    echo "exit" >> /home/vagrant/.bash_profile

    echo "############################################"
    echo "      DONE!"
    echo "############################################"

5. Within the Vagrantfile, it's important to make sure the oob-mgmt-server should have the following interface configuration:

        device.vm.provider "virtualbox" do |vbox|
          vbox.customize ['modifyvm', :id, '--nicpromisc2', 'allow-all']
          vbox.customize ["modifyvm", :id, "--nictype1", "virtio"]
          vbox.customize ["modifyvm", :id, "--nictype2", "virtio"]

This will make sure that the virtio driver is used to improve the speed of downloading NetQ.

6. helper_scripts > auto_mgmt_network > dhcpd.hosts:

Enable "option routers 192.168.200.254" and 8.8.8.8 as the DNS server:

        group {

          option domain-name-servers 8.8.8.8;
          option domain-name "simulation";
          option routers 192.168.200.254;
          option www-server 192.168.200.254;
          option default-url = "http://192.168.200.254/onie-installer";

7. helper_scripts > auto_mgmt_network > dhcpd.conf:

This is an ancillary configuration to #6

        # OOB Management subnet
        shared-network LOCAL-NET{
        subnet 192.168.200.0 netmask 255.255.255.0 {
          range 192.168.200.10 192.168.200.50;
          option domain-name-servers 8.8.8.8;
          option domain-name "simulation";
          option routers 192.168.200.254;
          default-lease-time 172800;  #2 days
          max-lease-time 345600;      #4 days
          option www-server 192.168.200.254;
          option default-url = "http://192.168.200.254/onie-installer";
          option cumulus-provision-url "http://192.168.200.254/ztp_oob.sh";
          option ntp-servers 192.168.200.254;
        }
