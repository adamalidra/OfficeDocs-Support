---
title: Upgrade SharePoint through Workflow Manager
description: Describes how to upgrade SharePoint 2013 to SharePoint 2016 without creating a new Workflow Manager, and provides resolutions to issues after upgrade.
author: helenclu
ms.author: luche
manager: dcscontentpm
audience: ITPro
ms.topic: troubleshooting
localization_priority: Normal
ms.custom: 
  - CSSTroubleshoot
ms.reviewer: 
appliesto: 
  - Microsoft SharePoint Server 2013 Service Pack 1
  - SharePoint Server 2013
  - Workflow Manager 1.0
search.appverid: MET150
ms.date: 12/17/2023
---
# Upgrade SharePoint 2013 to SharePoint 2016 through Workflow Manager

_Original KB number:_ &nbsp; 4073758

## Summary

When you upgrade Microsoft SharePoint 2013 to Microsoft SharePoint 2016, you don't have to create a new Workflow Manager installation. You can use the same installation that was used by the SharePoint 2013 farm in the new SharePoint 2016 farm.

However, you may have to create a new installation of Workflow Manager in certain circumstances. For example, if you want to move Workflow Manager to a different Windows operating system, or if the back-end database server is decommissioned. In these situations, follow the steps in [Workflow Manager Disaster Recovery](/archive/blogs/biztalknotes/workflow-manager-disaster-recovery) to create the new Workflow Manager installation by using the old databases. Make sure that you use the most recent copy of the Workflow Manager databases.

## Background

When you use SharePoint Server together with Workflow Manager, Workflow Manager keeps a record of the SharePoint sites that have published workflows. Each site is represented in Workflow Manager as a *scope*. Workflow Manager also stores the workflow definitions, all workflow instances, and their statuses.

SharePoint stores the workflow history and workflow task information for SharePoint workflows. When the workflow status page is loaded, SharePoint first makes a call to Workflow Manager to see whether the workflow exists. To do this, it uses the workflow instance ID. Then, SharePoint loads the rest of the workflow information. If the workflow instance ID is missing in Workflow Manager, or if an error occurs during communication with Workflow Manager, you receive an error message.

## Upgrade SharePoint 2013 to SharePoint 2016 through Workflow Manager

### Prerequisites

- Install the latest cumulative update for Workflow Manager by using Web Platform Installer (Web PI).
- Install the latest version of Workflow Manager Client on the SharePoint 2013 servers, and make sure that all workflows are functional.
- Install the SharePoint 2016 farm, and upgrade all service applications and content databases.
- On all SharePoint 2016 farm servers, install the latest version of Workflow Manager Client by using Web PI.

### Register Workflow Manager with SharePoint 2016

To do this, follow these steps:

1. In the SharePoint 2013 farm, go to **Central Admin** > **Application Management** > **Manage Service Application**, and then delete **Workflow Service Application Proxy**.
2. In the SharePoint 2016 farm, run the following PowerShell cmdlet to pair SharePoint 2016 together with the same Workflow Manager installation:

    ```powershell
    Register-SPWorkflowService -SPSite <SharePoint site URL> -WorkflowHostUri <Workflow service endpoint URL> -force
    ```

> [!NOTE]
> Be aware that the content databases that were upgraded in the new SharePoint 2016 farm should have been restored from the most recent backup. This helps prevent differences and gaps between Workflow Manager and the content databases.

## Common issues you may experience after the upgrade

### Issue 1: Site URL is changed

If your site URL is changed in SharePoint 2016 but the site ID remains the same, you must republish a workflow from the affected site by using SharePoint Designer.

### Issue 2: Workflows don't start on some sites

If workflows don't start on some sites, republish the workflows from the affected site. Or, run the **Refresh Trusted Security Token Services Metadata feed** timer job.

### Issue 3: Workflows fail and return "Cannot get app principal permission information" error

**Symptoms**

Consider the following scenario:

- You have SharePoint 2013 workflows and Workflow Manager configured in your farm.
- You have recently connected sites in the farm to a previously existing instance of Workflow Manager.

In this scenario, workflows that are created after you connect to the Workflow Manager installation finish successfully. However, workflows that are created before you connect to Workflow Manager don't finish. Instead, they get stuck when they try to finish or they remain in a suspended state. For workflows that remain suspended, you receive an HTTP 500 error. Additionally, the following entry is logged in the ULS log:

> Cannot get app principal permission information.

**Cause**
  
Workflow Manager already has a scope for the site on which the workflows are running. Because the scope has an incorrect `SPAuthenticationRealm` value in the `ApplicationID` field of the scope, no `SPAppPrincipal` class exists on the SPWeb object that matches the `ApplicationID` value of the scope. Therefore, the workflows fail and returns an error message.

**Resolution**

To resolve this issue, use the following PowerShell commands to register the new `SPAppPrincipal` object. You do this on the `SPWeb` object whose ID matches the `ApplicationID` value that's stored in the scope for the `SPWeb` object in Workflow Manager.

```powershell
#Variables
$webUrl = "http://sp.contoso.com/sites/teamsite/teamweb"
$oldAuthRealm = "58a2b173-0f88-4bff-935b-bf3778cd0524" #authentication realm expected by Workflow Manager
$newAuthRealm = "48834d17-d729-471e-b0d0-a0ec83b49de0" #authentication realm of current farm

#Get the SPWeb and SPSite objects, and the id of the web
$web = Get-SPWeb $webUrl
$site = $web.site
$clientId = $web.Id

#Create the old and new app principal ids
$oldAppId = "$clientId@$oldAuthRealm"
$newAppId = "$clientId@$newAuthRealm"

#Register the app principal with the old authentication realm
Register-SPAppPrincipal -DisplayName "Old Workflow" -Site $web -NameIdentifier $oldAppId

#Set permissions for the app principal
$oldAppPrincipal = Get-SPAppPrincipal -Site $web -NameIdentifier $oldAppId
Set-SPAppPrincipalPermission -Site $web -AppPrincipal $oldAppPrincipal -Scope SiteCollection -Right FullControl
Set-SPAppPrincipalPermission -Site $web -AppPrincipal $oldAppPrincipal -Scope Site -Right FullControl

#List the app principals with the old and new authentication realms in the ids
Get-SPAppPrincipal -Site $web -NameIdentifier $oldAppId | fl
Get-SPAppPrincipal -Site $web -NameIdentifier $newAppId | fl
```

**More information**

To get the `SPAuthenticationRealm` value of `ApplicationID` that's stored in the scope, follow these steps:

1. Run the following SQL query:

    ```sql
    SELECT *
    FROM [WFResourceManagementDB].[dbo].[Scopes] WITH (NOLOCK)
    WHERE Description like '%<WebID>%'
    ```

    > [!NOTE]
    > <*WebID*> is the placeholder for the ID of the `SPWeb` object.

2. In the query result, click the value in the `SecuritySettings` column to open the XML on a separate tab in SQL Server Management Studio.
3. In the XML file, locate the `ApplicationID` element that contains the value. For example, locate the following element:

    `<ApplicationId>SPWeb_object_ID@SPAuthenticationRealm</ApplicationId>`

    > [!NOTE]
    > The GUID that appears before the at sign (@) is the ID of the `SPWeb` object, and the GUID that appears after the at sign is the `SPAuthenticationRealm` value.

Alternatively, you can find the `SPAuthenticationRealm` value in ULS log, such as in the following example log entry:

> Date/Time                 w3wp.exe (SPWFE01:0x51FC)    0x1298  SharePoint Foundation  Authentication Authorization    an3eg    Medium               Cannot get app principal permission information. AppId=i:0i.t|ms.sp.ext|**\<SPWeb object ID>@\<SPAuthenticationRealm>**
>
> Date/Time                 w3wp.exe (SPWFE01:0x51FC)    0x1298  SharePoint Foundation  General 8nca                Medium               Application error when access /site/teamsite/teamweb/_vti_bin/client.svc, Error=Object reference not set to an instance of an object.   at Microsoft.SharePoint.SPAppRequestContext.EnsureTenantPermissions(SPServiceContext serviceContext, Boolean throwIfAppNotExits, Boolean allowFullReset)     at Microsoft.SharePoint.SPAppRequestContext.InitCurrent(HttpContext context)     at Microsoft.SharePoint.ApplicationRuntime.SPRequestModule.InitCurrentAppPrincipalToken(HttpContext context)     at Microsoft.SharePoint.ApplicationRuntime.SPRequestModule.PostAuthenticateRequestHandler(Object oSender, EventArgs ea)     at System.Web.HttpApplication.SyncEventExecutionStep.System.Web.HttpApplication.IExecutionStep.Execute()     at System.Web.HttpApplication.ExecuteStep(IExecutionStep step, Boolean& completedSynchronously)
