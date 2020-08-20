# Azure + Ansible Demo

## Pre-requites

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
