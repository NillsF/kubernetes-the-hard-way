# Provisioning Compute Resources

Kubernetes requires a set of machines to host the Kubernetes control plane and the worker nodes where containers are ultimately run. In this lab you will provision the compute resources required for running a secure and highly available Kubernetes cluster across a single [Azure region](https://azure.microsoft.com/en-us/regions/).

> Ensure a default compute zone and region have been set as described in the [Prerequisites](01-prerequisites.md#set-a-default-compute-region-and-zone) lab.

## Networking

The Kubernetes [networking model](https://kubernetes.io/docs/concepts/cluster-administration/networking/#kubernetes-model) assumes a flat network in which containers and nodes can communicate with each other. In cases where this is not desired [network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/) can limit how groups of containers are allowed to communicate with each other and external network endpoints.

> Setting up network policies is out of scope for this tutorial.

### Virtual Network

In this section a dedicated [Virtual Network](https://azure.microsoft.com/en-us/services/virtual-network/) (VNET) will be setup to host the Kubernetes cluster.

A subnet must be provisioned with an IP address range large enough to assign a private IP address to each node in the Kubernetes cluster.

Create the `kubernetes` VNET and subnet can be done with the following command


```
az network vnet create -n k8s-the-hard-way \
--address-prefix 10.240.0.0/22 \
--subnet-name k8s-cluster --subnet-prefix 10.240.0.0/24
```

> The `10.240.0.0/24` IP address range can host up to 250 compute instances.

### Network security
> An [public load balancer](https://docs.microsoft.com/en-us/azure/load-balancer/load-balancer-get-started-internet-arm-cli) will be used later to expose the Kubernetes API Servers to remote clients.


Create a network security group that allows internal communication across all protocols and allows external SSH, ICMP, and HTTPS.
```
az network nsg create -n k8s-nsg
az network nsg rule create --nsg-name k8s-nsg -n k8s-ssh-https --priority 1000 \
--source-address-prefixes 0.0.0.0/0 --source-port-ranges  0-65535 \
--destination-address-prefixes VirtualNetwork --destination-port-ranges 22 443 6443 --protocol Tcp --access Allow
az network nsg rule create --nsg-name k8s-nsg -n k8s-internal --priority 1010 \
--source-address-prefixes VirtualNetwork --source-port-ranges  \* \
--destination-address-prefixes VirtualNetwork --destination-port-ranges \* \
--protocol \* --access Allow
az network nsg rule create --nsg-name k8s-nsg -n k8s-deny-other-tcp --priority 4020 \
--source-address-prefixes 0.0.0.0/0 --source-port-ranges  \* \
--destination-address-prefixes 0.0.0.0/0 --destination-port-ranges \* \
--protocol TCP --access Deny
az network nsg rule create --nsg-name k8s-nsg -n k8s-deny-other-UDP --priority 4030 \
--source-address-prefixes 0.0.0.0/0 --source-port-ranges  \* \
--destination-address-prefixes 0.0.0.0/0 --destination-port-ranges \* \
--protocol UDP --access Deny
az network nsg rule create --nsg-name k8s-nsg -n k8s-allow-icmp --priority 4040 \
--source-address-prefixes 0.0.0.0/0 --source-port-ranges  \* \
--destination-address-prefixes 0.0.0.0/0 --destination-port-ranges \* \
--protocol \* --access Allow

```


List the firewall rules in the `k8s-nsg`:

```
az network nsg rule list --nsg-name k8s-nsg --output table
```

> output

```

Name                ResourceGroup       Priority  SourcePortRanges    SourceAddressPrefixes    SourceASG    Access    Protocol    Direction    DestinationPortRanges    DestinationAddressPrefixes    DestinationASG
------------------  ----------------  ----------  ------------------  -----------------------  -----------  --------  ----------  -----------  -----------------------  ----------------------------  ----------------
k8s-ssh-https       k8s-the-hard-way        1000  0-65535             0.0.0.0/0                None         Allow     Tcp         Inbound      22 443 6443              VirtualNetwork                None
k8s-internal        k8s-the-hard-way        1010  *                   VirtualNetwork           None         Allow     *           Inbound      *                        VirtualNetwork                None
k8s-deny-other-tcp  k8s-the-hard-way        4020  *                   0.0.0.0/0                None         Deny      Tcp         Inbound      *                        0.0.0.0/0                     None
k8s-deny-other-UDP  k8s-the-hard-way        4030  *                   0.0.0.0/0                None         Deny      Udp         Inbound      *                        0.0.0.0/0                     None
k8s-allow-icmp      k8s-the-hard-way        4040  *                   0.0.0.0/0                None         Allow     *           Inbound      *                        0.0.0.0/0                     None
```

Last step is to link this NSG to our subnet.

```
az network vnet subnet update \
--vnet-name k8s-the-hard-way \
--name k8s-cluster \
--network-security-group k8s-nsg
```

### Kubernetes Public IP Address

Allocate a static IP address + DNS name that will be attached to the external load balancer fronting the Kubernetes API Servers:

```
az network public-ip create -n k8s-ip --dns-name k8shard-ip --allocation-method Static
```

Verify the `kubernetes-the-hard-way` static IP address was created in your default compute region:

```
az network public-ip show -n k8s-ip --output json
```

> output

```
{
  "dnsSettings": {
    "domainNameLabel": "k8shard-ip",
    "fqdn": "k8shard-ip.westeurope.cloudapp.azure.com",
    "reverseFqdn": null
  },
  ...
  "ipAddress": "52.174.38.47",
  ...
  "location": "westeurope",
  "name": "k8s-ip",
  "provisioningState": "Succeeded",
  "publicIpAddressVersion": "IPv4",
  "publicIpAllocationMethod": "Static",
  "resourceGroup": "k8s-the-hard-way",
 ...
  "type": "Microsoft.Network/publicIPAddresses",
  "zones": null
}
```

## Virtual Machines

The virtual machines in this lab will be provisioned using [Ubuntu Server](https://www.ubuntu.com/server) 16.04, which has good support for the [cri-containerd container runtime](https://github.com/containerd/cri-containerd). Each compute instance will be provisioned with a fixed private IP address to simplify the Kubernetes bootstrapping process.

### Kubernetes Controllers
We'll create an availability set first to host the kubernetes controllers. 

```
az vm availability-set create --name k8s-ctrl
```

Create three VMs which will host the Kubernetes control plane:



```
for i in 0 1 2; do
  az vm create -n k8s-ctrl-${i} \
  --image ubuntults \
  --availability-set k8s-ctrl \
  --ssh-key-value publicKey.txt \
  --vnet-name k8s-the-hard-way \
  --subnet k8s-cluster \
  --private-ip-address 10.240.0.1${i} \
  --tags Role=Controller Project=k8s-the-hard-way \
  --os-disk-size-gb 200 --nsg "" \
  --size Standard_DS1_v2 --no-wait
done
```

### Kubernetes Workers

Each worker instance requires a pod subnet allocation from the Kubernetes cluster CIDR range. The pod subnet allocation will be used to configure container networking in a later exercise. The `pod-cidr`  tag will be used to expose pod subnet allocations to compute instances at runtime.

> The Kubernetes cluster CIDR range is defined by the Controller Manager's `--cluster-cidr` flag. In this tutorial the cluster CIDR range will be set to `10.200.0.0/16`, which supports 254 subnets.

Create three VMs in an availability set which will host the Kubernetes worker nodes:

```
az vm availability-set create --name k8s-worker
for i in 0 1 2; do
  az vm create -n k8s-worker-${i} \
  --image ubuntults \
  --availability-set k8s-worker \
  --ssh-key-value publicKey.txt \
  --vnet-name k8s-the-hard-way \
  --subnet k8s-cluster \
  --private-ip-address 10.240.0.2${i} \
  --tags pod-cidr=10.200.${i}.0/24 Role=Controller Project=k8s-the-hard-way \
  --os-disk-size-gb 200 --nsg "" \
  --size Standard_DS1_v2 --no-wait
done

```

### Verification

List the compute instances in your resource group:

```
az vm list --output table
```

> output

```
Name          ResourceGroup     PowerState    Location
------------  ----------------  ------------  ----------
k8s-ctrl-0    k8s-the-hard-way  VM running    westeurope
k8s-ctrl-1    k8s-the-hard-way  VM running    westeurope
k8s-ctrl-2    k8s-the-hard-way  VM running    westeurope
k8s-worker-0  k8s-the-hard-way  VM running    westeurope
k8s-worker-1  k8s-the-hard-way  VM running    westeurope
k8s-worker-2  k8s-the-hard-way  VM running    westeurope
```

Next: [Provisioning a CA and Generating TLS Certificates](04-certificate-authority.md)
