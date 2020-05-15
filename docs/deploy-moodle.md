# Deploy Moodle on LAMP Stack to Azure

After following the steps in this this document you will have a new Moodle instance, ready to host your application at scale.
The filesystem behind it is mirrored for high availability and
optionally backed up through Azure.


## Prerequisites

Host VM (where Moodle to be installed) and Ansible VM should be in the same resource group and same location.
This procedure tested on Ubuntu 16.04-LTS

## Enabling Password Authentication  
- Login into host VM and navigate home/azureadmin(user)/ 
- Run the following commands to enable password authentication.

```sh
sudo sed -i "s~PasswordAuthentication no~PasswordAuthentication yes~" /etc/ssh/sshd_config
sudo sed -i "s~#UseLogin no~UseLogin yes~" /etc/ssh/sshd_config
sudo sed -i "s~#   StrictHostKeyChecking ask~   StrictHostKeyChecking no~" /etc/ssh/ssh_config
sudo systemctl restart sshd
sudo passwd azureadmin(user)
```

Above command will prompt user to set a new password.

Note: 
- This new password should be given as an input to the Ansible VM template deployment.
- To copy SSH keys to Host VM it should be password enabled.

#### Create Database

- User should create a database with name “moodledb” in MySQL with the help of MySQL Workbench.
     Azure recomends MySQL 5.6, 5.7 & 8.0 and default version is 5.7.
- To create the database in MYSQL workbench user should add current client IP which can be found at Azure Database for MySQL server and navigate to the “connection security”. 

## Deploy the Ansible VM

-	Install Ansible VM through ARM Template by giving the following inputs.

            o	Host VM Name
            o	SSH Public Key
            o	Host VM Username
            o	Host Load Balancer IP
            o	Host VM IP
            o	Host VM Password
            o	Database Server Name
            o	Database Login Name
            o	Database Login Password
            o	Moodle Domain Name

-   User will be prompted to purchase and deployment starts.

-	The above template will execute a script "moodlemain.sh' in extensions at the time of ARM Ansible Template deployment.
-	moodlemain.sh script will download the Moodle script "moodlescript.sh" file which will be downloaded from the GitHub repository.
-	moodlemain.sh script will create a "run.sh" script at the home/azureadmin(user)/ 


#### Execute Ansible playbook

#### Install Moodle into host VM.

Login to Ansible VM and go to /home/azureadmin(username)/ path.
Run the below command to install Moodle in host VM
                
                bash run.sh


#### Installation and Replication of Moodle

##### Installing Moodle into Host VM.

run.sh file will execute ansible script "moodlescript.sh" which will run the playbook with the user provided details.
    
    Moodlescript.sh has the following functions 

    - Setup Ansible
         - Download & Install Ansible server with latest version 2.9.6v
         - Configure the host webserver in ansible host file (/etc/ansible/hosts)

    - Moodle Install
          - Clone the Moodle repo “ansible_playbook” which contains Roles, Playbooks, main.yml file.
          - Below are three roles in the ansible playbook for installing Moodle
               - Sshkeyconfig
               - Moodle
               - Replication

- In sshkeyconfig the task will generate openssl keypair and next it will copy the key to the host VM by password authentication.

- In Moodle role, Moodle will be downloaded in .zip format and installed in host virtual machine at targeted location (/var/www/html/).

- In replication role it will download and execute the script “moodlereplication.sh” . It will replicate Moodle directory to the Virtual Machine Scale Set (VMSS). 

- Replication Script(moodlereplication.sh) will have following functions
    - Change location: It will move the Moodle folder to /azlamp/html/ location
    - Configure SSL certs: Generate OpenSSL certificates to /azlamp/certs/ folder
    - Linking data location:  This function will link data to all shared  web frontend instances
    - Update Nginx Configuration: This function will update the Nginx configuration file
    - Moodledata: It will create parent and Moodle data directory for Moodle and this will be replicating to the VMSS
    - Replication: This function will replicate the Moodle folder to the VMSS instance

