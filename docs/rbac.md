- [Role Based Access Control](#role-based-access-control)
    - [Prepare environment and create new account](#prepare-environment-and-create-new-account)
    - [Basic RBAC](#basic-rbac)
    - [Fine grained access control with built-in roles](#fine-grained-access-control-with-built-in-roles)
    - [Fine grained access control with custom roles](#fine-grained-access-control-with-custom-roles)
    - [Privileged Identity Management with just-in-time access](#privileged-identity-management-with-just-in-time-access)
    - [Managed Service Identity for automation bots](#managed-service-identity-for-automation-bots)

# Role Based Access Control
Azure RBAC can be used to define access rights for individual users or service principals (service accounts). 

**Roles** are definitions of types of resources, actions and read vs read/write that user with this role can access. There are many useful roles built-in including generic Owner, Contributor and Reader as well as resource specific such as Virtual Machine Contributor and many more. You can also define custom roles in which you list part of API tree you want to allow or forbid. As roles are handled on Azure Resource Manager level it is applied for all management types including portal, CLI, PowerShell, SDKs or APIs used by automation tools such as Terraform or Ansible.

**Assignment** of users to roles is best handled with your identity management so by leveraging Security Groups in AD synchronized with AAD. You can also users directly without groups.

**Scope** can be defined as whole subscription and assignments are inherited by lower level objects, but you can overide this on lower levels. Another typical scope is Resource Group, so user can become owner of his workload, can read and monitor other Resource Groups while might have no visibility into others. You can also lock scope to single individual resource.

## Prepare environment and create new account
First we will create some resource groups to play with.

```
az group create -n rg-blue -l westeurope
az group create -n rg-red -l westeurope
az group create -n rg-green -l westeurope
```

In my subscription I am not allowed to create additional user accounts so for purpose of this demo I will user service prinicpals. It is special (or you can say "service") account very often used for scripting and automation. Let's create service principal with certificate-based authentication and skip assignment to default role.

```
az ad sp create-for-rbac -n myLimitedAccount \
    --skip-assignment \
    --create-cert

Please copy /home/tomas/tmpgr_em7l5.pem to a safe place. When run 'az login' provide the file path to the --password argument
```

Note client self-signed certificate has been created.

## Basic RBAC
In this section we will explore basic RBAC with simple built-in read/write and readonly roles and assign those on resource group scope.

As we are going to use our PC with limited user account let's open cloud shell (eg. shell.azure.com) to have full admin access there (so do not have have login back and forward all the time on our PC).

With admin access we can see all groups containing rg string, so all three colors.

```
az group list -o table --query "[?contains(name,'rg')]"
```

Check out built-in roles.

```
az role definition list -o table
```

We will start with generic roles Owner, Contributor (full access but cannot grant rights to other users) and Reader. We will now grant access to our limited user as Reader for blue group and Contributor for green group.

```
export rgblue=$(az group show -n rg-blue --query id -o tsv)
az role assignment create --role Reader \
    --assignee "http://myLimitedAccount" \
    --scope $rgblue

export rggreen=$(az group show -n rg-green --query id -o tsv)
az role assignment create --role Contributor \
    --assignee "http://myLimitedAccount" \
    --scope $rggreen
```

On our PC lets now connect as limited user. Note that we need to specify tenant id which you can get in GUI or by following command in cloud shell that will get tenant id of your first subscription.

```
az account list --query "[0].tenantId" -o tsv
```

Copy Tenand ID over and store it in env variable on your PC. Now we can connect.

```
export tenantId="0123456-0123456-0123456"
az login -u "http://myLimitedAccount" \
    --service-principal \
    -p /home/tomas/tmpgr_em7l5.pem \
    --tenant $tenantId \
    --allow-no-subscriptions
```

Check what groups are visible to this user. We should see only blue and green, but no red.
```
az group list -o table --query "[?contains(name,'rg')]"
```

Logged as regular user (in cloud shell) let's create resources in green and blue group.

```
az network public-ip create -g rg-green -n adminIP1
az network public-ip create -g rg-blue -n adminIP2
```

As limited user make sure both resources are visible, but not other resources (from different resource groups).

```
az resource list -o table
```

Now let's check limited user is able to create resources only in green resource group.

```
az network public-ip create -g rg-green -n limitedUserIP1
az network public-ip create -g rg-blue -n limitedUserIP1
```

## Fine grained access control with built-in roles
In previous sections we have already seen list of built-in roles:

```
az role definition list -o table
```

There are roles to limit user to manage Virtual Machines only, Web Sites, API Management, Storage accounts etc. We will now use role "Storage Account Key Operator Service Role" for blue resource group where are already Reader (which does not allow to read or regenerate storage keys) so our user is able to read and regenerate storage keys, but cannot do any other operations with storage account.

In cloud shell (full access) we will now assign this role to our user for red resource group only.

```
export rgblue=$(az group show -n rg-blue --query id -o tsv)
az role assignment create --role "Storage Account Key Operator Service Role" \
    --assignee "http://myLimitedAccount" \
    --scope $rgblue
```

Still in cloud shell let's create storage account.

```
az storage account create -n mystorage192837465 \
    -g rg-blue \
    -l westeurope \
    --sku Standard_LRS
```

Go back to PC where limited user is logged in and check that we can read and regenerate storage keys, but cannot modify or delete storage account. At the same time you should not be able to delete this account.

```
az storage account keys list -g rg-blue -n mystorage192837465
az storage account keys renew -g rg-blue -n mystorage192837465
az storage account delete -g rg-blue -n mystorage192837465
```

## Fine grained access control with custom roles
Suppose we have special need for access control that does not fit any built-on role. For example we want to get role definition that allows user to start, stop and restart VMs, but cannot do anything else with it (this is close to built-in Virtual Machine Contributor, but more restricted).

When defining custom roles we need to provide list of actions that reflects paths in Azure Resource Manager. You can use wildcards to provide access to complete subtree (for example all Compute related APIs) and also specify full access or read only. In our case we want to allow read on virtual machine resource, but only start, restart and stop (deallocate) actions are allowed. 

Check roleStartStopVM.json.example and modify subscription ID to reflect your environment. This is to define where this definition is visible (it does not mean you need to assign it then on the some scope, it can be subset).

Create custom definition.

```
az role definition create --role-definition @roleStartStopVM.json
```

Now - in cloud shell create VM in blue resource group.

```
az vm create -n myVirtualMachine -g rg-blue \
    --image UbuntuLTS \
    --public-ip-address "" \
    --nsg "" \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password
```

Once VM is up and running go to your PC where limited user is logged in and try to stop this VM. This should fail because currently you are only Reader on this resource group.

```
az vm deallocate -n myVirtualMachine -g rg-blue
```

Let's assign our user custom role we have created before (in cloud shell).

```
export rgblue=$(az group show -n rg-blue --query id -o tsv)
az role assignment create --role "VM Start Stop Restart only" \
    --assignee "http://myLimitedAccount" \
    --scope $rgblue
```

Now let's try again from PC with limited account logged in.

```
az vm deallocate -n myVirtualMachine -g rg-blue
```

This was successful. Make sure you are not able to do other actions such as delete this VM.

```
az vm delete -n myVirtualMachine -g rg-blue -y
```

## Privileged Identity Management with just-in-time access
In previous demo we have used account with full access to manage rights for our limited account. Azure AD Privileged Identity Management can provide sophisticated workflow around the same principles such as:
- One-time rights escalation: ordinary user with limited rights can get privileged access on just-in-time bases for limited time; typical example is person handling service ticket, administrator assigned for planned maintenance etc.
- Every rights escalation request is properly audited
- You can define approval workflow, for example IT manager has to approve request for read/write escalation for contractor account

Azure AD PIM is managed in Azure Portal and is not shown in this demo.

## Managed Service Identity for automation bots
Often there is need for something like service account - identity that is not bound to specific user as it is used by automation robot (such as Ansible or Terraform), Bash or PowerShell script or application accessing cloud resources such as SQL database, Key Vault with secrets or Service Bus for messaging. It is bad practice to include any credentials in code or stored in easily accessible files, but generation and delivery of proper credentials manualy can be painful. Azure Managed Service Identity automates this process by automatically creating identity for your VM so you can access certain Azure services from this VM without hardcoding any credentials.

We are going to create new VM and assign Managed Service Identity. We will use role Contributor and scope its rights only to resource msitest (for example if you deploy automation tool such as Terraform you might want to scope MSI to whole subscription).

```
az group create -n msitest -l westeurope
az group show -n msitest --query [].id -o tsv
az vm create -n exampleVM \
    -g msitest \
    --image UbuntuLTS \
    --public-ip-address-dns-name msitest192837 \
    --admin-username tomas \
    --admin-password Azure12345678 \
    --authentication-type password \
    --assign-identity \
    --role Contributor \
    --scope $(az group show -n msitest --query id -o tsv)
```

Let's connect to VM.

```
ssh tomas@msitest192837.westeurope.cloudapp.azure.com
```

We can install Azure CLI and login using Managed Service Identity. We should see resources in msitest resource group and can create some new.

```
echo "deb [arch=amd64] https://packages.microsoft.com/repos/azure-cli/ wheezy main" | \
     sudo tee /etc/apt/sources.list.d/azure-cli.list
sudo apt-key adv --keyserver packages.microsoft.com --recv-keys 52E16F86FEE04B979B07E28DB02C46DF417A0893
sudo apt-get install apt-transport-https -y
sudo apt-get update && sudo apt-get install azure-cli -y

az login --msi
az resource list -o table
```

If you are using APIs and SDKs to access Azure Resource Manager directly from your code (such as Python script), you can access token yourself (note that you are not getting some user name and password that you could easily transfer outside of this VM - you are getting only time limited access token):

```
curl http://localhost:50342/oauth2/token --data "resource=https://management.azure.com/" -H Metadata:true
```

With this token we can now use REST API directly. For example this is how we can deallocate VM from within it - for example when VM finish some job it can deallocate itself so you stop paying for resources.

```
sudo apt install jq -y
export token=$(curl -s http://localhost:50342/oauth2/token --data "resource=https://management.azure.com/" -H Metadata:true | jq -r ".access_token")
export subscriptionId=$(cat ~/.azure/azureProfile.json | jq -r ".subscriptions[0].id")


curl -X POST https://management.azure.com/subscriptions/$subscriptionId/resourceGroups/msitest/providers/Microsoft.Compute/virtualMachines/$(hostname)/deallocate?api-version=2017-12-01 -d "" -H "Authorization: Bearer $token"
```

With MSI account in Azure Active Directlory is created and you can use it for other purposes as well. For example with Azure SQL you can turn on AAD authentication and your MSI account can be used to access database.