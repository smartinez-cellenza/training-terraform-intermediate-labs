# Setup environment

## Lab overview

In this lab, you will learn how to deploy multiple resources of the same type in a single block.

## Objectives

After you complete this lab, you will be able to:

-   Create multiple Storage Accounts in a single block
-   Understand how Terraform handles loops

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
- **key**                  = "deploymultiplesa.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Deploy multiple Storage Accounts using a set

In the **variables.tf** file, add a new variable

```hcl
variable "storage_account_names" {
    type = set(string)
    description = "List of Storage Accounts to create"
}
```

> This variable is a set of string. It can only contains unique values.

> We will use this variable to handle the names of the Storage Account to create

in the *configuration/dev* folder, add to the *dev.tfvars* file the variable value :

```hcl
storage_account_names = ["a_unique_storage_account_name","another_unique_storage_account_name","a_third_unique_storage_account_name"]
```

> Storage Accounts have a public FQDN, based on their names. The Storage Account names should be globally unique accross all Azure Resources

In the *main.tf* file, add the following **resource** block to reference the feature Resource Group

```hcl
resource "azurerm_storage_account" "example" {
  for_each                 = var.storage_account_names
  name                     = each.key
  resource_group_name      = data.azurerm_resource_group.self.name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "GRS"
}
```

> Notice the for_each attribute. Its value is the set we added in the variables

> The name attribute of the Storage Account is each.key, referencing the occurence of the item in the set used in the for_each argument


Run the following commands to deploy Storage Accounts :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that the Storage Accounts have been created

Run the following command

```powershell
terraform state list
```

> This command can be used to list all the resources present in the tfstate and display their internal name

The name of the resources is suffixed with the Storage Account name.

Update the attribute name of the azurerm_storage_account block

```hcl
name                     = each.value
```

Run the following commands to deploy Storage Accounts :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Terraform does not identify any change. Since the value in the for_each is a set, **each.key** and **each.value** are the same

Remove the Storage Account using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

### Exercise 3: Deploy multiple Storage Accounts using a map

In the **variables.tf** file, update the **storage_account_name** definition

```hcl
variable "storage_account_names" {
  type = map(object({
    location                 = string
    account_replication_type = string
  }))
  description = "List of Storage Accounts to create"
}
```

> This variable is now a map of object

> Using a map allow to specify multiple attributes for each item. In most real life case, not only name needs to be defined, but also various attributes.


in the *configuration/dev* folder, update the *dev.tfvars* file content. Remove the previous value of **storage_account_names** and add the following block

```hcl
storage_account_names = {
    "a_unique_storage_account_name" = {
        location = "westeurope"
        account_replication_type = "GRS"
    }
    "another_unique_storage_account_name" = {
        location = "westeurope"
        account_replication_type = "LRS"
    }
    "a_third_unique_storage_account_name" = {
        location = "northeurope"
        account_replication_type = "GRS"
    }
}
```

> Storage Accounts have a public FQDN, based on their names. The Storage Account names should be globally unique accross all Azure Resources

In the *main.tf* file, replace the **resource** block by this one

```hcl
resource "azurerm_storage_account" "example" {
  for_each                 = var.storage_account_names
  name                     = each.key
  resource_group_name      = data.azurerm_resource_group.self.name
  location                 = each.value.location
  account_tier             = "Standard"
  account_replication_type = each.value.account_replication_type
}
```

> Notice the for_each attribute. Its value is the map we added in the variables

> The name attribute of the Storage Account is each.key.

> location used the location attribute we defined in the map. This attribute can now be variabilised

> account_replication_type used the account_replication_type attribute we defined in the map. This attribute can now be variabilised


Run the following commands to deploy Storage Accounts :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that the Storage Accounts have been created

Run the following command

```powershell
terraform state list
```

> This command can be used to list all the resources present in the tfstate and display their internal name

The name of the resources is suffixed with the Storage Account name.

Remove the Storage Account using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```
