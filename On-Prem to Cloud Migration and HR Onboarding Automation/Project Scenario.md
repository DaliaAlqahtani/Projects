# On-Premises to Cloud Migration and HR Onboarding Automation
 
A hands-on lab project that connects a local Active Directory environment to Microsoft Entra ID and then automates new-hire account creation using Azure Logic Apps.

## Scenario

**Najm Horizon Consulting** a company with more than 250 employees across four departments: IT, Finance, Sales, and Marketing. The company runs a traditional on-premises Active Directory for staff accounts, but it has started moving services to the cloud and wants to set up its identity.

Two problems needed solving:
 
1. **Two separate worlds.** Staff accounts lived only in the local Active Directory (`najmhorizon.local`). The cloud tenant (`najmhorizon.onmicrosoft.com`).
2. **Manual onboarding.** Every time HR hired someone, an IT administrator had to create the account by hand, set a password, and add the person to the right groups. For a company hiring at this scale, that is slow and easy to get wrong.

The goal of this project was to fix both: link the on-premises directory to the cloud so every employee has one identity, and then build an automated flow that creates a new cloud account the moment HR adds a new employee record, including putting the person in the correct department group and emailing them their sign-in details.

This lab was built using Windows Server 2022, Azure for Students subscription, and the free tier of Microsoft Entra ID

## What this project shows

- Connecting on-premises Active Directory to Microsoft Entra ID and keeping passwords in sync.
- Assigning administrative roles using the principle of least privilege.
- Building a cloud automation that reacts to a new database record and creates a user.
- Using a managed identity instead of a stored password so there is no secret to leak.
- Routing users to the right group based on their department.
- Sending an automated welcome email.

## Architecture overview
