## campus-switch-deployment

### Summary:

  - Cumulus Linux 3.7.11
  - Underlying Topology Converter to 4.7.0
  - Tested against Vagrant 2.1.5 on Mac and Linux. Windows is not supported.
  - Tested against Virtualbox 5.2.32 on Mac 10.14
  - Tested against Libvirt 1.3.1 and Ubuntu 16.04 LTS

### Description

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

### Install and Setup Virtualbox on Mac

Setup Vagrant for the first time on Mojave, MacOS 10.14.6

1. Install Homebrew 2.1.9 (This will also install Xcode Command Line Tools)

    https://brew.sh

2. Install Virtualbox (Tested with 5.2.32)

    https://www.virtualbox.org

I had to go through the install process twice to load the proper security extensions (System Preferences > Security & Privacy > General Tab > "Allow" on bottom)

3. Install Vagrant (Tested with 2.1.5)

    https://www.vagrantup.com

### Install and Setup Linux / libvirt demo environment:

First, make sure that the following is currently running on your machine:

1. This demo was tested on a Ubuntu 16.04 VM w/ 4 processors and 32Gb of Diagram

2. Following the instructions at the following link:

    https://docs.cumulusnetworks.com/cumulus-vx/Development-Environments/Vagrant-and-Libvirt-with-KVM-or-QEMU/

3. Download the latest Vagrant, 2.1.5, from the following location:

    https://www.vagrantup.com/

### Initializing the demo environment:

1. Copy the Git repo to your local machine:

```
    git clone https://github.com/chronot1995/campus-switch-deployment/
```

2. Change directories to the following

```
    campus-switch-deployment
```

3a. Run the following for Virtualbox:

```
    ./start-vagrant-vbox-poc.sh
```

3b. Run the following for Libvirt:

```
    ./start-vagrant-libvirt-poc.sh
```

### Running the Ansible Playbook

1a. SSH into the Virtualbox oob-mgmt-server:

```
    cd vx-vbox-simulation
    vagrant ssh oob-mgmt-server
```

1a. SSH into the Libvirt oob-mgmt-server:

```
    cd vx-libvirt-simulation  
    vagrant ssh oob-mgmt-server
```

2. Copy the Git repo unto the oob-mgmt-server:

```
    git clone https://github.com/chronot1995/campus-switch-deployment
```

3. Change directories to the following

```
    campus-switch-deployment/automation
```

4. Run the following:

```
    ./provision.sh
```

This will run the automation script and configure the environment.

### Errata

1. To shutdown the demo, run the following command from the vx-simulation directory:

```
    vagrant destroy -f
```

2. This topology was configured using the Cumulus Topology Converter found at the following URL:

    https://github.com/CumulusNetworks/topology_converter

3. The following command was used to run the Topology Converter within the appropriate vx-sim directory:

```
     ./topology_converter.py campus-switch-deployment.dot -c --provider=virtualbox
     ./topology_converter.py campus-switch-deployment.dot -c --provider=libvirt
```

After the above command is executed, the following configuration changes are necessary:

4. Within "<vx-sim>/helper_scripts/auto_mgmt_network/OOB_Server_Config_auto_mgmt.sh"

The following stanza:

echo " ### Creating cumulus user ###"
useradd -m cumulus

Will be replaced with the following:

echo " ### Creating cumulus user ###"
useradd -m cumulus -m -s /bin/bash

The following stanza:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.6.3

Will be replaced with the following:

    #Install Automation Tools
    puppet=0
    ansible=1
    ansible_version=2.9.3

Add the following ```echo``` right before the end of the file.

    echo " ### Adding .bash_profile to auto login as cumulus user"
    echo "sudo su - cumulus" >> /home/vagrant/.bash_profile
    echo "exit" >> /home/vagrant/.bash_profile
    echo "### Adding .ssh_config to avoid HostKeyChecking"
    printf "Host * \n\t StrictHostKeyChecking no\n" >> /home/cumulus/.ssh/config

    echo "############################################"
    echo "      DONE!"
    echo "############################################"
