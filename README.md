## campus-switch-deployment

### Summary:

This is an Ansible demo which configures a Cumulus VX switch with standard campus features using Jinja2 and the NCLU Ansible module

The purpose of this playbook is to show network engineers, new to automation, a demo to configure a campus switch solution.

### Network Diagram:

![Network Diagram](https://github.com/chronot1995/campus-switch-deployment/blob/master/documentation/campus-switch-deployment.png)

### Tasks completed in this demo:

1. Configures all of the front facing ports (1-29, in my example / it would be 1 - 4# in a real world design)
2. Configures VLANs 100, 200, 300, and 500
3. Sets the PVID and VID on all of the front facing ports for VoIP Phone access
4. Sets up DHCP Relaying
5. Sets up NTP
6. Sets up 802.1X and MAB auth
7. TACACS client install
8. TACACS server + TACACS accounting setup
9. Appropriate file permissions for TACACS user
10. Deploy VLAN and SVI configurations using Infrastructure as Code
11. SNMP Configuration
12. SSH Banner

### Initializing the demo environment:

First, make sure that the following is currently running on your machine:

1. Vagrant > version 2.1.5

    https://www.vagrantup.com/

2. Virtualbox > version 5.2.22

    https://www.virtualbox.org

3. Copy the Git repo to your local machine:

    ```git clone https://github.com/chronot1995/campus-switch-deployment```

4. Change directories to the following

    ```campus-switch-deployment```

6. Run the following:

    ```./start-vagrant-poc.sh```

### Running the Ansible Playbook

1. SSH into the oob-mgmt-server:

    ```cd vx-simulation```   
    ```vagrant ssh oob-mgmt-server```

2. Copy the Git repo unto the oob-mgmt-server:

    ```git clone https://github.com/chronot1995/campus-switch-deployment```

3. Change directories to the following

    ```campus-switch-deployment/automation```

4. Run the following:

    ```./provision.sh```


### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

    ```vagrant destroy -f```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the vx-simulation directory:

    ```python2 topology_converter.py campus-switch-deployment.dot -c```

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

5. helper_scripts > auto_mgmt_network > dhcpd.hosts:

Enable "option routers 192.168.200.254" and 8.8.8.8 as the DNS server:

        group {

          option domain-name-servers 8.8.8.8;
          option domain-name "simulation";
          option routers 192.168.200.254;
          option www-server 192.168.200.254;
          option default-url = "http://192.168.200.254/onie-installer";

6. helper_scripts > auto_mgmt_network > dhcpd.conf:

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
