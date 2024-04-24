terraform {
  required_providers {
    azurerm = {
      source = "hashicorp/azurerm"
      version = "3.100.0"
    }
        azuread = {
      source = "hashicorp/azuread"
      version = "2.48.0"
    }
  }
}

provider "azurerm" {
  features {}
}

resource "azurerm_resource_group" "databricks" {
  name     = "databricks"
  location = "Norway East"
}

resource "azurerm_databricks_workspace" "learndbw" {
  name                = "learndbw"
  resource_group_name = azurerm_resource_group.databricks.name
  location            = azurerm_resource_group.databricks.location
  sku                 = "premium"
  customer_managed_key_enabled = true
  custom_parameters {
    public_subnet_network_security_group_association_id = azurerm_subnet_network_security_group_association.nsg.id
    private_subnet_network_security_group_association_id = azurerm_subnet_network_security_group_association.nsg.id
    public_subnet_name  = "public-subnet"
    vnet_address_prefix                                  = "10.139"
    storage_account_name                                 = "dbstoragegt5ljd6qpjozc" 
    no_public_ip                                         = true
    storage_account_sku_name                             = "Standard_GRS"
    private_subnet_name                                  = "private-subnet"
    virtual_network_id = "/subscriptions/e137fc4c-1992-4cdf-853a-ffb99dee7ffe/resourceGroups/databricks/providers/Microsoft.Network/virtualNetworks/vnet-injected-vnet"
  }
  managed_disk_cmk_key_vault_key_id = "https://cmk-kv-sjdgs.vault.azure.net/keys/cmk-key-lesss-days/2833709dc1cb4a8c8fb5c11d80fdec87"
  managed_services_cmk_key_vault_key_id = "https://cmk-kv-sjdgs.vault.azure.net/keys/cmk-key-lesss-days/2833709dc1cb4a8c8fb5c11d80fdec87"
}

resource "azurerm_subnet_network_security_group_association" "nsg" {
  subnet_id                 = data.azurerm_subnet.databricks.id
  network_security_group_id = data.azurerm_network_security_group.databricks.id
}

import{
    to = azurerm_subnet_network_security_group_association.nsg
    id = "/subscriptions/e137fc4c-1992-4cdf-853a-ffb99dee7ffe/resourceGroups/databricks/providers/Microsoft.Network/virtualNetworks/vnet-injected-vnet/subnets/public-subnet"
}

data "azurerm_virtual_network" "databricks" {
  name                = "vnet-injected-vnet"
  resource_group_name = azurerm_resource_group.databricks.name
}

data "azurerm_subnet" "databricks" {
  name                 = "public-subnet"
  virtual_network_name = data.azurerm_virtual_network.databricks.name
  resource_group_name  = azurerm_resource_group.databricks.name
}

data "azurerm_network_security_group" "databricks" {
  name                = "databricksnsgd34ku72wslacg"
  resource_group_name = azurerm_resource_group.databricks.name
}

data "azuread_service_principal" "azuredatabricks" {
  display_name = "AzureDatabricks"
}

resource "azurerm_role_assignment" "encryption"{
    principal_id = data.azuread_service_principal.azuredatabricks.object_id
    scope = azurerm_key_vault.cmk_kv.id
    role_definition_name = "Key Vault Crypto Service Encryption User"
}

resource "azurerm_databricks_workspace_root_dbfs_customer_managed_key" "learndbw" {

  workspace_id     = azurerm_databricks_workspace.learndbw.id
  key_vault_key_id = data.azurerm_key_vault_key.databricks.id
}

data "azurerm_key_vault_key" "databricks" {
  name         = "cmk-key-lesss-days"
  key_vault_id = azurerm_key_vault.cmk_kv.id
}

resource "azurerm_role_assignment" "encryption_sa" {
  scope = azurerm_key_vault.cmk_kv.id
  principal_id    = azurerm_databricks_workspace.learndbw.storage_account_identity[0].principal_id
      role_definition_name = "Key Vault Crypto Service Encryption User"
}



data "azurerm_client_config" "current" {}

resource "azurerm_key_vault" "cmk_kv" {
  name                        = "cmk-kv-sjdgs"
  location                    = azurerm_resource_group.databricks.location
  resource_group_name         = azurerm_resource_group.databricks.name
  enabled_for_disk_encryption = true
  tenant_id                   = data.azurerm_client_config.current.tenant_id
  soft_delete_retention_days  = 90
  purge_protection_enabled    = true
  enable_rbac_authorization   = true

  sku_name = "standard"
}

resource "azurerm_key_vault_key" "cmk_01" {
  lifecycle {
    ignore_changes = [expiration_date]
  }
  name         = "cmk-key-lesss-days"
  key_vault_id = azurerm_key_vault.cmk_kv.id
  key_type     = "RSA"
  key_size     = 2048

  key_opts = [
    "decrypt",
    "encrypt",
    "sign",
    "unwrapKey",
    "verify",
    "wrapKey",
  ]

  rotation_policy {
    automatic {
      time_after_creation = "P360D"
    }
    
    expire_after         = "P367D"
    notify_before_expiry = "P29D"
  }
}
