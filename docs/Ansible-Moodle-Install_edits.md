# Installing Moodle on a LAMP Cluster in Microsoft Azure

This document explains how to install Moodle on a LAMP cluster in Azure. 
The following diagram shows how the required system components participate in the installation process.
![](https://github.com/krishnaitalent/LAMP/blob/lamp_docmentation/images/Moodle_Flow_Diagram.png)

## Requirements
To install Moodle, you must prepare your system environment as follows:
- Deploy a LAMP stack to create the Controller VM that will host the Moodle installation. To support the current installation, the LAMP stack must be running:
	*	Ubuntu 16.04 LTS
	*	Nginx web server 1.10.3
	*	MySQL PaaS 5.6, 5.7 or 8.0 database server
	*	PHP 7.2, 7.3, or 7.4 
  
- Make sure the Ansible VM is located in the same Azure resource group and region that hosts the Controller VM.

## Overview of the Installation Process

The installation process consists of the following high-level procedures:

- Enable password authentication on the Controller VM
- Deploy the Ansible VM to install Moodle on the Controller VM

### Enable Password Authentication on the Controller VM

Enabling password authentication allows you to obtain the Controller VM credentials and copy the SSH keys that you will need to deploy the Ansible VM later in the installation process. 

1. Log in to the Controller VM and navigate to the /home/*azureadmin(user)*/ directory, where *azureadmin(user)* represents the username of the Azure admin account.
2. Run the following commands to enable password authentication.
```
sudo sed -i "s~PasswordAuthentication no~PasswordAuthentication yes~" /etc/ssh/sshd_config
sudo sed -i "s~#UseLogin no~UseLogin yes~" /etc/ssh/sshd_config
sudo sed -i "s~#   StrictHostKeyChecking ask~   StrictHostKeyChecking no~" /etc/ssh/ssh_config
sudo systemctl restart sshd
sudo passwd azureadmin
```
3. When prompted, configure a new password for the Controller VM.
4. Gather the following information from the Controller VM: 
	*	Controller VM username
	*	Controller VM IP address
	*	Controller VM password (the new password that you configured in the previous step)


### Deploy the Ansible VM to Install Moodle

The Ansible VM will be used to execute the Moodle installation script. To deploy the Ansible VM, you use the ARM Template and provide input parameters describing the Controller VM and the new database that will be created during the installation of Moodle.
	
1. Use the [ARM Template](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fummadisudhakar%2FLAMP%2Fansible_playbook_mat32%2Fansibledeploy-moodle.json) to start the Ansible VM deployment. Make sure you are deploying Ansible VM into the same Azure resource group and region that hosts the Controller VM.
2. Enter the name of the Ansible VM.
3. Enter the Controller VM information that you gathered earlier:
	*	Conroller VM name
	*	SSH public key
	*	Controller VM username
	*	Controller VM IP address
	*	Controller VM password
4. Enter the information for the new Moodle database:
	*	Server name of the MySQL server in the LAMP cluster
	*	Login username for the MySQL server
	*	Login password for the MySQL server
	*	Name of the new Moodle database
5. Enter additional configuration information:
	*	Domain name of the Moodle instance
	*	Domain name of the load balancer, which you can obtain from the Azure Portal

The following processes occur during the deployment:

- The ARM Template executes the moodle_main.sh script from the extension’s.
- The moodle_main.sh script downloads the Ansible script (moodle_script.sh) from GitHub.
- The downloaded moodle_script.sh script is placed in the /home/*azureadmin(user)* directory on the Ansible VM, where *azureadmin(user)* represents the username of the Azure admin account.

Once the deployment is complete, the Ansible VM executes the moodle_script.sh script, which installs Moodle using an Ansible playbook.

The run.sh file runs the Ansible script moodle_script.sh consisting of the Ansible playbook with the following roles:

- **Role: SSH Key Configuration**  
Generates the SSH key pair and copies the key pair to the Controller VM using password authentication.

- **Role: Moodle**  
Downloads Moodle in .zip format and installs in on the Controller VM.

- **Role: Replication**  
Downloads the script moodle_replication.sh. This script executes the following functions.
	*	__Change Location:__ Moves the Moodle directory to the /azlamp/html/ location.
	*	__Configure SSL Certs:__ Generates the required OpenSSL certificates and places them in the /azlamp/certs/ directory.
	*	__Linking Data Location:__ Links the Moodle data to all shared web frontend instances.
	*	__Create Nginx Configuration:__ Updates the Nginx configuration file.
	*	__Moodledata:__ Creates the moodledata directory in the azlamp location.
	*	__Replication:__ Replicates the Moodle directory to the Virtual Machine Scale Set (VMSS) instance for high availability.

When all the roles are executed, the installation is complete.

## Accessing Moodle

- To open Moodle, connect to the load balancer for the Controller VM using the load balancer's IP address. You can obtain the load balancer's IP address from the Azure Portal.

- To log in to Moodle, use the username and password provided in the moodle.txt file at /home/*azureadmin(user)*, where *azureadmin(user)* represents the username of the Azure admin account.
