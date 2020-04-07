# Test-Terraform

This is a test of some terraform.

## Sample 1

This is a sample.  What does it deploy?

```hcl
provider "azurerm" {
  version = "~>2.2.0"
  features {}
}

locals {
  storage_account_name = "sa${var.app_name}-${terraform.workspace}"
  resource_group_name  = "rg${var.app_name}-${terraform.workspace}"
}

resource "azurerm_resource_group" "rg" {
  name = local.resource_group_name
}

resource "azurerm_storage_account" "storage_account" {
  name                = local.storage_account_name
  resource_group_name = azurerm_resource_group.rg.name

  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"

  static_website {
    index_document     = "index.html"
    error_404_document = "404error.html"
  }
}
```