# Title

* **Status**: Pending
* **Author**: @lakshmimsft

## Overview

As part of Terraform support in Radius, we currently support azurerm, kubernetes and aws providers when creating recipes. We pull credentials stored in UCP and setup and save provider configuration which is accessible to recipes.\
This document  describes a proposal to support multiple providers, setup their provider configuration, which would be accessible to all recipes within an  environment.

## Terms and definitions

| Term     | Definition                                                                                                                                                                                                 |
| -------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Terraform Provider | A Terraform provider is a plugin that allows Terraform to interact with and manage resources of a specific infrastructure platform or service, such as AWS, Azure, or Google Cloud. |


## Objectives

**Reference for new type recipeConfig and handling of secrets:** [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)

> **Issue Reference:** <!-- (If appropriate) Reference an existing issue that describes the feature or bug. -->
https://github.com/radius-project/radius/issues/6539

### Goals

1. Enable users to use terraform recipes with multiple providers (including and outside of Azure, Kubernetes, AWS).
2. Enable users to use terraform recipes with multiple configurations for the same provider using *alias*. 
3. Enable provider configurations to be available for all recipies used in an environment.


### Non goals
1. Updates to Bicep Provider configuration. The focus of this effort is targeted for Terraform provider support. Bicep provider support will be addressed as a separate initiative as warranted.
2. Authentication for providers hosted in private registries. This is out of scope for the current design effort and will be addressed in the future as required.
3. Support for .terraformrc files. These are CLI configuration files and not in scope of current effort to support multiple provider configuration to be used by recipes. ref: [link](https://developer.hashicorp.com/terraform/cli/config/config-file)

### User scenarios (optional)

#### User story 1
As an operator, I maintain a set of Terraform recipes for use with Radius. I have a set of provider configurations which are applicable across multiple recipes and I would like to configure them in a single centralized location for ease of maintenance.


#### User story 2
As an operator, I would like to manage cloud resources in different geographical regions. To enable this, I need to set up multiple configurations for the same provider for different regions and use an alias to refer to individual provider configurations. 

## Design
### Design details

Reviewing popular provider configurations (GCP, Oracle, Heroku, DigitalOcean, Docker etc.) a large number of provider configurations can be setup by handling a combination of key value pairs, key value pairs in nested objects, secrets and environment variables.

### Key-value pairs
Key-value pairs will be parsed and saved to a Terraform configuration file.

### Secrets
Secrets will be handled similarly to the approach described in document [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md) wherein Applications.Core/secretStores can point to an existing K8s secret.

The system will call the ListSecrets() api in Applications.Core namespace, retrieve contents of the secret and build the Terraform provider configuration.

### Environment variables
In a significant number of providers, as per documentation, environment variables are used one of the methods of saving sensitive credential data along with insensitive data. We allow the users to set environment variables for provider configuration. For sensitive information, we recommend the users save these values as secrets and point to them inside the env block.

```
...
 recipeConfig: {
    terraform: {
      ...
     providers: [...]
     env: {
        'MY_ENV_VAR_1': 'my_value'
        secrets: {                       // Individual Secrets from SecretStore
        'MY_ENV_VAR_2': {
            source: secretStoreConfig.id
            key: 'envsecret.one'
          }
        'MY_ENV_VAR_3': {
            source: secretStoreConfig.id
            key: 'envsecret.two'
          }
        }
      }
```

Environment variables apply to all providers configured in the environment. The system cannot set two separate values for the same environment variable for multiple provider configurations. In such cases, per provider documentation, there may be alternatives that users can avail (eg. For GCP, users can set  *credentials* field inside each instance of provider config as opposed to using env variable GOOGLE_APPLICATION_CREDENTIALS).


The system will allow for ability to set up multiple configurations for the same provider using the keyword *alias*. Validation for configuration 'correctness' will be handled by Terraform with calls to terraform init and terraform apply.

```
...
 recipeConfig: {
    terraform: {
      ...
      providers: [
      {
        name: 'azurerm',
        properties: {
          subscriptionid: 1234,
          secrets: {                  // Individual Secrets from SecretStore
            'my_secret_1': {
              source: secretStoreAz.id
              key: 'secret.one'
            }
            'my_secret_2': {
               source: secretStoreAzPayment.id
              key: 'secret.two'
            }
          }
        }
      },
      {
        name: 'azurerm',
        properties: {
          subscriptionid: 1234,
          alias: 'az-paymentservice'
       }
     }]
...       
```
Configuration for providers as described in this document will take precedence over provider credentials stored in UCP (currently these include azurerm, aws, kubernetes providers). So, for eg. In the scenario where credentials for Azure are saved with UCP during Radius install and a Terraform recipe created by an operator declares 'azurerm' under the the *required_providers* block; If there exists a provider configuration under the *providers* block under *recipeConfig*, these would take precedence and used to build the Terraform configuration file instead of Azure credentials stored in UCP. 


### Example Bicep Input :
``` diff
resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dsrp-resources-env-recipes-context-env'
  location: 'global'
  properties: {
    compute: {
      ...
    }
    providers: {
      ...
    }
    recipeConfig: {
      terraform: {
        authentication:{
          ...        
        }
+       providers: [
+         {
+            name: 'azurerm',
+            properties: {          
+             subscriptionid: 1234,
+             secrets: {                  // Individual Secrets from SecretStore
+               'my_secret_1': {
+                  source: secretStoreAz.id
+                  key: 'secret.one'
+                }
+               'my_secret_2': {
+                  source: secretStoreAzPayment.id
+                  key: 'secret.two'
+               }
+             }
+            }
+          },
+          {
+            name: 'azurerm',
+            properties: {
+              subscriptionid: 1234,
+              tenant_id: '745fg88bf-86f1-41af-'
+              alias: 'az-paymentservice',
+            }
+          },
+          {
+            name: 'gcp',
+            properties: {
+              project: 1234,
+              regions: ['us-east1', 'us-west1']
+            }
+          },
+          {
+            name: 'oraclepass',
+            properties: {
+              database_endpoint: "...",
+              java_endpoint: "...",
+              mysql_endpoint: "..."
+            }
+          }
+        ]
+      }
+      env: {
+        'MY_ENV_VAR_1': 'my_value'
+        secrets: {                       // Individual Secrets from SecretStore
+        'MY_ENV_VAR_2': {
+            source: secretStoreConfig.id
+            key: 'envsecret.one'
+          }
+        'MY_ENV_VAR_3': {
+            source: secretStoreConfig.id
+            key: 'envsecret.two'
+          }
+        }
+      }      
+    }
  }
  recipes: {      
    ...
  }
}

Option 2:
This option provides grouping and efficiency to retrieve all details for a single provider. 

resource env 'Applications.Core/environments@2023-10-01-preview' = {
  name: 'dsrp-resources-env-recipes-context-env'
  location: 'global'
  properties: {
    compute: {
      ...
    }
    providers: {
      ...
    }
    recipeConfig: {
      terraform: {
        authentication:{
          ...        
        }
+       providers: {
+         'azurerm': [
+           {
+             subscriptionid: 1234,
+             secrets: {                  // Individual Secrets from SecretStore
+               'my_secret_1': {
+                  source: secretStoreAz.id
+                  key: 'secret.one'
+                }
+               'my_secret_2': {
+                  source: secretStoreAzPayment.id
+                  key: 'secret.two'
+               }
+             }
+          },
+          {
+             subscriptionid: 1234,
+             tenant_id: '745fg88bf-86f1-41af-'
+             alias: 'az-paymentservice', 
+          }]
+          'gcp': [
+            {
+              project: 1234,
+              regions: ['us-east1', 'us-west1']
+            }
+          ]
+          'oraclepass': [
+            {
+              database_endpoint: "...",
+              java_endpoint: "...",
+              mysql_endpoint: "..."
+            }
+          ]
+        }
+     }
+     env: {
+        'MY_ENV_VAR_1': 'my_value'
+        secrets: {                       // Individual Secrets from SecretStore
+        'MY_ENV_VAR_2': {
+            source: secretStoreConfig.id
+            key: 'envsecret.one'
+          }
+        'MY_ENV_VAR_3': {
+            source: secretStoreConfig.id
+            key: 'envsecret.two'
+          }
+        }
+      }   
+    }
  }
  recipes: {      
    ...
  }
}


```


Limitations: Customers may store sensitive data in other formats which may not be supported. eg. sensitive data is stored on files, which customers will not currently be able to load on disk in applications-rp where tf init/apply commands are run.
Containerization work may alleviate this limitations. Further design work will be needed for towards this which is planned for the near future. 

### API design

***Model changes providers***

### Option 1:
```
Addition of new property to TerraformConfigProperties in `recipeConfig` under environment properties.

model TerraformConfigProperties{
  @doc(Specifies authentication information needed to use private terraform module repositories.)  
  authentication?: AuthConfig
+ providers?: Array<ProviderConfig>
}

@doc("ProviderConfig specifies provider configurations needed for recipes")
model ProviderConfig {
 name: string
 properties: ProviderConfigProperties
}

@doc("ProviderConfigProperties specifies provider configuration details needed for recipes")
model ProviderConfigProperties extends Record<unknown> {
  @doc("The secrets for referenced resource")
  secrets?: Record<ProviderSecret>;
}
```
### Option 2:

```
Addition of new property to TerraformConfigProperties in `recipeConfig` under environment properties.

model TerraformConfigProperties{
  @doc(Specifies authentication information needed to use private terraform module repositories.)  
  authentication?: AuthConfig
  providers?: Record<Array<ProviderConfigProperties>>
}

@doc("ProviderConfigProperties specifies provider configuration details needed for recipes")
model ProviderConfigProperties extends Record<unknown> {
  @doc("The secrets for referenced resource")
  secrets?: Record<ProviderSecret>;
}
```
***Model changes env***
```
Addition of new property to RecipeConfigProperties under environment properties.

@doc("Specifies recipe configurations needed for the recipes.")
model RecipeConfigProperties {
  @doc("Specifies the terraform config properties")
  terraform?: TerraformConfigProperties;
+ env?: EnvironmentVariables
}

@doc("EnvironmentVariables describes structure enabling environment variables to be set")
model EnvironmentVariables extends Record<string>{
  secrets?: Record<ProviderSecret>
}

@doc("Specifies the secret details")
model ProviderSecret {
  @doc("The resource id for the secret store containing credentials")
  secretStore: string;
  key: string;
}

```

## Decision on Options 1 versus 2 above:
We have decided to go ahead with Option 2 and discussed adding validation to check number of provider configurations to be a minimum of 1. Option 2 helps users keep track of all provider configurations for a provider in one place and lowers probability of, say, duplication of provider configurations if it is laid out in one list as in Option 1. Also, we can enforce some constraints on, say, minimum number of configurations for a provider. Option 2 is optimized for multiple provider configurations per provider and that may not apply for every provider configuration that users set up.


## Alternatives considered

Mentioned under Limitations above, work to containerize running terraform jobs was considered as a precursor to this work. The time sensitivity for unblocking customers on ability to configure and use providers they use today was given a priority and current design held. Containerization will be taken up as a parallel effort in the near future.

## Test plan (TBD)

#### Unit Tests (TBD)
-   Update environment conversion unit tests to validate providers, env type under recipeConfig.
-   Update environment controller unit tests for providers, env.

#### Functional tests
- Add e2e test to verify recipe using multiple provider configuration is deployed.
- (TBD) Discuss/List providers we want to test with.

## Security

Largely following security considerations and secret handling described in design for private terraform modules: [Design document for Private Terraform Repository](https://github.com/radius-project/design-notes/blob/3644b754152edc97e641f537b20cf3d87a386c43/recipe/2024-01-support-private-terraform-repository.md)
The work done here will be to read, decipher values and secrets and environment variables set by user in the *providers* block, to build internal terraform configuration file used by tf setenv, init and apply commands. 

## Monitoring

## Development plan
We're breaking down the effort into smaller units in order to enable parallel work and faster delivery of this feature within a shorteneed sprint.
The user stories created are as follows:
The numbers indicate sequential order of work that can be done. Having said that, work for numbers 1 and 2 are going ahead in parallel and we will resolve conflicts as PRs get merged.
1. Update Provider DataModel, TypeSpec, Convertor, json examples
1. Update Environment Variables DataModel, TypeSpec, Convertor, json examples
1. Documentation
2. Build Provider Config (minus secrets)
2. Process, update environment variables - minus secrets
3. Functional Tests
4. Update Secret DataModel, TypeSpec, Convertor, json examples
4. Secret processing - Providers + Environment Variables



## Open issues/questions
Do we consider first class support for GCP and other popular providers or continue with a generic approach??
Answer-We should start with a generic approach to unlock everything, and then add first-class support where users request it later on. That we support everything, and then we can add convenience later on.