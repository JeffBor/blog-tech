# Checking Azure AD Logs for Basic Authentication

# <https://www.enowsoftware.com/solutions-engine/microsoft-stops-basic-authentication-now-what>

During the second half of 2021, [Microsoft will cease supporting Basic Authentication](https://techcommunity.microsoft.com/t5/exchange-team-blog/basic-authentication-and-exchange-online-april-2020-update/ba-p/1275508).  This sounds scary and yet another PITA thing we have to deal with thanks to Microsoft, or is it?  Fear is the first sign of wisdom, so let's get wise and figure out what we need to do before it gets us.

## Basic vs. Modern Authentication

What is Basic Authentication anyway?  In the world of authentication (a la Microsoft at least) there are two general methods of determining who a user is: Basic and Modern.  You guessed it, Modern is where we are now and have been for many years, however Basic is a legacy method going back to the stone age of authentication.

With Basic, a user is authenticated by sending credentials in the Authenticate header.  Way back when during the Wild West Days of Authentication, that's all we had, and there were (are) all sorts of security problems.  For example, Basic Authenticate headers may go over the wire unencrypted as clear text if the user (or unthoughtful IT staff managing servers) were not careful.  Also, bad guys could pretty easily perform cross-site request forgery (CSRF) attacks, performing requests in the user's security context thus ables to do whatever that user can do on the authenticated site or service.  What could go wrong!?

Enter Modern Authentication that not only solves many or arguably all Basic's problems, but ushers in the capability of efficient authentication tokens with TTLs, claims, conditional access, etc.

So what should we do about all this?  How do we know if we have some lurking dependency on Basic that Suzy Smith in Accounting uses every day for payroll processing?

You need to how your users are authenticating and why.  As a consultant, I've helped many orgs noodle this out in their environments

## Your Friend Azure AD

Azure AD keeps a log of all authentication requests or "sign-ins" for every user.  If you have not discovered this in the Portal, you should check it out now for heaven's sakes!  It can save your posterior when debugging user logins. If anything, hit the pause button on this and go there now: <https://portal.azure.com/#blade/Microsoft_AAD_IAM/ActiveDirectoryMenuBlade/SignIns>.

You can use the Portal UI to get a filter logs, but real men don't use the Portal.  They use PowerShell or the Az CLI!  We may need to automate custom analysis of authentications to present a full picture of the situation, so let's crack open PowerShell and get started.

First, you need the AzureAD module for PowerShell.  Azure AD, even though it has the word "Azure" in its name, is really a separate service in the Microsoft cloud verse, so we need to grab its module:

```powershell
Install-Module AzureAD
```

If you're not logged into your PowerShell session as an admin, you will get something like:

```text
install-module : Administrator rights are required to install modules in ...
```

No worries, actually this is not a bad thing.  Before you go and fire up another runas admin session, just consider doing this:

```powershell
Install-Module -Name AzureAD -Scope CurrentUser
```

This will enable the module for the current user.  Within seconds you should have AzureAD module installed:

```powershell
Get-Module -Name AzureAD
```

Next, let's authenticate to AAD:

```powershell
Connect-AzureAD
```

A window will pop up and prompt for authentication.  Needless to say, this is indeed Modern authentication happening here, and if enabled, multi-factor or even password-less will kick in.  Awesome!

Now let's get some logs:

```powershell
Get-AzureADAuditSignInLogs -Top 10
```

If you do not have an Azure AD Premium license, then you may receive the following:

```text
Get-AzureADAuditSignInLogs : Error occurred while executing GetAuditSignInLogs
Code: Authentication_ApplicationHasNoDirectoryReadAccess
Message: Failed to do premium license check from ADGraph
```

Let's try a different one:

```powershell
Get-AzureADAuditDirectoryLogs -Top 10
```

If you receive the following, this means permission need to be set correctly to read the Microsoft Graph:

```text
Get-AzureADAuditDirectoryLogs : Error occurred while executing GetAuditDirectoryLogs
Code: Authentication_MSGraphPermissionMissing
Message: Application missing MSGraph API 'Read all audit log data' permission
```

Nope, one cannot just assume you can start reading reporting/audit data without permission!  Fear not; [I have an article on how to get this going via Terraform](msgraph-api-app-registration).
