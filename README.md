# Terraform Remote Backend
Provisions a storage account with a container for Terraform state file.

![visio.png](/img/visio.png)


---------------------------------------------------------
## Terraform documentation
### Requirements

| Name | Version |
|------|---------|
| <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) | ~>2.0 |
| <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) | ~>2.0 |

### Providers

| Name | Version |
|------|---------|
| <a name="provider_azurerm"></a> [azurerm](#provider\_azurerm) | 2.99.0 |

### Modules

No modules.

### Resources

| Name | Type |
|------|------|
| [azurerm_resource_group.terraformbackend](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) | resource |
| [azurerm_storage_account.terraform_backend](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account) | resource |
| [azurerm_storage_container.terraform_backend](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_container) | resource |

### Inputs

| Name | Description | Type | Default | Should Default be changed? |
|------|-------------|------|---------|:--------:|
| <a name="input_container_access_type"></a> [container\_access\_type](#input\_container\_access\_type) | n/a | `string` | `"blob"` | no |
| <a name="input_location"></a> [location](#input\_location) | n/a | `string` | `"norwayeast"` | no |
| <a name="input_resource_group_name"></a> [resource\_group\_name](#input\_resource\_group\_name) | n/a | `string` | `"terraformbackend"` | yes |
| <a name="input_storage_account_name"></a> [storage\_account\_name](#input\_storage\_account\_name) | n/a | `string` | `"terraformbackendxxyygg"` | yes |
| <a name="input_storage_account_replication_type"></a> [storage\_account\_replication\_type](#input\_storage\_account\_replication\_type) | n/a | `string` | `"LRS"` | no |
| <a name="input_storage_account_tier"></a> [storage\_account\_tier](#input\_storage\_account\_tier) | n/a | `string` | `"Standard"` | no |
| <a name="input_storage_container_name"></a> [storage\_container\_name](#input\_storage\_container\_name) | n/a | `string` | `"terraformbackend"` | yes |

### Outputs

No outputs.


-----------------------------------------------------------------------------


## Resources

- [Azure Storage Account](https://docs.microsoft.com/en-us/azure/storage/common/storage-account-overview) contains Azure Storage data objects, including blobs, file shares, queues, tables, and disks. 
- [Azure Storage Container](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-blobs-introduction ) organizes blobs. The container is used to store the terraform state file.



### Terraform State

Terraform keeps track of the infrastructure that is provisioned with Terraform in a *state file* called *terraform.tfstate*. When running Terraform, it compares the infrastructure code to the actual resources in Azure.  

For projects with multiple team members, the state file should be accessible for everyone in the team. The state file can be kept in Azure and referenced in the Terraform code. This is a good practice, since team members and provisioning pipelines can reference the same state file. The state file will be in a locked state when a team member or a pipeline runs *terraform apply*, stopping others from doing changes to the infrastructure at the same time. Since the state file can contain secrets, keeping it in Azure storage will reduce the risk of loosing secrets. This approach also ensures that the latest version of the state file is used, while also keeping older versions for the ability to roll back.



#### Provisioning the backend with Terraform

This is a two-stage operation since the backend, consisting of a resource group with storage account and storage container, must exist before it is referenced in backend.tf.

The stages to achieve this is:

1. Provision storage account and container without backend block

   This will create a terraform.tfstate locally

2. After provisioning; add the backend block with references to the resources just created

   Run terraform init. This will offer to migrate the local state file into Azure storage



![1.png](/img/1.png)

**modules/storage-account/main.tf**

```
resource "azurerm_resource_group" "terraform" {
    name     = var.resource_group_name
    location = var.location
}

resource "azurerm_storage_account" "terraformbackend" {
    name                = var.storage_account_name
    resource_group_name = azurerm_resource_group.terraform.name
    location            = var.location
    account_tier        = "Standard"
    account_replication_type = "LRS"
}

resource "azurerm_storage_container" "terraformbackend" {
  name                  = var.storage_container_name 
  storage_account_name  = azurerm_storage_account.terraformbackend.name
  container_access_type = var.container_access_type
}
```

**backend.tf without backend block**

```
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "~>2.0"
    }
  }
}

provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

Terraform apply will provision the resources and create a state file that is stored locally.

In an infrastructure with modules and different environments (test/stage/prod), the command to provision the backend (storage account, container): 

```
terraform plan|apply --target=module.storage-acount --var-file="stage/stage.tfvars"
```

Next, the *backend block* is inserted/uncommented with the values from the resources just created.

**backend.tf with backend block**

```
terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "~>2.0"
    }
  }
   backend "azurerm" {
         resource_group_name  = "terraform"
         storage_account_name = "terraformbackend"
         container_name       = "terraformbackend"
         key                  = "terraform.tfstate"
     }
}

provider "azurerm" {
  features {}

  subscription_id = var.subscription_id
}
```

A new Terraform init will now ask if you want to move the state file to remote backend. From now on, resources that is provisioned will be managed by the state file in Azure storage.

![3.png](/img/3.png)



![2.png](/img/2.png)







The backend in Azure is now managed by a local state. This means that it cannot be removed with *terraform destroy* without first going back to the local state. This is a positive effect, since the backend should continue to exist while other resources changes. 



#### References

- https://docs.microsoft.com/en-us/azure/developer/terraform/store-state-in-azure-storage?tabs=azure-cli
- Terraform documentation storage account: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_account
- Terraform documentation storage container: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/storage_container



**terraform backend**

```
terraform {
  backend "azurerm" {
    resource_group_name  = "terraformbackend"
    storage_account_name = "<storage account name>"
    container_name       = "terraformbackend"
    key                  = "terraform.tfstate"
  }
}
```

**access_key**; the storage access key is also needed. It is possible to specify it in the backend block, but to prevent the access key to be written to disk, use environment variable:

```
# Azure CLI
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $STORAGE_ACCOUNT_NAME --query '[0].value' -o tsv)
export ARM_ACCESS_KEY=$ACCOUNT_KEY
```

The access key can also be stored in Azure Key Vault.















