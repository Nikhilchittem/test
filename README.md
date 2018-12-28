# Overview

This project is about to deploy required resources in azure using ARM templates for the services in jvdc. 

# Jeppesen-VDC Services

This project contains an ARM template that deploys a Linux Availability set VMs for services into the project network in desired subnet. It also contains bootstrap script to configure deployed instances and wrapper script to deploy desired environment as a whole. 


## Prerequisites

* Global key-vault populated with service principal.
* Name of the Resource Group that the VNET resides in.
* Name of the existing VNET and Subnet you want to deploy the Availability set and required virtual machines into.
* Load balancer service that which balance the incoming traffic to the backend VMs in Availability set. 
* Existing storage that contains scrips and their locations in blob storage. 
*Database (DB) service (PostgreSQL) to store and retrieve the data that processed by the applications relied in application server machines.
*IBM message queue Service that used to decouple applications and services from each other and to transfer data between different applications and services using messages. 
* Bootstrap script (Automation) that contains bash script which configure the VMs to the desired state sequentially.
* Wrapper Script that deploys the required service tier in desired location & environment/stage like (dev, test, stage, prod).

## Service Tier Deployment

The service tier deployment architecture and required infrastructure resources are given below.
[Here (Architoon)]
* Key-Vault Service
* VM Availability set with Load Balancer
* DB (PostgreSQL Database)
* IBM Messaging Queue Service 

### Initialize Global Keyvault

The service tier deployment architecture leverages the project global keyvault as a security and discovery mechanism during service deployment. 

* Initialize the project global keyvault with service specific credentials. One time only for a subscription deployment.


#### Global Keyvault Secrets

Parameter Name     | Type      | Description
-------------------|-----------|-----------------------------------------
kvName             | String    | Name of the Vault.
tenantId           | String    | Tenant Id of the subscription.
objectId           | String    | Object Id of the AD user.
keysPermissions    | Array     | Permissions to keys in the vault. Valid values are: all, create, import, update, get, list, delete, backup, restore, encrypt, decrypt, wrapkey, unwrapkey, sign, and verify. 
secretsPermissions | Array     | Permissions to secrets in the vault. Valid values are: all, get, set, list, and delete. 
skuName            | String    | SKU for the vault.
enableVaultForDeployment | bool | Specifies if the vault is enabled for a VM deployment.
enableVaultForDiskEncryption | bool | Specifies if the azure platform has access to the vault for enabling disk encryption scenarios. 
enabledForTemplateDeployment | bool | Specifies whether Azure Resource Manager is permitted to retrieve secrets from the key vault. 

#### Azure CLI 2.0 (Deployment)

```bash
az login
az account set --subscription jeppesen-vdc
az group deployment create --resource-group {project-resource-group/VNET} --template-file prereqs/keyvault.azuredeploy.json --parameters prereqs/keyvault.params.json --verbose 
```

### Linux Availability Set VMs with Load Balancer Parameters

The service template is abstracted to support deployments in any region, stage, or project.
The template acquires a set of parameters from the global keyvault and assumes those values have been initialized from previous [Initialize Global Keyvault](#initialize-global-keyvault) section.

Parameter Name      | Type      | Description
--------------------|-----------|----------------------------------------
loadbalancerSku     | String    | The type of load balancer to deploy. Valid values: Basic, Standard.
lbRules             | Object    | Load Balancer Rules.
probeProperties     | Object    | Probe Properties
vmSku               | string    | Size of VMs in the cluster.
diskType            | String    | The Storage type of the data Disks.
adminUsername       | String    | The name of the adminUser retrieve from keyvault.
adminPassword       | SecureString    | Values of the adminPassword retrieve from keyvault.
linuxUsername       | String    | The Username at OS (Linux) level retrieve from key vault.
linuxPassword       | String    | Value of the Password at OS (Linux) level retrieve from key vault.
storageAccountName   | String    | Name of the storage account where the scripts resides in.
storageAccountKey    | String    | Storage account key value that retrieve in Keyvault.
vnetRGName           | String    | Virtual Network resource group name that where we deoploy our VM Availability set.
vnetName             | String    | Virtual Network name where VMs deploys in.
subnetName           | String    | Subnet name particular defined service deploys in.
availabilitySetName  | String    | Name of the VM Availability Set.
linuxGroupName       | String    | Name of the OS group.
vmName               | String    | Virtual Machine name.
numberOfInstances    | Int       | Desired number of instances in Availability set as a cluster.
imagePublisher       | String    | name of the image publisher/provider.
imageOffer           | String    | image type.
imageSku             | String    | Version of the image.
platformFaultDomainCount |  Int    | Required no of fault domains.
platformUpdateDomainCount |  Int    | Required no of update domains.


#### Deploy Command Line

Credential controls are managed by the global keyvault. Before the service can be deployed the global keyvault must be initialized with required service credentials. Many parameters managed by the global keyvault are shared among all stage services and are provisioned upon stage creation.

#### Azure CLI 2.0 (Deployment)

```bash
az login
az account set --subscription jeppesen-vdc
az group deployment create --resource-group {project-resource-group/VNET} --template-file prereqs/{ServiceVMname}.azuredeploy.json --parameters prereqs/{ServiceVMname}.params.json --verbose 
```
: {ServiceVMname} = jhasvm/jpdcvm/jsivm/wxd


### Integrate Database (PostgreSQL) to the application server VMs.

The Service template deploys PostgreSQL database in Jeppesen-VDC subscription for JHAS/JPDC application services. The template uses the existing networking configuration that are in use within the Jeppesen-VDC subscription. In-order to execute the template make sure to update the Key Vault and Networking Parameters.

#### Initialization 

Update the KeyValut parameters in the PostgreSQL deployment template.

Parameter Name      | Type      | Description
--------------------|-----------|----------------------------------------
adminUsername       | String    | The name of the adminUser retrieve from keyvault.
adminPassword       | SecureString    | Values of the adminPassword retrieve from keyvault.
dbserverName        | String    | The name of the PostgreSQL server.
dbName              | String    | The name of the Azure database.
storageMB           | Int       | Storage size of the database.
skuFamily           | String    | Azure database for PostgreSQL sku family.
skuCapacity         | Int       | compute capacity in vCores (2,4,8,16,32) of Azure database for PostgreSQL.
skuTier             | String    | pricing tier of Azure database for PostgreSQL.
skuName             | String    | sku name of Azure database for PostgreSQL.
postgresqlVersion   | String    | Version of the PostgreSQL service.
vnetRGName           | String    | Virtual Network resource group name that where we deploy our VM PostgreSQL server.
vnetName             | String    | Virtual Network name where PostgreSQL server deploys in.
subnetName           | String    | Subnet name.
virtualNetworkRuleName       | String    | Virtual Network Rule name. firewallRuleName             | String    | Firewall Rule name.
startIPaddress       | String    | designated IPaddress.
endIPaddress         | String    | designated IPaddress.
Location             | String    | Resource group location for all resources.

#### Azure CLI 2.0 (Deployment)

```bash
az login
az account set --subscription jeppesen-vdc
az group deployment create --resource-group {project-resource-group/VNET} --template-file prereqs/db.azuredeploy.json --parameters prereqs/db.params.json --verbose 
```

### Initializing IBM Messaging Service Queue.

The Service template deploys IBMQ in Jeppesen-VDC subscription for application services to decouple applications and services from each other and to transfer data between different applications and services using messages that implements workflows for message ordering.  

Parameter Name      | Type      | Description
--------------------|-----------|----------------------------------------
vmSku               | String    | Size of VMs in the cluster.
diskType            | String    | The Storage type of the data. adminUsername       | String    | The name of the adminUser retrieve from keyvault.
adminPassword       | SecureString    | Values of the adminPassword retrieve from keyvault.
storageAccountName   | String    | Name of the storage account where the scripts resides in.
storageAccountKey    | String    | Storage account key value that retrieve in Keyvault.
vnetRGName           | String    | Virtual Network resource group name that where we deoploy our VM Availability set.
vnetName             | String    | Virtual Network name where VMs deploys in.
subnetName           | String    | Subnet name particular defined service deploys in.
diagnosticStorageAccountType  | String    | Storage Account type for diagnostics.
availabilitySetName  | String    | Name of the VM Availability Set.
linuxGroupName       | String    | Name of the OS group.
vmName               | String    | Virtual Machine name.
imagePublisher       | String    | name of the image publisher/provider.
imageOffer           | String    | image type.
imageSku             | String    | Version of the image.
platformFaultDomainCount |  Int    | Required no of fault domains.
platformUpdateDomainCount |  Int    | Required no of update domains.


#### Azure CLI 2.0 (Deployment)

```bash
az login
az account set --subscription jeppesen-vdc
az group deployment create --resource-group {project-resource-group/VNET} --template-file prereqs/ibmq.azuredeploy.json --parameters prereqs/ibmq. params.json --verbose 
```

### Bootstrap Script (Bash)

Bootstrap script meant to configure the deployed virtual machines in the desired subnets while running the templates by adding VM extension to the service VM templates jpdcvm/jhasvm/jsivm/wxdvm. Whereas the bootstrap script location given in which they resides in Azure blob storage.

Locations:

“https://jvdcappjhasautomationnp.blob.core.windows.net/infra-builds/jhas-auto/jhasbootstrap.sh” 
“https://jvdcappjhasautomationnp.blob.core.windows.net/infra-builds/jpdc-auto/jpdcbootstrap.sh” 
“https://jvdcappjhasautomationnp.blob.core.windows.net/infra-builds/jsi-auto/jsibootstrap.sh” 
“https://jvdcappjhasautomationnp.blob.core.windows.net/infra-builds/wxd-auto/wxdbootstrap.sh” 


### Wrapper Script (Bash)

Deploy.sh bash script file execute the keyvault/VM/PostgreSQL & IBM queue arm templates.

This file has two input values which t & e
 -t [all or (keyvault jpdcvm jhasvm jsivm wxdvm db ibmq)] Select which tier want to deploy either all services together or simply tier vault. 
 -e [np/prod/dev/test/stage/perf/intg] Select the environment for deployment.

 Note: Based on Environment Input, parameter files will be updated for specific environment name.


#### Azure CLI 2.0 (Deployment)

```bash
az login
az account set --subscription jeppesen-vdc
./deploy -e np -t all 
or 
./deploy -e dev -t vault
```
: {Service tier} = [all or (keyvault jpdcvm jhasvm jsivm wxdvm db ibmq)]
: {environment} = [np/prod/dev/test/stage/perf/intg]



### Service Components

Component           | Resource Type      | Description
--------------------|--------------------|-------------------------------
Keyvault            | Azure Key Vault    | Store service configuration and sensitive data like credentials and passwords.
VM Availability Set | Azure Availability set | to increase availability and reliability of Virtual Machines.  
Load Balancer       | Azure Load Balancer | Load balancer to distribute load across instances in Availability set.
Database            | Azure Database     | A fully managed, scalable PostgreSQL relational database with high availability and security built-in.
IBM MQ Service      | Azure Service Bus | to transfer data between different applications and services using messages. 

### Network Topography

In addition to the standard infrastructure NSG configuration baseline requires:

Service     | Port | Direction | Source               | Destination          | Description
------------|-------|-----------|----------------------|----------------------|-------------------------------------
HTTP        | 80   | Inbound   | Virtual Network      | Any  |Allow HTTP
HTTPS       | 443   | Inbound   | Virtual Network      | Any  | Allow HTTPS

### Diagram

![Full stack Deployment Diagram](img/link)

## Post Installation Procedure

None

## Support

### Status

 Status can be viewed in the Azure Portal.

## Troubleshooting

### Logs

DA Azure logs are found at the following locations

Bootstrap log - `/tmp/bootstrap.log`

