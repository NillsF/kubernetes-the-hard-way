# Provisioning Pod Network Routes

Pods scheduled to a node receive an IP address from the node's Pod CIDR range. At this point pods can not communicate with other pods running on different nodes due to missing network [routes](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-udr-overview#user-defined).

In this lab you will create a route for each worker node that maps the node's Pod CIDR range to the node's internal IP address.

> There are [other ways](https://kubernetes.io/docs/concepts/cluster-administration/networking/#how-to-achieve-this) to implement the Kubernetes networking model.

## The Routing Table

In this section you will gather the information required to create routes in the `kubernetes-the-hard-way` VNET.


## Routes

Create network routes for each worker instance:

```
az network route-table create --name k8s-workers
for i in 0 1 2; do
  az network route-table route create --name kubernetes-route-10-200-${i}-0-24 \
    --route-table-name k8s-workers \
    --next-hop-ip-address 10.240.0.2${i} \
    --address-prefix 10.200.${i}.0/24 \
    --next-hop-type VirtualAppliance
done		
```

Now we need to link this route table to our subnet:

```
az network vnet subnet update --name k8s-cluster --vnet-name k8s-the-hard-way --route-table k8s-workers
```


List the routes in the `k8s-workers` route table:

```
az network route-table route list --route-table-name k8s-workers
```

> output

```
AddressPrefix    Name                            NextHopIpAddress    NextHopType       ProvisioningState    ResourceGroup
---------------  ------------------------------  ------------------  ----------------  -------------------  ----------------
10.200.0.0/24    kubernetes-route-10-200-0-0-24  10.240.0.20         VirtualAppliance  Succeeded            k8s-the-hard-way
10.200.1.0/24    kubernetes-route-10-200-1-0-24  10.240.0.21         VirtualAppliance  Succeeded            k8s-the-hard-way
10.200.2.0/24    kubernetes-route-10-200-2-0-24  10.240.0.22         VirtualAppliance  Succeeded            k8s-the-hard-way
```

Next: [Deploying the DNS Cluster Add-on](12-dns-addon.md)
