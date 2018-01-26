- [Azure Policy](#azure-policy)
    - [Prepare testing resource group](#prepare-testing-resource-group)
    - [Do not allow Public IP address to be created in resource group](#do-not-allow-public-ip-address-to-be-created-in-resource-group)
    - [Make sure tagging is enforced](#make-sure-tagging-is-enforced)
    - [Enforce specific subnet for VMs](#enforce-specific-subnet-for-vms)
    - [Allow only selected custom VM images](#allow-only-selected-custom-vm-images)

# Azure Policy
Azure Policy is used to create policies on Azure Resource Manager level when creating or updating resources. As rules are defined on ARM level they do apply to all means of management including APIs (like when using Terraform, Ansible or Azure SDK), in Portal, via Azure CLI or PowerShell. Rules are enforced by checking ARM JSON structures to include or exclude certain fields and values. With that you can enforce pretty much any policy such as preventing certain resources to be created, limit list of available images for VMs, restrict networking such as specific subnet, user default routes, assignment of public IP address or enforce tagging schemas.

Azure Policy is available in Preview in Azure Portal also, but in our demo we will use CLI as in either case we need to understand policy definitions that are in JSON. Azure Policy comes with se of built-in policies and a lot of examples. There is free Basic SKU that can be used for enforcements (hard limits). Standard SKU (in Preview) is paid service that can also be used in "auditing mode" that do not enforce user behavior (deployments do not fail), but full compliance reports are available including offending components. This is especially useful when dealing with existing deployments or when hard enforcement might be to restrictive as we cannot forseen or potential needs so want to track first and potentialy enforce later.

Azure Policy can be scoped and all resources bellow inherit it, but you can also exclude some resources implicitly. Lowest scope is Resource Group, higher scope is Subscription and you can also scope to Management Group (currently in Preview) which can reflect for example business unit with multiple subscriptions.

## Prepare testing resource group
We will use following resource group, vnet and subnet for testing.

```
az group create -n policy -l westeurope
az network vnet create -g policy -n myNet --address-prefix 10.0.0.0/16
az network vnet subnet create -g policy --vnet-name myNet -n sub1 \
    --address-prefix 10.0.1.0/24
az network vnet subnet create -g policy --vnet-name myNet -n sub2 \
    --address-prefix 10.0.2.0/24
```

## Do not allow Public IP address to be created in resource group
First let's check we are currently able to create Public IP in our resource group

```
az network public-ip create -g policy -n ip1
```

Now we will define policy to not allow ceratin resources (oppositie policy when you want to allow only what is listed is pre-defined in Azure already, but we want to do opposite). Policy rules are in doNotAllowResources.rules.json file. Basically if ARM see any request regarding specific type of resource, it will be denied. We do not hardcode list of resources here, but rather define this as array parametr in doNotAllowResources.params.definition.json so we decide what to block later during assignment of policy.

```
az policy definition create -n doNotAllowResources \
    --display-name "Do not allow resources" \
    --mode all \
    --rules doNotAllowResources.rules.json \
    --params doNotAllowResources.params.definition.json
```

We will now assign this policy to our resource group. Input parameters (in our case with single denied resource being Public IP) are in doNotAllowResources.params-ip.json.

```
az policy assignment create -n noPublicIP-rg-policy \
    --display-name "Do not allow creation of Public IP in resource group" \
    --policy doNotAllowResources \
    --params doNotAllowResources.params-ip.json \
    --resource-group policy \
    --sku free
```

We can now attempt to create Public IP which will be denied. Also note that since ip1 was created before policy was in effect it is not destroyed automatically. If you want your policies to rather be in auditing mode you can use standard sku to get such reporting withou strong enforcement.

```
az network public-ip create -g policy -n ip2

Resource 'ip2' was disallowed by policy. Policy identifiers: '[{"policyDefinitionId":"/subscriptions/a0f4a733-4fce-4d49-b8a8-d30541fc1b45/providers/Microsoft.Authorization/policyDefinitions/doNotAllowResources","policyAssignmentId":"/subscriptions/a0f4a733-4fce-4d49-b8a8-d30541fc1b45/providers/Microsoft.Authorization/policyAssignments/noPublicIP-rg-policy"}]'.
```

## Make sure tagging is enforced
Sometimes we might want all resources in Azure to include metadata specific for organization. Azure allows to use tags with any key and value. Let's say that in our case for billing purposes we need to include tag organization (it, finance, marketing) and environment (dev, test, prod). Also in our environment we always want to have contact tag present so other administrators know whome to call if questions about resourceshould arise. We will now create policy that checks that all resources being deployed in our resource group include proper tags.

Checkout policy rules definition in tagging.rules.json. Create policy and assign it to our resource group.

```
az policy definition create -n tagging \
    --display-name "Ensure company tags are included" \
    --mode all \
    --rules tagging.rules.json
```

We will now assign this policy to our resource group. Input parameters (in our case with single denied resource being Public IP) are in doNotAllowResources.params-ip.json.

```
az policy assignment create -n tagging-rg-policy \
    --display-name "Ensure company tags are included in resource group policy" \
    --policy tagging \
    --resource-group policy \
    --sku free
```

Let's try to create resource, for example NIC, without tags.

```
az network nic create -g policy --vnet-name myNet --subnet sub1 -n myNIC

Resource 'myNIC' was disallowed by policy. Policy identifiers: '[{"policyDefinitionId":"/subscriptions/a0f4a733-4fce-4d49-b8a8-d30541fc1b45/providers/Microsoft.Authorization/policyDefinitions/tagging","policyAssignmentId":"/subscriptions/a0f4a733-4fce-4d49-b8a8-d30541fc1b45/resourceGroups/policy/providers/Microsoft.Authorization/policyAssignments/tagging-rg-policy"}]'.
```

Let's now try with proper tagging

```
az network nic create -g policy --vnet-name myNet --subnet sub1 -n myNIC \
    --tags environment=dev organization=finance contact='tomas@mydomain.com'
```

## Enforce specific subnet for VMs
In traditional environment systems administrators and developers are not responsible (usualy even not allowed) for creating networks. That is handled by network/security team and particular workload can be deployed only into predefined networking environment.

To mimic this policy in Azure we will now create rules that will lock VMs within scope (resource group, subscription or more subscriptions) into specific precreated subnet.

First let's create policy definition. In order to easily reuse it with different scopes and subnets we are using array parameters here, so actual list of subnet IDs is defined during policy assignment, not during its definition.

```
az policy definition create -n lockSubnet \
    --display-name "Lock VMs to specified Subnet ID" \
    --mode all \
    --rules lockSubnet.rules.json \
    --params lockSubnet.params.definition.json
```

We will now assign this policy to our resource group. 

```
export subnetId=$(az network vnet subnet show -g policy --vnet-name myNet -n sub1 --query id -o tsv)
az policy assignment create -n lockSubnet-rg-policy \
    --display-name "Lock VMs to specified Subnet ID in resource group policy" \
    --policy lockSubnet \
    --resource-group policy \
    --sku free \
    --params "{'subnetId': {'value': ['$subnetId']}}"
```

Creating NIC bounded to different subnet should fail.

```
az network nic create -g policy --vnet-name myNet --subnet sub2 -n myNIC2 \
    --tags environment=dev organization=finance contact='tomas@mydomain.com'
```

While correct subnet should be fine.

```
az network nic create -g policy --vnet-name myNet --subnet sub1 -n myNIC3 \
    --tags environment=dev organization=finance contact='tomas@mydomain.com'
```

## Allow only selected custom VM images
In some restricted environments central IT might want to provide curated hardened images (for example with company security settings, security software, deployment and monitoring tools preinstalled) and forbid usage of any other images. We can define policy to enforce that.

In my subscription I have following custom images:

```
az image list -g muj-image-katalog -o table
Location    Name               ProvisioningState    ResourceGroup
----------  -----------------  -------------------  -----------------
westeurope  image-z-vhd        Succeeded            muj-image-katalog
westeurope  muj-linux-image    Succeeded            muj-image-katalog
westeurope  muj-windows-image  Succeeded            muj-image-katalog
```

I want to select only images containing "muj" string. To get array of custom images IDs we can use followig az command with query string and output it as json array.

```
export images=$(az image list -g muj-image-katalog --query "[?contains(name,'muj')].id" -o json)
```

Now we are going to create policy definition

```
az policy definition create -n approvedImagesOnly \
    --display-name "Approved images only" \
    --mode all \
    --rules approvedImagesOnly.rules.json \
    --params approvedImagesOnly.params.definition.json
```

With our stored array of custom images IDs we can assign policy to resource group "policy".

```
export params=$(echo '{"imageIds":{"value":'$images'}}')
az policy assignment create -n approvedImagesOnly-rg-policy \
    --display-name "Approved images only in resource group" \
    --policy approvedImagesOnly \
    --params "$params" \
    --resource-group policy \
    --sku free
```

We can now test creating VM with unapproved vs. approved image

```
az vm create -n myUnapprovedVM -g policy \
    --image UbuntuLTS \
    --tags environment=dev organization=finance contact='tomas@mydomain.com' \
    --public-ip-address "" \
    --vnet-name myNet \
    --subnet sub1 \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password

The template deployment failed because of policy violation. Please see details for more information.
```

As we should see deployment violates policy and is failing. Let's try with approved image.

```
export imageId=$(az image show -n muj-linux-image -g muj-image-katalog --query id -o tsv)
az vm create -n myApprovedVM -g policy \
    --image $imageId \
    --tags environment=dev organization=finance contact='tomas@mydomain.com' \
    --public-ip-address "" \
    --vnet-name myNet \
    --subnet sub1 \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password
```