# Installing Ansible Tower

PRE-REQUISITES:
Please refer to the [Ansible Tower installation guide](https://docs.ansible.com/ansible-tower/latest/html/installandreference/requirements_refguide.html) to know the OS and resource requirements.

Minimum:
2 vCPU
4GB RAM
20GB of dedicated hard disk space
64-bit support required

1. Obtain the Ansible Tower installer:

  ```
  $ wget https://releases.ansible.com/ansible-tower/setup/ansible-tower-setup-3.7.1-1.tar.gz
  ```

2. Un-tar the Ansible Tower installer:
  ```
  $ tar xvf ansible-tower-setup-3.7.1-1
  ```

3. Configure your inventory file's passwords

  ```
  [azureuser@QuickstartAnsible-vm ~]$ cd ansible-tower-setup-3.7.1-1/
  [azureuser@QuickstartAnsible-vm ansible-tower-setup-3.7.1-1]$ vi inventory

  [all:vars]
  admin_password=''
  pg_password=''
  ```

4. Run the setup script to begin the Ansible Tower installation. You must run this as root user.
  ```
  [azureuser@QuickstartAnsible-vm ansible-tower-setup-3.7.1-1]$ sudo ./setup.sh
  ```
