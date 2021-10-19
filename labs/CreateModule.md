# Setup environment

## Lab overview

In this lab, you will learn how to create and consume a module.

## Objectives

After you complete this lab, you will be able to:

-   Create a module to manage Storage Account
-   Understand how to use Terraform modules

## Instructions

### Before you start

- Ensure Terraform (version >= 1.0.0) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment

In your *main* Resource Group (the on tagged with attribute **layer** with value **main**), create a Storage Account, with a Blob container nammed **tfstate**

Clone the repository https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup

```bash
git clone https://github.com/smartinez-cellenza/training-terraform-intermediate-labs-setup.git
cd training-terraform-intermediate-labs-setup
```

> This template contains a basic Terraform project configuration

In the *configuration/dev* folder, update the **backend.hcl** file :

- **resource_group_name**  = "the_name_of_your_main_resource_group"
- **storage_account_name** = "the_name_of_the_storage_account_you_just_created"
- **container_name**       = "tfstates"
- **key**                  = "module.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Create and consume a module for Storage Account

In this exercice, we will create a module for Storage Account.

This module will create a Storage Account and a blob container.

It will take as input the name of the Resource Group, the name for the storage (and append a prefix and suffix) and the name of the blob container.

It will outputs the name of the storage.

#### Create folder and files for the module

The source of this module will be a local path.

Create a new folder, nammed **modules**, and in this folder create a folder **storageaccount**

```powershell
cd src
mkdir modules
cd modules
mkdir storageaccount
```

> This folder will contains all the terraform files for the module

In the **storageaccount** create the 3 following files

- **main.tf** : It will contains resources template
- **variables.tf** : It will contains variable blocks
- **outputs.tf** : It will contains the outputs of the module

In the **variables.tf** file add the following content

```hcl
variable "resource_group_name"  {
    type = string
    description = "Name of the Resource Group"
}

variable "storage_name"  {
    type = string
    description = "Name of the Storage Account to create"
}

variable "container_name" {
    type = string
    description = "Name of the Blob Container to create"
}
```

> This variables values should be passed when a consummer instanciate this module

In the **main.tf** file add the following content

```hcl
resource "azurerm_storage_account" "sa" {
  name                     = "stomodule${var.storage_name}lab"
  resource_group_name      = var.resource_group_name
  location                 = "westeurope"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "container" {
  name                  = var.container_name
  storage_account_name  = azurerm_storage_account.sa.name
  container_access_type = "private"
}
```

> This is a simple template to create a Storage Account and a Container using the variables provided

In the **outputs.tf** file add the following content

```hcl
output "storage_account_full_name" {
  value = azurerm_storage_account.sa.name
}
```

> The full Storage Account name will be shared to the module consummer. The module consummer can't access all the Storage Account Resource properties

#### Consume this module

In the **main.tf** file in the root module (the src folder) add the following content

```hcl
module "storage" {
  source = "./modules/storageaccount"

  resource_group_name = data.azurerm_resource_group.self.name
  storage_name = "a_unique_name_goes_here"
  container_name = "content"
}
```

> This module block use the folder we just created as source

Run the following commands to initialize backend (in the src folder of the root module) :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

> Notice the Initializing modules step in the init logs

Run the following commands to create resources :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> Notice the identifier of the created resources, they are prefix with module.storage
> module.storage.azurerm_storage_account.sa for the Storage Account
> module.storage.azurerm_storage_container.container for the Container

### Exercise 3: Use module outputs

In this exercice, we will create a Storage Account Queue for the Storage we have provisionned.

We will create this resource in the root module.

In the **main.tf** file of the root module, add the following content to create the Queue

```hcl
resource "azurerm_storage_queue" "queue" {
  name                 = "mysamplequeue"
  storage_account_name = module.storage.azurerm_storage_account.sa.name
}
```

Run the following commands to create resources :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> Notice the error. We are not able to access the Storage Account properites directly.

> That's why we setup an ouptut for this module

Replace the content we added with this block

```hcl
resource "azurerm_storage_queue" "queue" {
  name                 = "mysamplequeue"
  storage_account_name = module.storage.storage_account_full_name
}
```

Run the following commands to create resources :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> Notice the success in applying the template

> We used the output of the module in order to retrieve the Storage Account name

### Exercise 4: Implicit dependency

In this exercice, we will create a Storage Account Queue for the Storage we have provisionned.

We will create both resources during the same apply of the template, to ensure implicit dependency is well managed

Run the following commands to create resources :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> Notice the success of the Apply

> Dependencies have been respected.

We can see in the logs

```text
module.storage_dependency.azurerm_storage_account.sa: Creating...
module.storage_dependency.azurerm_storage_account.sa: Still creating... [10s elapsed]
module.storage_dependency.azurerm_storage_account.sa: Still creating... [20s elapsed]
module.storage_dependency.azurerm_storage_account.sa: Creation complete after 25s [id=/subscriptions/8d2a1f75-7232-45f5-8b46-a0e16f40c8d8/resourceGroups/sylvain.martinez/providers/Microsoft.Storage/storageAccounts/stomoduletestsma2lab]
azurerm_storage_queue.queue_dependency: Creating...
module.storage_dependency.azurerm_storage_container.container: Creating...
azurerm_storage_queue.queue_dependency: Creation complete after 0s [id=https://stomoduletestsma2lab.queue.core.windows.net/mysamplequeue]
module.storage_dependency.azurerm_storage_container.container: Creation complete after 0s [id=https://stomoduletestsma2lab.blob.core.windows.net/content]
```

The creation of the Container and the Queue has been triggered after the Storage Account Creation. Not all resources of a module should be deployed to be able to begin deployment of other resources of the root module.

### Exercise 5: Explicit dependency

Modules support Explicit dependency using the **depends_on** meta-attribute.

In this exercice we're going to add a dependency to our module, and manage it with an explicit dependency.

In the **verions.tf** file, edit the **required_providers** and add a new provider, time. The required_providers should now be

```hcl
required_providers {
    azurerm = ">= 2.75.0"
    time    = ">= 0.7"
  }
```

> We are going to use this provider to delay the creation of dependant resources

in the **main.tf** file add a new **resource block** for a **null_resource**

```hcl
resource "null_resource" "fake" {}
```

> The null_resource is a particular type of Resource. It is agnostic of any Cloud Provider, and does not deploy anything. It's lifecycle is still managed by Terraform

> This null_resource will be our dependency

in the **main.tf** file add a new **resource block** for a **time_sleep**

```hcl
resource "time_sleep" "delay" {
  depends_on = [null_resource.fake]

  create_duration = "10s"
}
```

> The time_sleep resource is provided by the time provider

> It allows to delay the creation of resources. It will simulate the fact that our dependency might be long to deploy and make the explicit dependency more visible

> The time_sleep resource might be helpfull in real world scenario, for instance when we manage an Azure Public DNS Zone and we want to delay the provisioning of resources after a record creation.

in the **main.tf** file add a new **module block**

```hcl
module "storage_explicit_dependency" {
  source = "./modules/storageaccount"

  resource_group_name = data.azurerm_resource_group.self.name
  storage_name = "a_unique_name_goes_here"
  container_name = "content"

  depends_on = [time_sleep.delay]
}
```

> Notice the explicit dependency to the time_sleep resource

Run the following commands to donwload the time module:

```powershell
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

Run the following command to create resources

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

In the execution logs, we can see the explicit dependency action

```text
null_resource.fake: Creating...
null_resource.fake: Creation complete after 0s [id=8690051244183217110]
time_sleep.delay: Creating...
time_sleep.delay: Still creating... [10s elapsed]
time_sleep.delay: Creation complete after 10s [id=2021-10-19T01:16:08Z]
module.storage_explicit_dependency.azurerm_storage_account.sa: Creating...
module.storage_explicit_dependency.azurerm_storage_account.sa: Still creating... [10s elapsed]
module.storage_explicit_dependency.azurerm_storage_account.sa: Still creating... [20s elapsed]
module.storage_explicit_dependency.azurerm_storage_account.sa: Creation complete after 26s [id=/subscriptions/8d2a1f75-7232-45f5-8b46-a0e16f40c8d8/resourceGroups/sylvain.martinez/providers/Microsoft.Storage/storageAccounts/stomoduletestsma3lab]
```

> The first created resource is the null_resource

> The time_sleep resource is then created. Its creation duration is specified to long for 10 secondes

> After the complete creation of the time_sleep resource, the Storage Account module creation starts, respecting the explicit dependency

### Exercise 6: Create with a loop

The modules support the for_each argument, and its behavior is the same than for resource block.

In this exercice we're going to create 3 Storage Account module using a for_each meta-argument.

In the **variables.tf** file of the root module, add a new variable

```hcl
variable "storage_accounts" {
  type = map(object({
    container_name           = string
  }))
  description = "List of Storage Accounts to create"
}
```

> This map of object has a single attribute, the container name.

> The name of the Storage Account will be the key of each map item.

In the **dev.tfvars** set the value for this variable

```hcl
storage_accounts = {
    "a_unique_name_goes_here" = {
        container_name = "content"
    }
    "a_unique_name_goes_here" = {
        container_name = "content"
    }
    "a_unique_name_goes_here" = {
        container_name = "content"
    }
}
```

In the *main.tf* file of the root module add the following content

```hcl
module "storage_loop" {
  source = "./modules/storageaccount"

  for_each  = var.storage_accounts
  resource_group_name = data.azurerm_resource_group.self.name
  storage_name = each.key
  container_name = each.value.container_name
}
```

> the for_each meta-argument allows to create multiple Storage Account module

> We use each.key and each.value to provide informations for the module inputs

Run the following commands to initialize module:

```powershell
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

Run the following command to create resources

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

In the logs we can see the identifier generated

```text
module.storage_loop["testsma6"].azurerm_storage_account.sa: Creating...
module.storage_loop["testsma5"].azurerm_storage_account.sa: Creating...
module.storage_loop["testsma4"].azurerm_storage_account.sa: Creating...
module.storage_loop["testsma6"].azurerm_storage_account.sa: Still creating... [10s elapsed]
module.storage_loop["testsma4"].azurerm_storage_account.sa: Still creating... [10s elapsed]
module.storage_loop["testsma5"].azurerm_storage_account.sa: Still creating... [10s elapsed]
module.storage_loop["testsma5"].azurerm_storage_account.sa: Still creating... [20s elapsed]
module.storage_loop["testsma6"].azurerm_storage_account.sa: Still creating... [20s elapsed]
module.storage_loop["testsma4"].azurerm_storage_account.sa: Still creating... [20s elapsed]
module.storage_loop["testsma5"].azurerm_storage_account.sa: Creation complete after 23s [id=/subscriptions/8d2a1f75-7232-45f5-8b46-a0e16f40c8d8/resourceGroups/sylvain.martinez/providers/Microsoft.Storage/storageAccounts/stomoduletestsma5lab]
module.storage_loop["testsma5"].azurerm_storage_container.container: Creating...
module.storage_loop["testsma5"].azurerm_storage_container.container: Creation complete after 0s [id=https://stomoduletestsma5lab.blob.core.windows.net/content]
module.storage_loop["testsma4"].azurerm_storage_account.sa: Creation complete after 24s [id=/subscriptions/8d2a1f75-7232-45f5-8b46-a0e16f40c8d8/resourceGroups/sylvain.martinez/providers/Microsoft.Storage/storageAccounts/stomoduletestsma4lab]
module.storage_loop["testsma4"].azurerm_storage_container.container: Creating...
module.storage_loop["testsma4"].azurerm_storage_container.container: Creation complete after 0s [id=https://stomoduletestsma4lab.blob.core.windows.net/content]
module.storage_loop["testsma6"].azurerm_storage_account.sa: Creation complete after 26s [id=/subscriptions/8d2a1f75-7232-45f5-8b46-a0e16f40c8d8/resourceGroups/sylvain.martinez/providers/Microsoft.Storage/storageAccounts/stomoduletestsma6lab]
module.storage_loop["testsma6"].azurerm_storage_container.container: Creating...
module.storage_loop["testsma6"].azurerm_storage_container.container: Creation complete after 0s [id=https://stomoduletestsma6lab.blob.core.windows.net/content]
```

### Exercise 7: Import an existing resource

It is possible to import an existing Azure Resource to manage its lifecycle with Terraform, using a module.

In this exercice we're going to create a Storage Account using the portal, and import it in tfstate and manage its lifecycle with a module.

Using the Azure Portal, create a Storage Account

> The Storage Account name and location should match the module configuration

Get the resource ID (you can find it in the Storage Endpoints tab)

In the **main.tf** file of the root module, add the following content

```
module "storage_import" {
  source = "./modules/storageaccount"

  resource_group_name = data.azurerm_resource_group.self.name
  storage_name = "the_variable_name_of_the_storage_you_created"
  container_name = "content"
}
```

Run the following commands to initialize module:

```powershell
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

Run the following commands to import the existing Storage Account in Terraform state:

```powershell
$resourceID = "the_resource_id_you_get_from_portal"
terraform import -var-file="..\configuration\dev\dev.tfvars" module.storage_import.azurerm_storage_account.sa $resourceID
```

> Notice the successful import

Run the following commands to create the missing container for the imported Storage Account:

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> Notice the Container Creation

> You may also find in-update for the Storage Account if the creation using Azure Portal used different configuration

### Exercise 8: Remove Resources

Remove all the created resources using the destroy command

```hcl
terraform destroy -var-file="..\configuration\dev\dev.tfvars"
```
