# Azure + Ansible Demo

## Pre-requites

### Install Azure CLI

Since I will have Ansible installed on my local computer (macOS), I will also need to install the Azure CLI.

```
% brew install azure-cli
```

NOTE: Azure CLI has a dependency on the Homebrew ```python3``` package and will install on it.

You can login to Azure via CLI now:
```
az login
```

That will bring up a window pop-up

### Install Ansible

On macOS, you can install Ansible using brew:
```
% brew install ansible
```

Verify Ansible is installed:
```
% ansible --version
```

### Obtain Azure Account

1. If you don't already have one, you can get a [free one](https://azure.microsoft.com/en-us/free/).

2. Create a [service principal](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest). Make note of the following values: appId, displayName, password, and tenant.
