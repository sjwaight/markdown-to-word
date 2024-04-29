# Fix Logic App Connections Managed Identity errors in Bicep templates

Off the back of my [recent post on building an Azure Logic Apps solution](2022-12-20-cross-posting-blog-posts-to-mastodon-twitter-and-linkedin-using-azure-logic-apps.md) for cross-posting on social media I wanted to export what I'd quickly built and make an easily deployable asset in Azure using [Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/overview). Right now there is no way to export deployed artefacts from Azure directly to Bicep - you must first export to an Azure Resource Manager (ARM) template and then use the Bicep tooling to convert.

The majority of the time the result is a Bicep template that requires a little tidy up before it can be used, however, once I'd tidied up my solution, I went to test the Bicep template and immediately hit upon this error:

> "The API connection 'keyvault' is not configured to support managed identity."

Replace **keyvault** with whatever the name of the Logic App API Connection is you are using in Azure.

### Create a User Assigned Managed Identity

First off, I thought this was due to my use of a System-Assigned Managed Identity which is created and managed transparently by an Azure resource such as a Logic App. The beauty of this type of Managed Identity is that if you delete the resource it was created by, the Managed Identity is also deleted.

To fix this I thought I'd add a User-Assigned Managed Identity to the Bicep file, so I went ahead and added the below.

```bicep:managed-identity.bicep
resource managed_service_identity_kv_resource 'Microsoft.ManagedIdentity/userAssignedIdentities@2018-11-30' = {
  name: managed_identity_name
  location: resource_group_location
}
```

Now I have a Managed Identity that I can reference elsewhere in the Bicep file without creating a circular dependency.

### Allow Managed Identity to access Key Vault

Next up I can reference this Managed Identity when I create my Key Vault instance. See line 13 in the below GitHub Gist.

```bicep:key-vault.bicep
resource key_vault_resource 'Microsoft.KeyVault/vaults@2022-07-01' = {
  name: key_vault_name
  location: resource_group_location
  properties: {
    sku: {
      family: 'A'
      name: 'standard'
    }
    tenantId: subscription().tenantId
    accessPolicies: [
      {
        tenantId: subscription().tenantId
        objectId: managed_service_identity_kv_resource.properties.principalId
        permissions: {
          certificates: []
          keys: []
          secrets: [
            'get'
          ]
        }
      }
    ]
    enabledForDeployment: false
    enabledForDiskEncryption: false
    enabledForTemplateDeployment: false
    enableSoftDelete: true
    softDeleteRetentionInDays: 7
    enableRbacAuthorization: false
    publicNetworkAccess: 'Enabled'
  }
}
```

### Logic App Parameters

Our connections will be listed in the Parameters section of our Logic App in the Bicep template, and we should find that the authentication type is setup to **ManagedServiceIdentity** as per line 9 below.

```bicep;logic-app-parameters.bicep
parameters: {
      '$connections': {
        value: {
          keyvault: {
            connectionId: connections_keyvault_resource.id
            connectionName: 'keyvault'
            connectionProperties: {
              authentication: {
                type: 'ManagedServiceIdentity'
              }
            }
            id: subscriptionResourceId('Microsoft.Web/locations/managedApis', resource_group_location, 'keyvault')
          }
        }
      }
 }
```

Sadly, none of this solved my issue.

### Connection definition

This is where the magic happens. There is currently a gap in ARM and Bicep's ability to export / define the values required to correctly map a Connection to a Managed Identity (there's an [open bug on GitHub](https://github.com/Azure/bicep/issues/5056) on it). Here's our Connection definition in the Bicep file.

```bicep:connection-logic-app-kv-msi.bicep
resource connections_keyvault_resource 'Microsoft.Web/connections@2016-06-01' = {
  name: connections_keyvault_name
  location: resource_group_location
  properties: {
    displayName: 'KeyVaultMIAccess'
    parameterValueType: 'Alternative'
    alternativeParameterValues: {
      vaultName: key_vault_name
    }
    customParameterValues: {}
    api: {
      name: 'keyvault'
      displayName: 'Azure Key Vault'
      description: 'Azure Key Vault is a service to securely store and access secrets.'
      iconUri: 'https://connectoricons-prod.azureedge.net/releases/v1.0.1597/1.0.1597.3005/keyvault/icon.png'
      brandColor: '#0079d6'
      id: subscriptionResourceId('Microsoft.Web/locations/managedApis', resource_group_location, 'keyvault')
      type: 'Microsoft.Web/locations/managedApis'
    }
  }
}
```

The key lines that solve our issue are as follows. This entry is required to map the Connection to the Key Vault.

```json
parameterValueType: 'Alternative' alternativeParameterValues: { vaultName: key_vault_name }
```

So, there we go. We can now you can deploy Azure Logic Apps that use a Managed Service Identity to connect to an Azure Key Vault!

I'm chalking this post up as "would have saved me a bunch of time" because it took a look of looking around to get to the ultimate answer. I hope this one saves you some time! Find the [full Bicep file on GitHub](https://github.com/sjwaight/NoCodeSocialCrossPosting).

😎
