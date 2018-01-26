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

Go back to PC where limited user is logged in and check that we can read and regenerate storage keys, but cannot modify or delete storage account.

```
az storage account keys list -g rg-blue -n mystorage192837465
az storage account keys renew -g rg-blue -n mystorage192837465
```

## Fine grained access control with custom roles
TBD

## Privileged Identity Management with just-in-time access
TBD

## Managed Service Identity for automation bots
TBD

