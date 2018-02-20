# Prerequisites

## Azure

This tutorial leverages [Azure](https://azure.microsoft.com/en-us/) to streamline provisioning of the compute infrastructure required to bootstrap a Kubernetes cluster from the ground up. [Sign up](https://azure.microsoft.com/en-us/free/) for $200 in free credits.

## Azure CLI

### Install the Azure CLI

Follow the Azure [documentation](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) to install and configure the `az` command line utility.

### Set a Default Compute Region and Zone

This tutorial assumes a default data center region and resource group have been configured.

If you are using the `az` command-line tool for the first time `login` is the easiest way to do this:

```
az login
```

Let's create the resource group for our deployment, and configure the CLI to use this resource group (and its location) as default. It is recommended to create a new resource group for this walkthrough, as we'll remove it in the [Final step](14-cleanup.md).
```
az group create -l westeurope -n k8s-the-hard-way
az configure --defaults group=k8s-the-hard-way location=westeurope
```

Next: [Installing the Client Tools](02-client-tools.md)
