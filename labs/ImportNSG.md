# Setup environment

## Lab overview

In this lab, you will learn how to import resource and let Terraform manage their lifecycle.

## Objectives

After you complete this lab, you will be able to:

-   Import a Network Security Group
-   Understand how Terraform handles existing infrastrucutre using import

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
- **key**                  = "import.tfstate"

> This template use Partial backend configuration. The Storage Account just created is used for tfstate file persistence

In the *configuration/dev* folder, update the *dev.tfvars* file :

- **resource_group_name** = "the_name_of_your_main_resource_group"

### Exercise 2: Deploy a Network Security Group using AZ CLI

We will create a Network Security Group using Azure CLI.

> This NSG will be our existing infrastructure. It has not been deployed using Terraform.

Run the following command to configure Subsription and Resource Group and login to Azure

```powershell
az login
az account set --subscription "id_of_the_training_subscription"
$resourceGroupName = "your_resource_group_name"
```

Run the following command to create a Network Security Group

```powershell
az network nsg create --name "importdemo-nsg" --resource-group $resourceGroupName --location "westeurope"
```

We will add rule to this Network Security Group

Create a deny rule over TCP for a specific IP adress range

```powershell
az network nsg rule create -g $resourceGroupName `
                           --nsg-name "importdemo-nsg" `
                           -n "SampleRule" `
                           --priority 100 `
                           --direction Outbound `
                           --source-address-prefixes '*' `
                           --source-port-ranges '*' `
                           --destination-address-prefixes '*' `
                           --destination-port-ranges '*' `
                           --access Allow `
                           --protocol Tcp
```

> Ensure rules are created using Azure Portal

### Exercise 3: Import the Network Security Group in tfstate

Run the following commands to initialize backend :

```powershell
az login
az account set --subscription "the_main_subscription_id"
$env:ARM_SUBSCRIPTION_ID="the_main_subscription_id"
cd src
terraform init -backend-config="..\configuration\dev\backend.hcl" -reconfigure
```

We will now prepare the terraform template.

Add a new variable for the Network Security Group name if the **variables.tf** file

```hcl
variable "network_security_group_name" {
  type = string
  description = "Network Security Group Name"
}
```

Add the value for this variable in the **dev.tfvars** file

```hcl
network_security_group_name = "importdemo-nsg"
```

Update the template in the **main.tf** file

```hcl
resource "azurerm_network_security_group" "nsg" {
  name                = var.network_security_group_name
  location            = var.location
  resource_group_name = data.azurerm_resource_group.self.name
}
```

> Our template is now ready for importing the resource in tfstate

Get the Network Security Group Resource Id from the Azure Portal.

Navigate to the deployed Network Security Group in you subscription, and copy the Resource ID available in the Settings -> Properties pane.

Run the following command to import resource

```powershell
$nsgResourceId = "the_resource_id_you_just_copied"
terraform import -var-file="..\configuration\dev\dev.tfvars" azurerm_network_security_group.nsg $nsgResourceId
```

> Notice the successful import message

> The resource is now imported in tfstate. It does not mean that its configuration is matching what is present in template.

> The resource is now managed by Terraform. It can be updated or deleted during Terraform apply or destroy command.

Generate a plan using the following command

```powershell
terraform plan -var-file="..\configuration\dev\dev.tfvars"
```

> Notice that the plan is indicating that no change are required.

### Exercise 4: Import the Network Security Group in tfstate

The Network Security Group Rule is a distinct resource in Terraform, that's why since we only imported the Network Security Group itself, then plan generated did not indicate any change.

In the **main.tf** file add a new resource block for the Network Security Group Rule

```hcl
resource "azurerm_network_security_rule" "example" {
  name                        = "SampleRule"
  priority                    = 100
  direction                   = "Outbound"
  access                      = "Allow"
  protocol                    = "Tcp"
  source_port_range           = "*"
  destination_port_range      = "*"
  source_address_prefix       = "*"
  destination_address_prefix  = "*"
  resource_group_name         = data.azurerm_resource_group.self.name
  network_security_group_name = azurerm_network_security_group.nsg.name
}
```
We now need to get the Azure Resource ID for the rule.

It's not possible to get the Resource ID from the Azure portal for a Network Security Group Rule. Terraform documentation might be helpfull in order to get the Resource ID pattern for a specific type of resource (https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/network_security_rule). Az CLI might also be used.

Run the following command to get the Resource ID using Az CLI

```powershell
az network nsg rule show -g $resourceGroupName --nsg-name "importdemo-nsg" -n "SampleRule"
```

In the json output, get and copy the "id" attribute value

We can now run the import command

```powershell
$ruleResourceId = "the_resource_id_you_just_got_from_az_cli"
terraform import -var-file="..\configuration\dev\dev.tfvars" azurerm_network_security_rule.example $ruleResourceId
```

> Notice the successful import message

> The resource is now imported in tfstate. It does not mean that its configuration is matching what is present in template.

> The resource is now managed by Terraform. It can be updated or deleted during Terraform apply or destroy command.

### Exercise 5: Destroy the infrastructure

The Network Security Group and the rule are now managed by Terraform.

We can remove them using a destroy command

```powershell
terraform destroy -var-file="..\configuration\dev\dev.tfvars"
```

> Check in the Azure portal that the resources have been removed