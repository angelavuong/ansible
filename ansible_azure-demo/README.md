# Azure + Ansible Demo

![Azure Deployment](https://github.com/angelavuong/ansible/blob/main/ansible_azure-demo/images/Azure%20Deployment.png)

In this demo, we will learn how to:
* [Part 1: Install and configure Ansible in Azure](https://github.com/angelavuong/ansible_azure_workshop#part-1-build-ansible-vm-in-azure)
* [Part 2: Use Ansible Playbooks to deploy a CentOS VM in Azure](https://github.com/angelavuong/ansible_azure_workshop#part-2-build-centos-vm-using-ansible-playbooks)
* [Part 3: Build an inventory file](https://github.com/angelavuong/ansible_azure_workshop#part-3-build-an-inventory-file)
* [Part 4: Install Apache on VM](https://github.com/angelavuong/ansible_azure_workshop#part-4-install-apache-on-vm)
* [Part 5: Create second VM in Azure for database](https://github.com/angelavuong/ansible_azure_workshop#part-5-create-second-vm-in-azure-for-database)

## Prerequisites

### Obtain Azure Account

1. If you don't already have one, you can get a [free one](https://azure.microsoft.com/en-us/free/).

2. Install Azure CLI on your local system.

For macOS, you can do this via homebrew:
  ```
  % brew install azure-cli
  ```

  NOTE: Azure CLI has a dependency on the Homebrew ```python3``` package and will install on it.

For RHEL, you need to import the Microsoft repository key first:
```
sudo rpm --import https://packages.microsoft.com/keys/microsoft.asc
```
Then, create local azure-cli repository info:
```
sudo sh -c 'echo -e "[azure-cli]
name=Azure CLI
baseurl=https://packages.microsoft.com/yumrepos/azure-cli
enabled=1
gpgcheck=1
gpgkey=https://packages.microsoft.com/keys/microsoft.asc" > /etc/yum.repos.d/azure-cli.repo'
```
Finally, do a yum package install:
```
sudo yum install azure-cli
```
More details are [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-yum?view=azure-cli-latest).


  You can login to Azure via CLI now:
  ```
  % az login
  ```

  That will bring up a window pop-up to verify your Azure login.

3. Create a [service principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest) using the Azure CLI.

  ```
  % az ad sp create-for-rbac
  ```

  Make note of the following values: appId, displayName, password, and tenant.

## Part 1: Build Ansible VM in Azure

1. Generate SSH hostkeys (on local system where you will use Azure CLI)

    ```
    $ ssh-keygen -m PEM -t rsa -b 2048 -C "azureuser@azure" -f ~/.ssh/ansible_rsa -N ""
    ```

2. Create a resource group:

    ```
    $ az group create --name QuickstartAnsible-rg --location eastus
    ```

3. Create a VM:

    ```
    $ az vm create \
    --resource-group QuickstartAnsible-rg \
    --name QuickstartAnsible-vm \
    --image OpenLogic:CentOS:7.7:latest \
    --admin-username azureuser \
    --ssh-key-values <ssh_public_key_filename>
    ```

    where ```<ssh_public_key_filename>``` points to your SSH key file (e.g. ~/.ssh/ansible_rsa.pub)

4. Verify the creation (and state) of the new virtual machine:

    ```
    $ az vm list -d -o table --query "[?name=='QuickstartAnsible-vm']"
    ```

5. Install Ansible on the virtual machine:

    ```
    $ az vm extension set \
     --resource-group QuickstartAnsible-rg \
     --vm-name QuickstartAnsible-vm \
     --name customScript \
     --publisher Microsoft.Azure.Extensions \
     --version 2.1 \
     --settings '{"fileUris":["https://raw.githubusercontent.com/MicrosoftDocs/mslearn-ansible-control-machine/master/configure-ansible-centos.sh"]}' \
     --protected-settings '{"commandToExecute": "./configure-ansible-centos.sh"}'
     ```

6. Verify you can login to your Ansible VM from your local system using SSH:

    ```
    $ ssh -i ansible_rsa azureuser@<vm_ip_address>
    [azureuser@QuickstartAnsible-vm ~]$
    [azureuser@QuickstartAnsible-vm ~]$
    ```

7. Set up your Ansible credentials for the VM.

    First, let's create a credentials file.

    ```
    [azureuser@QuickstartAnsible-vm .ssh]$ mkdir ~/.azure
    [azureuser@QuickstartAnsible-vm .ssh]$ vi ~/.azure/credentials

    [default]
    subscription_id=<your-subscription_id>
    client_id=<security-principal-appid>
    secret=<security-principal-password>
    tenant=<security-principal-tenant>
    ```

    Insert values for credentials using the values you obtained for the service principal values.
    NOTE: For ```subscription_id```, use the value for ```id``` when you logged into the Azure CLI.

    Next, let's set up the environment variables with the same values:

    ```
    [azureuser@QuickstartAnsible-vm .ssh]$ export AZURE_SUBSCRIPTION_ID=<your-subscription_id>
    [azureuser@QuickstartAnsible-vm .ssh]$ export AZURE_CLIENT_ID=<security-principal-appid>
    [azureuser@QuickstartAnsible-vm .ssh]$ export AZURE_SECRET=<security-principal-password>
    [azureuser@QuickstartAnsible-vm .ssh]$ export AZURE_TENANT=<security-principal-tenant>
    ```

8. Verify Ansible is installed (and check version):

    ```
    [azureuser@QuickstartAnsible-vm ~]$ ansible --version

    ansible 2.9.12
      config file = None
      configured module search path = ['/home/azureuser/.ansible/plugins/modules', '/usr/share/ansible/
    plugins/modules']
      ansible python module location = /usr/local/lib/python3.6/site-packages/ansible
      executable location = /usr/local/bin/ansible
      python version = 3.6.8 (default, Apr  2 2020, 13:34:55) [GCC 4.8.5 20150623 (Red Hat 4.8.5-39)]
    [azureuser@QuickstartAnsible-vm ~]$
    ```
9. Install required Ansible python packages to manage your hosts:

   Create a file called requirements-azure.txt (https://github.com/ansible/ansible/blob/stable-2.7/packaging/requirements/requirements-azure.txt).
   Install the required packages using pip: ```pip install -r requirements-azure.txt``` 

   In addition, please install the following packages:
   ``` 
   $ pip install paramiko -U
   $ pip install netmiko -U
   ``` 

## Part 2: Build CentOS VM using Ansible Playbooks

1. Before we start, generate SSH keys on your Ansible VM:
```
$ ssh-keygen -m PEM -t rsa -b 4096
```

2. Create a file named ```azure_create_complete_vm.yml```:

```
[azureuser@QuickstartAnsible-vm ~]$ mkdir ~/ansible_azure_workshop
[azureuser@QuickstartAnsible-vm ~]$ vi ~/ansible_azure_workshop/azure_create_complete_vm.yml
[azureuser@QuickstartAnsible-vm ~]$
```

**Section 1: Create a resource group**

This will create a resource group called "myResourceGroup" in the ```eastus``` location:
```
- name: Create resource group
  azure_rm_resourcegroup:
    name: myResourceGroup
    location: eastus
```

**Section 2: Create a virtual network**

Adding a virtual network will allow our VM to be accessed:
```
- name: Create virtual network
  azure_rm_virtualnetwork:
    resource_group: myResourceGroup
    name: myVnet
    address_prefixes: "10.0.0.0/16"
```

**Section 3: Creating a subnet**

All VM resources are deployed into a subnet within a virtual network. So we need to add a subnet for our VM:
```
- name: Add subnet
  azure_rm_subnet:
    resource_group: myResourceGroup
    name: mySubnet
    address_prefix: "10.0.1.0/24"
    virtual_network: myVnet
```

**Section 4: Create a public IP address**

We will ask Azure to dynamically assign an available public IP address to our VM instance. This will enable reachability of the VM from outside our network (via the Internet).
```
- name: Create public IP address
  azure_rm_publicipaddress:
    resource_group: myResourceGroup
    allocation_method: Static
    name: myPublicIP
```

**Section 5: Create a network security group**

We will create a security group that allows SSH traffic (via port 22) to the VM:
```
- name: Create Network Security Group that allows SSH
  azure_rm_securitygroup:
    resource_group: myResourceGroup
    name: myNetworkSecurityGroup
    rules:
      - name: SSH
        protocol: Tcp
        destination_port_range: 22
        access: Allow
        priority: 1001
        direction: Inbound
```

**Section 6: Create virtual network interface card**

This will create a  virtual NIC that connects the VM to the virtual network:
```
- name: Create virtual network interface card
  azure_rm_networkinterface:
    resource_group: myResourceGroup
    name: myNIC
    virtual_network: myVnet
    subnet: mySubnet
    public_ip_name: myPublicIP
    security_group: myNetworkSecurityGroup
```

**Section 7: Create a virtual machine**

Lastly, we will create a VM called "myVM":
```
- name: Create VM
  azure_rm_virtualmachine:
    resource_group: myResourceGroup
    name: myVM
    vm_size: Standard_DS1_v2
    admin_username: azureuser
    ssh_password_enabled: false
    ssh_public_keys:
      - path: /home/azureuser/.ssh/authorized_keys
        key_data: <your-key-data>
    network_interfaces: myNIC
    image:
      offer: CentOS
      publisher: OpenLogic
      sku: '7.5'
      version: latest
```

3. Here is the complete playbook. Use ```i``` for insert, paste the contents below to create a VM using Ansible:

```
- name: Create Azure VM
  hosts: localhost
  connection: local
  tasks:
  - name: Create resource group
    azure_rm_resourcegroup:
      name: myResourceGroup
      location: eastus
  - name: Create virtual network
    azure_rm_virtualnetwork:
      resource_group: myResourceGroup
      name: myVnet
      address_prefixes: "10.0.0.0/16"
  - name: Add subnet
    azure_rm_subnet:
      resource_group: myResourceGroup
      name: mySubnet
      address_prefix: "10.0.1.0/24"
      virtual_network: myVnet
  - name: Create public IP address
    azure_rm_publicipaddress:
      resource_group: myResourceGroup
      allocation_method: Static
      name: myPublicIP
    register: output_ip_address
  - name: Dump public IP for VM which will be created
    debug:
      msg: "The public IP is {{ output_ip_address.state.ip_address }}."
  - name: Create Network Security Group that allows SSH
    azure_rm_securitygroup:
      resource_group: myResourceGroup
      name: myNetworkSecurityGroup
      rules:
        - name: SSH
          protocol: Tcp
          destination_port_range: 22
          access: Allow
          priority: 1001
          direction: Inbound
  - name: Create virtual network interface card
    azure_rm_networkinterface:
      resource_group: myResourceGroup
      name: myNIC
      virtual_network: myVnet
      subnet: mySubnet
      public_ip_name: myPublicIP
      security_group: myNetworkSecurityGroup
  - name: Create VM
    azure_rm_virtualmachine:
      resource_group: myResourceGroup
      name: myVM
      vm_size: Standard_DS1_v2
      admin_username: azureuser
      ssh_password_enabled: false
      ssh_public_keys:
        - path: /home/azureuser/.ssh/authorized_keys
          key_data: <your-key-data>
      network_interfaces: myNIC
      image:
        offer: CentOS
        publisher: OpenLogic
        sku: '7.5'
        version: latest
  ```

Be sure to update \<your-key-data\> with your public SSH key (should be in ~/.ssh/id_rsa.pub). You can want to paste the entire SSH public key. For example: "ssh-rsa AAAB6d9c7ede mykey".

Full details and explanation on each playbook section can be found [here](https://docs.microsoft.com/en-us/azure/developer/ansible/vm-configure?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fansible%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fbreadcrumb%2Ftoc.json#complete-sample-ansible-playbook).

4. To run the playbook, run the following in terminal:
```
$ ansible-playbook ~/ansible_azure_workshop/azure_create_complete_vm.yml
```

5. Verify no errors/failures occurred during the run. You can also verify the VM was created through the Azure management UI

6. SSH from your Ansible VM to your new VM (called myVM) via SSH:

  ```
  [azureuser@QuickstartAnsible-vm .ssh]$ ssh azureuser@<ip-address>
  [azureuser@myVM ~]$
  [azureuser@myVM ~]$
  ```

## Part 3: Build an inventory file

1. Let's create an inventory file on our Ansible host that includes the new VM's public IP address

  ```
  [azureuser@QuickstartAnsible-vm ansible_azure_workshop]$ sudo mkdir /etc/ansible
  [azureuser@QuickstartAnsible-vm ansible_azure_workshop]$ sudo touch /etc/ansible/hosts
  [azureuser@QuickstartAnsible-vm ansible_azure_workshop]$ vi /etc/ansible/hosts
  ```

  And add the name of your node with its allocated public IP address:
  ```
  myVM ansible_host=<public_ip_address>
  ```

  Save and close (```:wq!```).

2. Let's run a quick ping test to the host using the Ansible CLI:

  ```
  [azureuser@QuickstartAnsible-vm ansible_azure_workshop]$ ansible all -m ping
  myVM | SUCCESS => {

      "ansible_facts": {
          "discovered_interpreter_python": "/usr/bin/python"
      },
      "changed": false,
      "ping": "pong"
  }
  ```

Great! We know our inventory file correctly lists our VM and Ansible is able to reach it!

## Part 4: Install Apache on VM

Let's build a playbook which installs the latest Apache on our VM and creates a web-page.

1. Create a playbook called apache.yml with the following content:

  ```
  ---
  - name: Apache server installed
    hosts: myVM
    become: true
    tasks:

    - name: latest Apache version installed
      yum:
        name: httpd
        state: latest

    - name: Apache enabled and running
      service:
        name: httpd
        enabled: true
        state: started

    - name: copy index.html
      copy:
        src: web.html
        dest: /var/www/html/index.html
  ```

2. Create a web-page HTML file called ```web.html``` for Apache:

  ```
  <body>
  <h1>Apache is running fine</h1>
  </body>
  ```

3. Then run the playbook:

  ```
  $ ansible-playbook apache.yml
  ```

4. Connect to the VM via SSH:

  ```
  $ ssh azureuser@<ip_address>
  ```

5. Verify Apache (HTTPD) was installed on the host:
```
azureuser@myVM ~]$ rpm -qi httpd
Name        : httpd
Version     : 2.4.6
Release     : 93.el7.centos
Architecture: x86_64
Install Date: Mon 24 Aug 2020 01:51:31 PM UTC
Group       : System Environment/Daemons
Size        : 9821040
License     : ASL 2.0
Signature   : RSA/SHA256, Fri 03 Apr 2020 08:53:10 PM UTC, Key ID 24c6a8a7f4a80eb5
Source RPM  : httpd-2.4.6-93.el7.centos.src.rpm
Build Date  : Thu 02 Apr 2020 01:15:44 PM UTC
Build Host  : x86-01.bsys.centos.org
Relocations : (not relocatable)
Packager    : CentOS BuildSystem <http://bugs.centos.org>
Vendor      : CentOS
URL         : http://httpd.apache.org/
Summary     : Apache HTTP Server
Description :
The Apache HTTP Server is a powerful, efficient, and extensible
web server.
```

## Part 5: Create second VM in Azure for database

Let's create another VM so we can use to install MySQL.

1. Create the VM in Azure using Ansible:

Build a new playbook called ```azure_create_mysql_vm.yml``` that will create our second VM:
```
- name: Create Azure mySQL VM
  hosts: localhost
  connection: local
  tasks:
  - name: Create public IP address
    azure_rm_publicipaddress:
      resource_group: myResourceGroup
      allocation_method: Static
      name: myPublicIP2
    register: output_ip_address
  - name: Dump public IP for VM which will be created
    debug:
      msg: "The public IP is {{ output_ip_address.state.ip_address }}."
  - name: Create virtual network interface card
    azure_rm_networkinterface:
      resource_group: myResourceGroup
      name: myNIC2
      virtual_network: myVnet
      subnet: mySubnet
      public_ip_name: myPublicIP2
      security_group: myNetworkSecurityGroup
    - name: Create VM
    azure_rm_virtualmachine:
      resource_group: myResourceGroup
      name: myVM2
      vm_size: Standard_DS1_v2
      admin_username: azureuser
      ssh_password_enabled: false
      ssh_public_keys:
        - path: /home/azureuser/.ssh/authorized_keys
          key_data: <your_public_key>
      network_interfaces: myNIC2
      image:
        offer: CentOS
        publisher: OpenLogic
        sku: '7.5'
        version: latest
```

NOTE: Be sure to replace ```<your_public_key>``` before moving on.

Run the playbook:
```
$ ansible-playbook azure_create_mysql_vm.yml
```

Verify you can login to the newly created VM:
```
$ ssh azureuser@<public_ip_address>
[azureuser@myVM2 ~]$
```

2. Add myVM2 to your Ansible host file:
```
$ sudo vi /etc/ansible/hosts
myVM ansible_host=<public_ip_address>
myVM2 ansible_host=<public_ip_address>
```

3. Create playbook called ```mysql.yml``` that will install MySQL on myVM2:

```
---
- name: MySQL DB server installed
  hosts: myVM2

  become: true
  tasks:

  - name: Obtain mysql 7.5 community release repository
    yum:
      name: http://repo.mysql.com/mysql-community-release-el7-5.noarch.rpm
      state: present
  - name: Yum package update
    yum:
      name: '*'
      state: present
  - name: Install mysql-server
    yum:
      name: mysql-server
      state: latest
  - name: MySQL enabled and running
    service:
      name: mysqld
      enabled: true
      state: started
```

4. Run the ```mysql.yml``` playbook to install mysqld on myVM2:

```
$ ansible-playbook mysql.yml
```

5. Login to myVM2 and verify the database is installed:

```
$ ssh azureuser@<myVM2_public_ip_address>
[azureuser@myVM2 ~]$ mysql -u root -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 2
Server version: 5.6.49 MySQL Community Server (GPL)

Copyright (c) 2000, 2020, Oracle and/or its affiliates. All rights reserved.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql>
```

NOTE: When prompted for the password, pass <ENTER> if no password is set.

## Resources:
1. [Microsoft Docs - Build Ansible VM in Azure](https://docs.microsoft.com/en-us/azure/developer/ansible/install-on-linux-vm)
2. [Microsoft Docs - How to build VM using Ansible Playbooks](https://docs.microsoft.com/en-us/azure/developer/ansible/vm-configure?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fansible%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fbreadcrumb%2Ftoc.json#complete-sample-ansible-playbook)
3. [Ansible - How to build your inventory](https://docs.ansible.com/ansible/latest/user_guide/intro_inventory.html#intro-inventory)
4. [Ansible - Creating Apache Playbook](https://github.com/ansible/workshops/tree/devel/exercises/ansible_rhel/1.3-playbook)
5. [Ansible - yum package manager](https://docs.ansible.com/ansible/latest/modules/yum_module.html)
6. [MySQL for Centos 7.5](https://www.linode.com/docs/databases/mysql/how-to-install-mysql-on-centos-7/)
