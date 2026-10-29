---
title: "The Transition: MSOnline and AzureAD to Microsoft Graph"
date: 2024-09-30
medium_url: https://medium.com/@rabbit_knight/the-transition-msonline-and-azuread-to-microsoft-graph-4c0e80a111d0
---

> **Original Published Article on Medium:** [The Transition: MSOnline and AzureAD to Microsoft Graph](https://medium.com/@rabbit_knight/the-transition-msonline-and-azuread-to-microsoft-graph-4c0e80a111d0)

### The Transition: MSOnline and AzureAD to Microsoft Graph

![Article Figure](https://cdn-images-1.medium.com/max/800/1*7sDFBqW7goMfDn6TiVoahQ.jpeg)

The Old Guard: MSOnline and AzureAD For years, MSOnline (Azure Active Directory PowerShell module) and AzureAD (Azure Active Directory Module for Windows PowerShell) have been the go-to tools for IT professionals managing Azure Active Directory (Azure AD). These modules provided streamlined methods for handling user accounts, groups, and other directory resources, tailored for specific administrative tasks, making them easy to use.

However, as technology evolves, so do the tools we use. Enter the new powerhouse: Microsoft Graph.

#### What is Microsoft Graph?

1. A Shift from Specialized Modules to Unified API Access: Unlike MSOnline and AzureAD, which had unique command sets, Microsoft Graph functions as a unified PowerShell module wrapped around REST API calls. It serves as a bridge between PowerShell and the Microsoft Graph API, offering broader access across the Microsoft 365 ecosystem.
2. RESTful Command Structure: MS Graph’s commands map directly to the standard HTTP methods used in RESTful APIs: GET, PUT, POST, PATCH, and DELETE. This structure not only makes MS Graph more flexible but also aligns it with industry-wide standards, easing the learning curve for IT pros familiar with REST APIs.
3. One API for Everything Microsoft: Beyond just Azure AD, Microsoft Graph covers Microsoft 365 services like SharePoint, OneDrive, Teams, and more. It’s the one-stop shop for managing resources across the Microsoft 365 environment.

#### Working with MSOnline: The Legacy Approach

While MSOnline is being phased out, many admins still rely on it. Here’s a quick guide to some essential commands. (Some screenshots of the input/output will be included to guide you through the process) :

Installing and Importing MSOnline: If you’re using PowerShell 7 (PS7), you might face compatibility issues with the Connect-MsolService cmdlet. It's best to stick with PowerShell version 5.1 for now:

```
Install-Module MSOnline
Import-Module MSOnline
Connect-MsolService
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*bBPdQ_XJ5bBmEMwiGiQFBg.png)

Retrieving User Information: The following command fetches user details, limited to 5 results:

```
Get-MsolUser -MaxResults 5 | Select DisplayName, UserPrincipalName, Department
```

Modifying User Properties: Here’s how to update a user’s department::

```
Get-MsolUser -UserPrincipalName "user@example.com" | Set-MsolUser -Department "Sales"
```

Listing Security Groups: To retrieve all Distribution lists:

```
Get-MsolGroup -All | Where-Object {$_.GroupType -eq "DistributionList"}
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*YHgjfsNAMVasmk3z_65V8w.png)

#### AzureAD Module: Expanding Administrative Reach

For tasks requiring deeper integration with Azure AD, the AzureAD module offers more functionality than the Azure portal itself. This module supports automation, user management, and group administration at scale:

Installing and Connecting:

```
Install-Module AzureAD
Import-Module AzureAD
Connect-AzureAD
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*zaPbGrhVlBmwM0kIep8jBA.png)

Retrieving Users by Department: To filter users based on their department (e.g., “Sales”):

```
Get-AzureADUser -Filter "Department eq 'Sales'"
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*FPeCSZwUb-UYbD6-TPc14g.png)

The New Contender: Microsoft Graph PowerShell Module Microsoft Graph is the modern solution for accessing data across the Microsoft 365 platform. Covering everything from Exchange Online to Azure AD, it consolidates what MSOnline and AzureAD did but on a much larger scale.

Installing and Connecting:

```
Install-Module Microsoft.Graph
Import-Module Microsoft.Graph
Connect-MgGraph -Scopes "Directory.ReadWrite.All"
```

Retrieving Users and Groups: Here’s how to query user and group data with Microsoft Graph:

```
Get-MgUser -All | Select DisplayName, Id, UserPrincipalName
Get-MgGroup -Filter "DisplayName eq 'Engineering Department'"
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*BX8Xzd5r2fdRk0PvomMVQA.png)

Creating a New Group: To create a new group in Microsoft 365:

```
New-MgGroup -DisplayName "TechSupport" -MailEnabled:$false -MailNickname "TechSupport" -SecurityEnabled:$true -Visibility "Private"
```

![Article Figure](https://cdn-images-1.medium.com/max/800/1*VcXq5z1RS7xjQtnlby1ilQ.png)

Conclusion The transition from MSOnline and AzureAD to Microsoft Graph marks a significant shift in how IT administrators manage Microsoft 365 environments. With expanded functionality and broader integration, Microsoft Graph offers more capabilities — it’s not just a replacement, it’s an upgrade. That said, if you’re like me, you might still find some comfort in the simplicity of the old commands. The ease of use with the older modules has its appeal, and while I appreciate what Microsoft Graph brings to the table, I haven’t fully decided if I’m ready to let go of the old tools just yet.
