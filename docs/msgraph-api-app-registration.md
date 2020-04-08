# Registering an App for Microsoft Graph Reporting API using Terraform


## Authenticating to Azure AD

First, we need to authenticate to AAD so Terraform can do its magic.  There's a lot more on the different ways to auth the AzureAD provider, but I prefer the Azure CLI method when working interactively.  Read more here ???.

```powershell
az login
```
A browser session will prompt you to authenticate, after which you will have a valid token for Terraform (and anything else that needs it) in your session.

If you only have Office 365, there's no Azure subscription available to you, and `az login` will bark back.  Just use the following:
```powershell
az login --allow-no-subscriptions
```
`allow-nosubscriptions` tells Azure CLI to ease up on the subscription requirements!

Next, let's set up a simple Terraform HCL "code" to define the application.  First, let's tell Terraform how to configure itself in the Terraform block:
```terraform
terraform {
  required_version = "~> 0.12.24"
}
```
Change the version to the verion available.  The Terraform version is not that important as long as it's > 0.12.

We're going to use the AzureAD provider, so let's set that up in its block:
```terraform
provider "azuread" {
  version   = "~>0.8"
  tenant_id = "3461c0eb-67e9-4b9e-8203-c86c80123456"
}
```
For providers, it's always good to pin the version, otherwise you may get unexpected results later.  The `tenant_id` is the *Directory ID* in which you want to register the application.  Note that, despite the name, this is not your AAD tenant ID, although the value may be the same in single-tenant environments.  In plain-jane O365 single-tenant environments, the *values* will match, but as a rule, never assume *tenant ID* = *directory ID*.

Next, let's build the resource block for the application registration itself.  There's a lot going on here:
```terraform
resource "azuread_application" "AADReportingAPIApp" {
  name                       = "AADReportingAPIApp"
  available_to_other_tenants = false
  identifier_uris            = []
  oauth2_allow_implicit_flow = false
  public_client              = false
  type                       = "webapp/api"
  reply_urls                 = ["https://localhost"]

  required_resource_access {

    resource_app_id = "00000002-0000-0000-c000-000000000000"
    resource_access {
      id   = "5778995a-e1bf-45b8-affa-663a9f3f4d04"
      type = "Role"
    }

  }

  required_resource_access {
    resource_app_id = "00000003-0000-0000-c000-000000000000"
    resource_access {
      id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"
      type = "Scope"
    }

  }
}
```
The usual application registration attributes are in play here as documented in the [azuread_application docs](https://www.terraform.io/docs/providers/azuread/r/application.html), however what about those `required_resource_access` blocks?

- `id   = "5778995a-e1bf-45b8-affa-663a9f3f4d04"` is for AAD Graph, Directory.Read.All *application* permissions.  Naturally, this will allow reading of AAD directory data.
- `id   = "e1fe6dd8-ba31-4d61-89e7-88639da4683d"` is for the Microsoft Graph, User.Read *delegated* permissions.  This will allow reading all audit log data.  This is typically automatically added to application registrations but still required in your configuration for proper managment by Terraform.

Both of these are required for accessing AAD/Microsoft Graph Reporting API.

What else do we need?

We'll need the app registration's Domain Name, Client or Application ID, and Client Secret.

Getting the Domain Name(s) is super easy; you just read the "data" using the `data` resource, then output them:
```terraform
data "azuread_domains" "aad_domains" {}

output "domains" {
  value = "${data.azuread_domains.aad_domains.domains}"
}
```

For the Client or Application ID, just output the registration `application_id` attribute:
```terraform
output "appid" {
  value = "${azuread_application.AADReportingAPIApp.application_id}"
}
```

For the Client Secret, we need to create that.  Registering an app doesn't automatically create this (it's not always needed, depending on the application type).  Here's the HCL for the Client Secret:
```terraform
resource "azuread_application_password" "AADReportingAPIApp_Secret" {
  application_object_id = azuread_application.AADReportingAPIApp.id
  value                 = "VT=uasdfSgbTanZhyz@%asdf+Tfay_MRV#"
  end_date              = "2030-01-01T00:00:00Z"
}
```
Then output your secret for safekeeping:
```terraform
output "clientsecret" {
  value = "${azuread_application_password.AADReportingAPIApp_Secret.value}"
}
```
Remember to stash these in your favorite secrets vault (like Vault or Azure Key Vault, etc.)