---
title: "Recovering Deleted Emails in Microsoft 365: A PowerShell Approach"
date: 2024-09-10
medium_url: https://medium.com/@rabbit_knight/recovering-deleted-emails-in-microsoft-365-a-powershell-approach-88976f0d3daf
---

> **Original Published Article on Medium:** [Recovering Deleted Emails in Microsoft 365: A PowerShell Approach](https://medium.com/@rabbit_knight/recovering-deleted-emails-in-microsoft-365-a-powershell-approach-88976f0d3daf)

### Recovering Deleted Emails in Microsoft 365: A PowerShell Approach

![Article Figure](https://cdn-images-1.medium.com/max/800/1*CSI7EW4cxocUvJll6o2LNg.jpeg)

As an IT enthusiast and Microsoft 365 administrator, I’ve encountered my fair share of challenges. One recurring issue is dealing with deleted emails. Recently, I faced this problem when attempting to recover deleted items from the Exchange Admin Center. However, the catch was that my tenant is using a more cost-effective Microsoft 365 plan that doesn’t offer advanced recovery options through the Exchange Admin Center. So, I decided to explore command-line tools as a solution. Here’s a step-by-step guide to solving this issue with PowerShell.

Step 1: Install the Exchange Online PowerShell Module

The first step in managing Exchange Online via PowerShell is to install the necessary module. Run this command in a PowerShell window (as Administrator) to install the module:

```
Install-Module ExchangeOnlineManagement
```

Step 2: Load the Exchange Online PowerShell Module Once installed, load the module into your session with:

```
Import-Module ExchangeOnlineManagement
```

Step 3: Connect and Authenticate to Exchange Online Next, you’ll need to establish a connection with Exchange Online. Run the following command:

```
Connect-ExchangeOnline -UserPrincipalName user@userdomain.onmicrosoft.com
```

Replace user@userdomain.onmicrosoft.com with your actual admin account’s email address. This command will prompt your credentials to authenticate securely.

Note : Be mindful of PowerShell’s execution policies. These are security measures that control how PowerShell scripts are executed on your machine. You may need to adjust your execution policy.

Authentication Issues: Navigating Version Compatibility During authentication, I encountered a roadblock. PowerShell prompted the admin to log in via a pop-up, but the login attempt failed. After some digging, I realized my PowerShell version (5) wasn’t compatible. To check your version, use:

```
$
PSVersionTable
```

The solution was to upgrade to PowerShell 7. You can install it by running:

```
winget search Microsoft.PowerShell
winget install --id Microsoft.PowerShell
```

Important : When you install PowerShell 7, it coexists alongside Windows PowerShell 5.1. This means that both versions remain available on your system, and they don’t replace each other. Here’s why you might still see PowerShell 5.1 even after installing PowerShell 7. So don’t forget to use the correct version. PowerShell 7 is installed in a separate directory and doesn’t overwrite Windows PowerShell 5.1. They have different executable names: powershell.exe for Windows PowerShell and pwsh.exe for PowerShell 7.

Step 4: Connect to Exchange Online Once you’ve run the Connect-ExchangeOnline command in PowerShell 7, you're connected to Exchange Online.

Step 5: Assigning Permissions for Email Recovery

When dealing with recoverable items in Exchange Online, there are a few things to consider: Permissions and Role Assignment. Specifically, you should have the “Mailbox Import Export” role or be a member of a Role Group that contains this role (such as “Discovery Management”). To find out which role you need to be assigned for a given cmdlet ( we are going to use Get-RecoverableItems ), you can run the following command:

```
Get-ManagementRole -Cmdlet Get-RecoverableItems
```

This will show you the role(s) required to run the cmdlet successfully. In the Exchange Admin Center, go to Roles > Admin Roles > Discovery Management and assign the necessary permissions to your account. Wait for Permissions to Take Effect. Sometimes, after assigning a new role, it takes a little time for the settings to take effect.

![Article Figure](https://cdn-images-1.medium.com/max/800/1*hiqVWbp_-R1qZeJ-G3F6TA.png)

Step 6: Finding and Restoring Deleted Items

Before diving into commands, take a moment to relaunch the Exchange Online Management module. Trust me, it’s worth it. If you start firing off commands right away, you might encounter errors.

Restoring deleted emails from the Get-RecoverableItems output involves a two-step process: first, identifying the deleted items using Get-RecoverableItems , and then restoring them using Restore-RecoverableItems

To identify deleted items, use:

```
Get-RecoverableItems -Identity user@example.com -FilterItemType IPM.Note -SubjectContains "home"
```

This will retrieve deleted email messages containing “home” in the subject for the specified mailbox.

Once you’ve identified the items, use the Restore-RecoverableItems cmdlet to restore them. Specify the mailbox identity and any additional filters (if needed):

```
Restore-RecoverableItems -Identity user@example.com -FilterItemType IPM.Note -SubjectContains "home"
```

This restores the messages that match the criteria back to their original location.

#### Conclusion: Problem Solved — Emails Recovered

After spending some time exploring various solutions, I was able to successfully recover the deleted emails using PowerShell. While Microsoft didn’t make the process easy, the challenge was rewarding
