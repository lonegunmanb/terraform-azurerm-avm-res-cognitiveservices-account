<!-- BEGIN_TF_DOCS -->
# Azure AI Service example

This deploys an Azure AI service

```hcl
terraform {
  required_version = ">= 1.3.0"
  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
    random = {
      source  = "hashicorp/random"
      version = ">= 3.5.0, < 4.0.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = false
    }
    key_vault {
      purge_soft_delete_on_destroy    = true
      recover_soft_deleted_key_vaults = true
    }
  }
}

# This ensures we have unique CAF compliant names for our resources.
module "naming" {
  source  = "Azure/naming/azurerm"
  version = ">= 0.3.0"
}

# This is required for resource modules
resource "azurerm_resource_group" "this" {
  location = "East US"
  name     = "avm-res-aiservice-${module.naming.resource_group.name_unique}"
}

resource "random_string" "suffix" {
  length  = 5
  numeric = false
  special = false
  upper   = false
}

data "azurerm_client_config" "this" {}

resource "azurerm_user_assigned_identity" "this" {
  location            = azurerm_resource_group.this.location
  name                = "uai-such-aiservice"
  resource_group_name = azurerm_resource_group.this.name
}

resource "azurerm_key_vault" "this" {
  location                   = azurerm_resource_group.this.location
  name                       = "aisuchi${replace(random_string.suffix.result, "-", "")}"
  resource_group_name        = azurerm_resource_group.this.name
  sku_name                   = "premium"
  tenant_id                  = data.azurerm_client_config.this.tenant_id
  purge_protection_enabled   = true
  soft_delete_retention_days = 7

  access_policy {
    certificate_permissions = [
      "Create",
      "Delete",
      "DeleteIssuers",
      "Get",
      "Purge",
      "Update"
    ]
    key_permissions = [
      "Create",
      "Delete",
      "Get",
      "Purge",
      "Recover",
      "Update",
      "GetRotationPolicy",
      "SetRotationPolicy"
    ]
    object_id = data.azurerm_client_config.this.object_id
    secret_permissions = [
      "Delete",
      "Get",
      "Set",
    ]
    tenant_id = data.azurerm_client_config.this.tenant_id
  }
  access_policy {
    key_permissions = [
      "Get",
      "Create",
      "List",
      "Restore",
      "Recover",
      "UnwrapKey",
      "WrapKey",
      "Purge",
      "Encrypt",
      "Decrypt",
      "Sign",
      "Verify",
    ]
    object_id = azurerm_user_assigned_identity.this.principal_id
    secret_permissions = [
      "Delete",
      "Get",
      "Set",
    ]
    tenant_id = data.azurerm_client_config.this.tenant_id
  }
}

resource "azurerm_key_vault_certificate" "cert" {
  count = 3

  key_vault_id = azurerm_key_vault.this.id
  name         = "acchsmcert${count.index}"

  certificate_policy {
    issuer_parameters {
      name = "Self"
    }
    key_properties {
      exportable = true
      key_type   = "RSA"
      reuse_key  = true
      key_size   = 2048
    }
    secret_properties {
      content_type = "application/x-pkcs12"
    }
    lifetime_action {
      action {
        action_type = "AutoRenew"
      }
      trigger {
        days_before_expiry = 30
      }
    }
    x509_certificate_properties {
      key_usage = [
        "cRLSign",
        "dataEncipherment",
        "digitalSignature",
        "keyAgreement",
        "keyCertSign",
        "keyEncipherment",
      ]
      subject            = "CN=hello-world"
      validity_in_months = 12
      extended_key_usage = []
    }
  }
}

resource "azurerm_key_vault_managed_hardware_security_module" "this" {
  admin_object_ids                          = [data.azurerm_client_config.this.object_id]
  location                                  = azurerm_resource_group.this.location
  name                                      = "hsuiet${replace(random_string.suffix.result, "-", "")}"
  resource_group_name                       = azurerm_resource_group.this.name
  sku_name                                  = "Standard_B1"
  tenant_id                                 = data.azurerm_client_config.this.tenant_id
  purge_protection_enabled                  = true
  security_domain_key_vault_certificate_ids = [for cert in azurerm_key_vault_certificate.cert : cert.id]
  security_domain_quorum                    = 2
  soft_delete_retention_days                = 7
}

resource "random_uuid" "role_assignments_names" {
  count = 3
}

# this gives your service principal the HSM Crypto User role which lets you create and destroy hsm keys
resource "azurerm_key_vault_managed_hardware_security_module_role_assignment" "hsm_crypto_user" {
  name               = random_uuid.role_assignments_names[0].result
  principal_id       = data.azurerm_client_config.this.object_id
  role_definition_id = "/Microsoft.KeyVault/providers/Microsoft.Authorization/roleDefinitions/21dbd100-6940-42c2-9190-5d6cb909625b"
  scope              = "/keys"
  managed_hsm_id     = azurerm_key_vault_managed_hardware_security_module.this.id
}

# this gives your service principal the HSM Crypto Officer role which lets you purge hsm keys
resource "azurerm_key_vault_managed_hardware_security_module_role_assignment" "hsm_crypto_officer" {
  name               = random_uuid.role_assignments_names[1].result
  principal_id       = data.azurerm_client_config.this.object_id
  role_definition_id = "/Microsoft.KeyVault/providers/Microsoft.Authorization/roleDefinitions/515eb02d-2335-4d2d-92f2-b1cbdf9c3778"
  scope              = "/keys"
  managed_hsm_id     = azurerm_key_vault_managed_hardware_security_module.this.id
}

# this gives your service principal the HSM Crypto User role to UAI for wrap/unwrap operations
resource "azurerm_key_vault_managed_hardware_security_module_role_assignment" "uai_crypto_user" {
  name               = random_uuid.role_assignments_names[2].result
  principal_id       = azurerm_user_assigned_identity.this.principal_id
  role_definition_id = "/Microsoft.KeyVault/providers/Microsoft.Authorization/roleDefinitions/21dbd100-6940-42c2-9190-5d6cb909625b"
  scope              = "/keys"
  managed_hsm_id     = azurerm_key_vault_managed_hardware_security_module.this.id
}

resource "azurerm_key_vault_managed_hardware_security_module_key" "this" {
  key_opts       = ["decrypt", "encrypt", "sign", "unwrapKey", "verify", "wrapKey"]
  key_type       = "RSA-HSM"
  managed_hsm_id = azurerm_key_vault_managed_hardware_security_module.this.id
  name           = "hsmkeysuchi"
  key_size       = 2048

  depends_on = [
    azurerm_key_vault_managed_hardware_security_module_role_assignment.hsm_crypto_user,
    azurerm_key_vault_managed_hardware_security_module_role_assignment.hsm_crypto_officer,
    azurerm_key_vault_managed_hardware_security_module_role_assignment.uai_crypto_user
  ]
}

module "test" {
  source                      = "../../"
  kind                        = "AIServices"
  location                    = azurerm_resource_group.this.location
  name                        = "AIService-${module.naming.cognitive_account.name_unique}"
  resource_group_name         = azurerm_resource_group.this.name
  sku_name                    = "S0"
  is_hardware_security_module = true
  managed_identities = {
    system_assigned            = false
    user_assigned_resource_ids = toset([azurerm_user_assigned_identity.this.id])
  }
  customer_managed_key = {
    key_vault_resource_id = azurerm_key_vault_managed_hardware_security_module.this.id
    key_name              = azurerm_key_vault_managed_hardware_security_module_key.this.name
    user_assigned_identity = {
      resource_id = azurerm_user_assigned_identity.this.id
    }
  }
}
```

<!-- markdownlint-disable MD033 -->
## Requirements

The following requirements are needed by this module:

- <a name="requirement_terraform"></a> [terraform](#requirement\_terraform) (>= 1.3.0)

- <a name="requirement_azurerm"></a> [azurerm](#requirement\_azurerm) (~> 4.0)

- <a name="requirement_random"></a> [random](#requirement\_random) (>= 3.5.0, < 4.0.0)

## Resources

The following resources are used by this module:

- [azurerm_key_vault.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault) (resource)
- [azurerm_key_vault_certificate.cert](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_certificate) (resource)
- [azurerm_key_vault_managed_hardware_security_module.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_managed_hardware_security_module) (resource)
- [azurerm_key_vault_managed_hardware_security_module_key.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_managed_hardware_security_module_key) (resource)
- [azurerm_key_vault_managed_hardware_security_module_role_assignment.hsm_crypto_officer](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_managed_hardware_security_module_role_assignment) (resource)
- [azurerm_key_vault_managed_hardware_security_module_role_assignment.hsm_crypto_user](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_managed_hardware_security_module_role_assignment) (resource)
- [azurerm_key_vault_managed_hardware_security_module_role_assignment.uai_crypto_user](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/key_vault_managed_hardware_security_module_role_assignment) (resource)
- [azurerm_resource_group.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/resource_group) (resource)
- [azurerm_user_assigned_identity.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/resources/user_assigned_identity) (resource)
- [random_string.suffix](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/string) (resource)
- [random_uuid.role_assignments_names](https://registry.terraform.io/providers/hashicorp/random/latest/docs/resources/uuid) (resource)
- [azurerm_client_config.this](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/data-sources/client_config) (data source)

<!-- markdownlint-disable MD013 -->
## Required Inputs

No required inputs.

## Optional Inputs

No optional inputs.

## Outputs

No outputs.

## Modules

The following Modules are called:

### <a name="module_naming"></a> [naming](#module\_naming)

Source: Azure/naming/azurerm

Version: >= 0.3.0

### <a name="module_test"></a> [test](#module\_test)

Source: ../../

Version:

<!-- markdownlint-disable-next-line MD041 -->
## Data Collection

The software may collect information about you and your use of the software and send it to Microsoft. Microsoft may use this information to provide services and improve our products and services. You may turn off the telemetry as described in the repository. There are also some features in the software that may enable you and Microsoft to collect data from users of your applications. If you use these features, you must comply with applicable law, including providing appropriate notices to users of your applications together with a copy of Microsoft’s privacy statement. Our privacy statement is located at <https://go.microsoft.com/fwlink/?LinkID=824704>. You can learn more about data collection and use in the help documentation and our privacy statement. Your use of the software operates as your consent to these practices.
<!-- END_TF_DOCS -->