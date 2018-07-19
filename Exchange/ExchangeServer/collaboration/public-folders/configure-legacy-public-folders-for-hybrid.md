---
title: "Configure legacy on-premises public folders for a hybrid deployment"
ms.author: dmaguire
author: msdmaguire
manager: serdars
ms.date: 6/19/2018
ms.audience: ITPro
ms.topic: article
ms.prod: exchange-server-it-pro
localization_priority: Normal
ms.collection:
- Ent_O365_Hybrid
- Strat_EX_EXOBlocker
ms.assetid: bcb0ac98-2949-486b-a8ab-8549c021651b
description: "Summary: Use the steps in this article to synchronize public folders between Exchange Online and your on-premises Exchange deployment."
---

# Configure legacy on-premises public folders for a hybrid deployment

 **Summary**: Use the steps in this article to synchronize public folders between Exchange Online and your on-premises Exchange deployment.
  
In a hybrid deployment, your users can be in Exchange Online, Exchange on-premises, or both, and your public folders are either in Exchange Online or Exchange on-premises. Public folders can only reside in one place, so you must decide whether your public folders will be in Exchange Online or on-premises. They can't be in both locations. Public folder mailboxes are synchronized to Exchange Online by the Directory Synchronization service. However, mail-enabled public folders aren't synchronized across premises.
  
This article describes how to synchronize mail-enabled public folders when your users are in Office 365 and your Exchange 2010 SP3 or later version public folders are on-premises. However, an Office 365 user who is not represented by a MailUser object on-premises (local to the target public folder hierarchy) won't be able to access legacy or Exchange 2016 on-premises public folders.
  
> [!NOTE]
> This topic refers to the Exchange 2010 SP3 or later servers as the *legacy Exchange server*.
  
You will use the following scripts to sync your mail-enabled public folders. The scripts are initiated by a Windows task that runs in the on-premises environment:
  
- `Sync-MailPublicFolders.ps1`: This script synchronizes mail-enabled public folder objects from your local Exchange on-premises deployment with Office 365. It uses the local Exchange on-premises deployment as master to determine what changes need to be applied to O365. The script will create, update, or delete mail-enabled public folder objects on O365 Active Directory based on what exists in the local on-premises Exchange deployment.
    
- `SyncMailPublicFolders.strings.psd1`: This is a support file used by the preceding synchronization script and should be copied to the same location as the preceding script.
    
When you complete this procedure your on-premises and Office 365 users will be able to access the same on-premises public folder infrastructure.
  
## What hybrid versions of Exchange will work with public folders?

The following table describes the supported version and location combinations of user mailboxes and public folders. "Hybrid not applicable" is still a supported scenario, but is not considered a hybrid scenario because both the public folders and the users are residing in the same location.
  
|**Scenario**|**On-Premises Exchange 2010 User Mailbox**|**On-Premises Exchange 2016 User Mailbox**|**Exchange Online User Mailbox**|
|:-----|:-----|:-----|:-----|
|On-Premises Exchange 2010 Public Folders  <br/> |Hybrid not applicable  <br/> |Hybrid not applicable  <br/> |Supported  <br/> |
|On-Premises Exchange 2013 or Exchange 2016 Public Folders  <br/> |Hybrid not applicable  <br/> |Hybrid not applicable  <br/> |Supported  <br/> |
|Exchange Online Public Folders  <br/> |Not supported  <br/> |Supported  <br/> |Hybrid not applicable  <br/> |
   
A hybrid configuration with Exchange 2003 public folders is not supported. If you're running Exchange 2003 in your organization, you must move all public folder databases and replicas to Exchange 2010 SP3 or later. No public folder replicas can remain on Exchange 2003.
  
## Step 1: What do you need to know before you begin?

1. These instructions assume that you have used the Hybrid Configuration Wizard to configure and synchronize your on-premises and Exchange Online environments and that the DNS records used for most users' Autodiscover references an on-premises end-point. For more information, see [Hybrid Configuration Wizard](http://technet.microsoft.com/library/2e6ed294-ee74-4038-8b71-b61786372ba4.aspx).
    
2. These instructions assume that Outlook Anywhere is enabled and functional on the on-premises legacy Exchange servers. For information on how to enable Outlook Anywhere, see [Outlook Anywhere](http://technet.microsoft.com/library/9026d461-ec6a-4ef5-ba9d-de33030858f3.aspx).
    
3. Implementing legacy public folder coexistence for a hybrid deployment of Exchange with Office 365 may require you to fix conflicts during the import procedure. Conflicts can happen due to non-routable email address assigned to mail enabled public folders, conflicts with other users and groups in Office 365, and other attributes.
    
4. These instructions assume your Exchange Online organization has been upgraded to a version that supports public folders.
    
5. In Exchange Online, you must be a member of the Organization Management role group. This role group is different from the permissions assigned to you when you subscribe to Exchange Online. For details about how to enable the Organization Management role group, see [Manage role groups](../../permissions/role-groups.md).
    
6. In Exchange 2010, you must be a member of the Organization Management or Server Management Role Based Access Control (RBAC) role groups. For details, see [Add Members to a Role Group](https://go.microsoft.com/fwlink/p/?linkId=299212).
    
7. In order to access public folders cross-premises, users must upgrade their Outlook clients to the November 2012 or later Outlook public update.
    
1. To download the November 2012 Outlook update for Outlook 2010, see [Update for Microsoft Outlook 2010 (KB2687623) 32-Bit Edition](https://www.microsoft.com/download/details.aspx?id=35702).
    
2. To download the November 2012 Outlook update for Outlook 2007, see [Update for Microsoft Office Outlook 2007 (KB2687404)](https://www.microsoft.com/download/details.aspx?id=35718).
    
8. Outlook 2016 for Mac (and earlier versions) and Outlook for Mac for Office 365 are not supported for cross-premises public folders. Users must be in the same location as the public folders to access them with Outlook for Mac or Outlook for Mac for Office 365. In addition, users whose mailboxes are in Exchange Online won't be able to access on-premises public folders using Outlook on the web.
    
9. After you have followed the instructions in this article to configure your on-premises public folders for a hybrid deployment, users who are external to your organization won't be able to send messages to your on-premises public folders unless you take additional steps. You can either set the accepted domain for the public folders to Internal Relay (see [Manage accepted domains in Exchange Online](https://technet.microsoft.com/library/jj945194%28v=exchg.150%29.aspx) for more information) or you can disable Directory Based Edge Blocking (DBEB), as described in [Use Directory Based Edge Blocking to Reject Messages Sent to Invalid Recipients](https://technet.microsoft.com/library/dn600322%28v=exchg.150%29.aspx).
    
## Step 2: Make remote public folders discoverable
<a name="Discoverable"> </a>

1. If your public folders are on Exchange 2010 servers, you must install Client Access services on all mailbox servers that have a public folder database. This enables the Exchange RpcClientAccess service to run, which enables all clients to access public folders. For more information, see [Install Exchange Server 2010](https://technet.microsoft.com/library/bb124778%28v=exchg.141%29.aspx).
    
    > [!NOTE]
    > This server doesn't have to be part of the Client Access load balancing. For more information, see [Understanding Load Balancing in Exchange 2010](https://technet.microsoft.com/library/ff625247%28v=exchg.141%29.aspx).
  
2. Create an empty mailbox database on each public folder server.
    
    For Exchange 2010, run the following command in the Exchange Management Shell. This command excludes the mailbox database from the mailbox provisioning load balancer. This prevents new mailboxes from automatically being added to this database.
    
  ```
  New-MailboxDatabase -Server <PFServerName_with_CASRole> -Name <NewMDBforPFs> -IsExcludedFromProvisioning $true
  ```

    For Exchange 2007, run the following command in the Exchange Management Shell:
    
  ```
  New-MailboxDatabase -StorageGroup "<PFServerName>\StorageGroup>" -Name <NewMDBforPFs>
  ```

    > [!NOTE]
    > We recommend that the only mailbox that you add to this database is the proxy mailbox that you'll create in the next step. No other mailboxes should be created on this mailbox database.
  
3. Create a proxy mailbox within the new mailbox database, and hide the mailbox from the address book. The SMTP of this mailbox will be returned by AutoDiscover as the _DefaultPublicFolderMailbox_ SMTP, so that by resolving this SMTP the client can reach the legacy exchange server for public folder access.
    
  ```
  New-Mailbox -Name <PFMailbox1> -Database <NewMDBforPFs>
  ```

  ```
  Set-Mailbox -Identity <PFMailbox1> -HiddenFromAddressListsEnabled $true
  ```

4. For Exchange 2010, enable Autodiscover to return the proxy public folder mailboxes.
    
  ```
  Set-MailboxDatabase <NewMDBforPFs> -RPCClientAccessServer <PFServerName_with_CASRole>
  ```

5. Repeat the preceding steps for every public folder server in your organization.
    
## Step 3: Download the scripts
<a name="download"> </a>

1. Download the following files from [Mail-enabled Public Folders - directory sync script](https://www.microsoft.com/download/details.aspx?id=46381):
    
  - `Sync-MailPublicFolders.ps1`
    
  - `SyncMailPublicFolders.strings.psd1`
    
2. Save the files to the local computer on which you'll be running PowerShell. For example, C:\PFScripts.
    
## Step 4: Configure directory synchronization
<a name="dirsync"> </a>

The Directory Synchronization service doesn't synchronize mail-enabled public folders. Running the following script will synchronize the mail-enabled public folders across premises. Special permissions assigned to mail-enabled public folders will need to be recreated in the cloud since cross-premise permission are not supported in Hybrid Deployment scenarios.
  
> [!NOTE]
> Synchronized mail-enabled public folders will appear as mail contact objects for mail flow purposes and will not be viewable in the Exchange admin center. See the Get-MailPublicFolder command. To recreate the SendAs permissions in the cloud, use the Add-RecipientPermission command.
  
1. On the legacy Exchange server, run the following command to synchronize mail-enabled public folders from your local on-premises Active Directory to O365.
    
  ```
  Sync-MailPublicFolders.ps1 -Credential (Get-Credential) -CsvSummaryFile:sync_summary.csv
  ```

    Where `Credential` is your Office 365 user name and password, and `CsvSummaryFile` is the path to where you would like to log synchronization operations and errors, in .csv format.
    
> [!NOTE]
> Before running the script, we recommend that you first simulate the actions that the script would take in your environment by running it as described above with the `-WhatIf` parameter. We also recommend that you run this script daily to synchronize your mail-enabled public folders.
  
## Step 5: Configure Exchange Online users to access on-premises public folders
<a name="Access"> </a>

The final step in this procedure is to configure the Exchange Online organization and to allow access to the legacy on-premises public folders.
  
You will point to all of the proxy public folder mailboxes that you created in [Step 2: Make remote public folders discoverable](configure-legacy-public-folders-for-hybrid.md#Discoverable) to enable theExchange Online organization to access the on-premises public folders.
  
Run the following command in Exchange Online PowerShell. To learn how to use Windows PowerShell to connect to Exchange Online, see [Connect to Exchange Online PowerShell](https://docs.microsoft.com/powershell/exchange/exchange-online/connect-to-exchange-online-powershell/connect-to-exchange-online-powershell).
  
```
Set-OrganizationConfig -PublicFoldersEnabled Remote -RemotePublicFolderMailboxes PFMailbox1,PFMailbox2,PFMailbox3
```

You must wait until Active Directory synchronization has completed to see the changes. This process can take up to 3 hours to complete. If you don't want to wait for the recurring synchronizations that occur every three hours, you can force directory synchronization at any time. For detailed steps to force directory synchronization, see [Force directory synchronization](https://technet.microsoft.com/library/jj151771.aspx). Office 365 randomly selects one of the public folder mailboxes that's supplied in this command.
  
> [!IMPORTANT]
> An Office 365 user who is not represented by a MailUser object on-premises (local to the target public folder hierarchy) won't be able to access legacy or Exchange 2016 on-premises public folders. See the Knowledge Base article [Exchange Online users can't access legacy on-premises public folders](https://go.microsoft.com/fwlink/p/?LinkId=699451) for a solution.
  
## How do I know this worked?
<a name="Access"> </a>

1. Log on to Outlook for a user who is in Exchange Online and perform the following public folder tests:
    
  - View the hierarchy.
    
  - Check permissions
    
  - Create and delete public folders.
    
  - Post content to and delete content from a public folder.
    
