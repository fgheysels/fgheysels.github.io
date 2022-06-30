---
layout: post
title: Execute SQL commands in Azure DevOps pipelines using Azure AD authentication
comments: true
---

In this [blogpost](2022-03-30-managed-identity-users-in-sql-via-devops.md), I mentionned that it was not possible to execute a `SqlAzureDacpacDeployment@1` task.  As mentionned [here](https://docs.microsoft.com/en-us/azure/devops/pipelines/tasks/deploy/sql-azure-dacpac-deployment?view=azure-devops), using Azure AD authentication is not supported on hosted agents, you need to create a private DevOps agent.
But, it is possible to execute SQL commands against an Azure SQL Server using Azure AD authentication, albeit not via DacPac.

It is actually pretty simple:  instead of using the `SqlAzureDacpacDeployment@1` task, we simply make use of the `Invoke-SqlCmd` Powershell CmdLet.
As it turns out, the [`Invoke-SqlCmd` command](https://docs.microsoft.com/en-us/powershell/module/sqlserver/invoke-sqlcmd?view=sqlserver-ps) has a parameter that allows you to specify an AccessToken.

So, if the Service Principal that is used to execute the tasks in your pipeline has access to the SQL database, all you need to do is retrieve an appropriate access-token and pass it to the `Invoke-SqlCmd` command.

This is how it's done:

```yaml
- task: AzurePowerShell@5
  displayName: 'Execute database commands'
  inputs:
    azureSubscription: ${{parameters.azureResourceManagerConnection}}
    scriptType: inlineScript
    azurePowerShellVersion: 'LatestVersion'
    Inline: |
      Import-Module Az.Accounts -MinimumVersion 2.2.0
      Install-Module SqlServer

      $context = Get-AzContext    
      $tokenInfo = Get-AzAccessToken -ResourceUrl https://database.windows.net -DefaultProfile $context
      
      $token = $tokenInfo.Token
      
      Invoke-SqlCmd -ServerInstance "tcp:$(AzureSqlDatabase.Name)-ondemand.sql.azuresynapse.net,1433" -Database $databaseName -AccessToken $token -InputFile "$(Pipeline.Workspace)/somefile.sql"
```

An additional advantage of this approach, is that unlike `SqlAzureDacpacDeployment@1`, you can also execute this on Linux agents.

Hope you find this useful!

Frederik