# Setup environment

## Lab overview

In this lab, you will learn how to deploy resources to multiple subscription in the same template.

## Objectives

After you complete this lab, you will be able to:

-   Deploy two Virtual Networks on two subscriptions and create a peering between them
-   Understand how Terraform handles multiple subscriptions using provider and alias

## Instructions

### Before you start

- Ensure Terraform (version >= 1.0.0) is installed and available from system's PATH.
- Ensure Azure CLI is installed.
- Check your access to the Azure Subscriptions and Resource Groups provided for this training.

### Exercise 1: Setup your environment

Create a Storage Account with a container nammed tfstates to store the tfstate file.

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
- **key**                  = "deploytomultiplesub.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Configure a new provider for the feature Subscription

In the **provider.tf** file, add a new *provider* block, with the following configuration

```hcl
provider "azurerm" {
  skip_provider_registration = true
  features {}
  alias = "feature"
  subscription_id = var.feature_subscription_id
}
```

> Notice the new alias attribute. This alias will be used later on other blocks to reference this provider explicitly

> Notice the subscription_id attribute. It allows to target a specific subscription.
> The subscription Id should have been set directly in the attribute. Using a variable allows to specify it when running Terraform commands

In the *variables.tf* file, add the following block for the feature Subscription configuration

```hcl
variable "feature_subscription_id" {
    type = string
    description = "Feature Subscription Id"
}
```

in the *configuration/dev* folder add to the *dev.tfvars* file the variable value :

```hcl
feature_subscription_id = "Id_of_the_feature_subscription"
```

> The feature subscription can be found using the Azure Portal, and navigating to the the Resource Group tagged with attribute layer and value feature

In the *main.tf* file, add the following **data** block to reference the feature Resource Group

```hcl
data "azurerm_resource_group" "feature_rg" {
  name = var.resource_group_name
  provider = azurerm.feature
}
```

> The name of the Resource Group is the same in the main and feature Subscriptions

> The provider attribute allows to target the desired Subscription. It is built using the provider type and the provider alias

Run the following commands to ensure both Resource Groups are available :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
terraform apply -var-file="..\configuration\dev\dev.tfvars"
```

> At this point we only added data blocks, so Terraform indicate that no changes are required

### Exercise 3: Create the Virtual Networks

Add the following **resource** block to create the Virtual Network in the **main** Subscription

```hcl
resource "azurerm_virtual_network" "main_vnet" {
  name                = "main-network"
  address_space       = ["10.0.0.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

> We do not explicitly set the provider. The default one, without alias, will be used, targeting the main subscription

Run the following commands to deploy this Virtual Network :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

Add the following **resource** block to create the Virtual Network in the **feature** Subscription

```hcl
resource "azurerm_virtual_network" "feature_vnet" {
  name                = "feature-network"
  address_space       = ["10.0.1.0/24"]
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
  provider            = azurerm.feature
}
```

> We explicitly set the provider using its alias, targeting the feature subscription

Run the following commands to deploy this Virtual Network :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check in the Azure Portal that both Virtual Networks have been created, one in the main Subscription, one in the feature Subscription

### Exercise 4: Create peering

The peering must be created on both side (not visible when peering is done using the Azure Portal)

Add the following **resources** block to create the peering

```hcl
resource "azurerm_virtual_network_peering" "peer-main-to-feature" {
  name                      = "peer-main-to-feature"
  resource_group_name       = data.azurerm_resource_group.self.name
  virtual_network_name      = azurerm_virtual_network.main_vnet.name
  remote_virtual_network_id = azurerm_virtual_network.feature_vnet.id
}

resource "azurerm_virtual_network_peering" "peer-feature-to-main" {
  name                      = "peer-feature-to-main"
  resource_group_name       = data.azurerm_resource_group.feature_rg.name
  virtual_network_name      = azurerm_virtual_network.feature_vnet.name
  remote_virtual_network_id = azurerm_virtual_network.main_vnet.id
  provider                  = azurerm.feature
}
```

> Notice the provider attribute in the second peering block

Run the following commands to deploy this Virtual Network :

```powershell
terraform apply -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

> Check the Network peering in the Azure Portal


### Exercise 5: Remove resources

Providers allow to target a different subscription. But they can also use different credential for authentication

Destroy the created resources

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars" -auto-approve
```

