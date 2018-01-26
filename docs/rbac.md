- [Role Based Access Control](#role-based-access-control)
    - [Prepare environment and create new account](#prepare-environment-and-create-new-account)
    - [Basic RBAC](#basic-rbac)
    - [Fine grained access control with built-in roles](#fine-grained-access-control-with-built-in-roles)
    - [Fine grained access control with custom roles](#fine-grained-access-control-with-custom-roles)
    - [Privileged Identity Management with just-in-time access](#privileged-identity-management-with-just-in-time-access)
    - [Managed Service Identity for automation bots](#managed-service-identity-for-automation-bots)

# Role Based Access Control

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
TBD

## Managed Service Identity for automation bots
TBD

