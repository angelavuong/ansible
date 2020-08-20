# Azure + Ansible Demo

In this demo, we will learn how to:
* [Part 1: Install and configure Ansible in Azure](https://github.com/angelavuong/ansible_azure_workshop#part-1-build-ansible-vm-in-azure)
* [Part 2: Use Ansible Playbooks to deploy a CentOS VM in Azure](https://github.com/angelavuong/ansible_azure_workshop#part-2-build-centos-vm-using-ansible-playbooks)

## Prerequisites

### Obtain Azure Account

1. If you don't already have one, you can get a [free one](https://azure.microsoft.com/en-us/free/).

2. Install Azure CLI on your local system. For macOS, you can do this via homebrew:

  ```
  % brew install azure-cli
  ```

  NOTE: Azure CLI has a dependency on the Homebrew ```python3``` package and will install on it.

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

This will createa  virtual NIC that connects the VM to the virtual network:
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

## Part 3: Install Apache on the new VM

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

## Resources:
1. [Microsoft Docs - Build Ansible VM in Azure](https://docs.microsoft.com/en-us/azure/developer/ansible/install-on-linux-vm)
2. [Microsoft Docs - How to build VM using Ansible Playbooks](https://docs.microsoft.com/en-us/azure/developer/ansible/vm-configure?toc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fansible%2Ftoc.json&bc=https%3A%2F%2Fdocs.microsoft.com%2Fen-us%2Fazure%2Fdeveloper%2Fbreadcrumb%2Ftoc.json#complete-sample-ansible-playbook)
