# Setup environment

## Lab overview

In this lab, you will learn how to deploy multiple nested blocks of the same kind.

## Objectives

After you complete this lab, you will be able to:

-   Create multiple IP restrictions for an App Service
-   Understand how Terraform handles dynamic blocks

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
- **key**                  = "dynamic.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Deploy multiple IP Restrictions on an App Service

In the **variables.tf** file, add the **ip_restrictions** definition

```hcl
variable "ip_restrictions" {
  type = map(object({
    ip_address = string
    priority   = number
    action     = string
  }))
  description = "List of IP restrictions"
}
```

> This variable is a map of object

> Using a map allow to specify multiple attributes for each item


in the *configuration/dev* folder, update the *dev.tfvars* file content. Add the following lines

```hcl
ip_restrictions = {
    "allow_ip_first" = {
        ip_address = "10.0.1.0/24"
        priority = 100
        action = "Allow"
    }
    "allow_ip_second" = {
        ip_address = "10.0.2.0/24"
        priority = 110
        action = "Allow"
    }
    "allow_ip_third" = {
        ip_address = "10.0.3.0/24"
        priority = 120
        action = "Allow"
    }
}
```

> An IP Restriction block requires ip_address, priority, action and name

In the *main.tf* file, add the following content

```hcl
resource "azurerm_app_service_plan" "self" {
  name                = "dynamicblockasp"
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name


  sku {
    tier     = "Standard"
    size     = "S1"
  }
}

resource "azurerm_app_service" "self" {
  name                = "testsmatfdylol"
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
  app_service_plan_id = azurerm_app_service_plan.self.id

  site_config {

    scm_use_main_ip_restriction = false

    dynamic "ip_restriction" {
      for_each = var.ip_restrictions
      content {
        ip_address = ip_restriction.value.ip_address
        name       = ip_restriction.key
        priority   = ip_restriction.value.priority
        action     = ip_restriction.value.action
      }
    }
  }
}
```

> Notice the dynamic block of ip_restriction kind

> The for_each argument is set to the map we defined

> The content of each block is created using attributes specified for each occurence


Run the following commands to deploy resources :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that the App Service is deployed.

> Check in the Azure Portal that the App Service has access restriction matching the provided rules

Remove resources using the command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```
