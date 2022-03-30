---
layout: post
title: Adding Managed Identity users to an Azure SQL database via DevOps pipelines using the DacPac task
comments: true
---

Creating users in Azure SQL based on Managed Identity or AzureAD accounts is very simple yet very powerfull.  However, there are some issues when you want to create those kind of users in your Azure SQL database via Azure DevOps.  This post explains how to work around them.

## Introduction

Accessing an Azure SQL Database via a Managed Identity has been around for quite some time now.  I am sure that we all agree that accessing an Azure SQL database via a Managed Identity is the way to go, as it enables us to avoid using username/password credentials.
Creating a database user for a managed identity is really simple:

- First of all, you need to make sure that you have an Azure Active Directory Admin set in your Azure SQL Server

- Once you've logged in with the Azure Active Directory Admin, simply execute this statement and you're done:

  ```sql
  CREATE USER [<name>] FROM EXTERNAL PROVIDER
  ```

  <sub>Where `<name>` can be a user-name or a resource name (For instance an App Service).</sub>

## Automating creating Managed Identity SQL users

When creating Azure solutions, we love to automate deployments.  This means that we might also have to automate the provisioning of SQL users in our pipeline.
Given the statements that we've mentioned in the previous paragraph, this should be an easy task.  However, there are some obstacles:

- Active Directory users can only be created by other Active Directory users
- This means that our DacPac task that would create those users, needs to login using `aadAuthenticationIntegrated` as authentication-type.
- The `aadAuthenticationIntegrated` authentication-type is not supported on hosted agents, as mentionned [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/sql-azure-dacpac-deployment?view=azure-devops).

There is an (undocumented) way to create Azure AD users in Azure SQL using a DacPac task where you log in using a SQL Server user account that has administrator rights:

- First of all, you need to generate the `sid` of the service principal of the managed identity of the resource or user which you would like to grant access
- Using that `sid` you can then create a user for the managed identity.

It's done like this:

```yaml
- task: AzureCLI@2
  displayName: Determine SQL Server SID for managed identity of MyWebApp
  inputs:
    azureSubscription: ${{parameters.azureResourceManagerConnection}}
    scriptType: pscore
    scriptLocation: inlineScript
    failOnStandardError: false
    inlineScript: |      
      # We need to have the ApplicationId of the App Registration that represents the
      # WebApp.   The SID is calculated based on the ApplicationId
            
      $apiSp = az ad sp list --display-name $(MyWebApp.Name) | ConvertFrom-Json
      $appId = $apiSp.appId
            
      [guid]$guid = [System.Guid]::Parse($appId)

      foreach ($byte in $guid.ToByteArray()) {
        $byteGuid += [System.String]::Format("{0:X2}", $byte)
      }

      $sid = "0x" + $byteGuid

      Write-Host "##vso[task.setvariable variable=MyWebApp.Sid]$sid"
      
- task: SqlAzureDacpacDeployment@1
  displayName: 'Create login for MyWebApp'
  inputs:
    azureSubscription: ${{parameters.azureResourceManagerConnection}}    
    sqlUserName: $(SqlServer.Admin.UserName)
    sqlPassword: $(SqlServer.Admin.Password)
    ServerName: '$(SqlServer.Name).database.windows.net'
    DatabaseName: $(SqlDatabase.Name)
    deployType: 'inlineSqlTask'
    sqlInline: |   
      IF NOT EXISTS (SELECT * FROM sys.database_principals WHERE name = '$(MyWebApp.Name)')
      BEGIN      
        CREATE USER [$(MyWebApp.Name)] WITH sid = $(MyWebApp.Sid), TYPE = E

        ALTER ROLE db_datareader ADD MEMBER [$(MyWebApp.Name)];
        ALTER ROLE db_datawriter ADD MEMBER [$(MyWebApp.Name)];
      END
              
    IpDetectionMethod: 'AutoDetect'
```

With the code snippet above, we first determine the `sid` of the Service Principal that represents the identity of an Azure Web App.  As you can see, the `sid` is calculated based on the `ApplicationId` of the Service Principal.

Once that is done, we simply use the `CREATE USER` statement to create the user for the specified `sid`.
After that, the user can be added to the required database roles.

Hope this helps,

Frederik