# Cleaning Up

In this lab you will delete the compute resources created during this tutorial.

# Removing the resource group
```
az group delete -n k8s-the-hard-way
```
This will ask for confirmation, and then delete all resources in the resource group.

Also make sure to 'reset' your az cli defaults:

```
az configure --defaults group='' location=''
```