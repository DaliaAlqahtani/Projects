# On-Premises to Cloud Migration and HR Onboarding Automation
 
A hands-on lab project that connects a local Active Directory environment to Microsoft Entra ID and then automates new-hire account creation using Azure Logic Apps.

## Scenario

**Najm Horizon Consulting**, a company with more than 250 employees across four departments: IT, Finance, Sales, and Marketing. The company runs a traditional on-premises Active Directory for staff accounts, but it has started moving services to the cloud and wants to set up its identity.

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
<img width="2555" height="1483" alt="architecture diagram" src="https://github.com/user-attachments/assets/4cce4692-22fa-450a-be2e-ad7b2ea6e846" />

## Environment and tools
 
| Area | What I used |
|---|---|
| On-premises host | VMware, running a Windows Server 2022 virtual machine |
| Domain controller | `NAJM-DC01`, forest `najmhorizon.local`, static IP `10.20.0.10` |
| Cloud tenant | `najmhorizon.onmicrosoft.com`, Microsoft Entra ID Free tier |
| Subscription | Azure for Students *(region limitation, see note)* |
| Sync tool | Microsoft Entra Connect Cloud Sync |
| Automation | Azure Logic Apps (Consumption) |
| HR data source | Azure SQL Database (free tier) |
| Scripting | PowerShell and the Microsoft Graph PowerShell module |
| Deployed region | Italy North (`itn`) |

*A note on the region:* Azure for Students subscriptions only allow a limited set of regions. I deployed to Italy North, and the resource names use the `itn` short code to reflect that. 

## On-premises Active Directory

**Networking**
 
The server was given a fixed address inside the company range so it never changes:
 
- IP address: `10.20.0.10/16`
- Default gateway: `10.20.0.1/16`
- DNS: pointed at the loopback address `127.0.0.1` since the server is also the DNS server

**Domain controller**
 
The Active Directory Domain Services role was installed and the server was promoted to the first domain controller in a new forest called `najmhorizon.local`.

**Organizational units and users**
 
Four organizational units were created, one per department, populated with users.

<p align="center">https://github.com/user-attachments/assets/4dbdc04b-e560-4ec8-92e8-fb47894c4829</p>

**Routable sign-in name**
 
By default, local accounts sign in as `name@najmhorizon.local`, which is a private suffix the cloud cannot use. To avoid sync problems later, a routable suffix (`najmhorizon.onmicrosoft.com`) was added using the script below, and every existing user was re-stamped so their sign-in name reads `First.Last@najmhorizon.onmicrosoft.com`. 

<p align="center"><img width="1741" height="212" alt="on-prem-pre-migration-upn-suffix-for-users" src="https://github.com/user-attachments/assets/a262edae-7c90-4463-8534-a4a33787a05f" /></p>

After the script ran, the routable suffix can be viewed 

<p align="center"><img width="333" height="403" alt="on-prem-pre-migration-additional-upn-suffix(copy)" src="https://github.com/user-attachments/assets/54b6415a-8811-43c0-a8ab-40fe455d88d0" /> <p/>

## Step 2 - Cloud groups and admin roles

**Tenant**

Here is an overview of the tenant used for this project. 
- The tenant is named Najm Horizon Consulting.
- Primary Domain changed to fit the company name as `najmhorizon.onmicrosoft.com`. In a real production environment, the Domain name would end in a real domain like `.com` or `.net` not a fallback domain.
- A Global Administrator was created to have full control, kept to a cloud-only account.

<p align="center"><img width="2560" height="939" alt="azure-overview-initial-setup" src="https://github.com/user-attachments/assets/d6f22c14-21ab-4152-879f-02217840855e" /></p>

**Security groups**
 
Four security groups were created using a simple, consistent naming standard, `SG-NJH-<Department>`:
 
- `SG-NJH-IT`
- `SG-NJH-Finance`
- `SG-NJH-Sales`
- `SG-NJH-Marketing`

```powershell
$departments = "IT","Finance","Sales","Marketing"

foreach ($dept in $departments) {
    New-MgGroup `
        -DisplayName "SG-NJH-$dept" `
        -Description "Cloud security group for the $dept department" `
        -MailEnabled:$false `
        -MailNickname "SG-NJH-$dept" `
        -SecurityEnabled:$true
}
```

<p align="center"><img width="2560" height="813" alt="security-groups" src="https://github.com/user-attachments/assets/aa930873-0945-4be8-a2dd-5901be515cd7" /></p>

These use **Assigned** membership, where members are added directly. The **Dynamic** membership (where rules add people based on attributes) needs a paid license, and I only used the free tier.

These use **Assigned** membership, where members are added directly. The more automatic **Dynamic** membership (where rules add people based on attributes) needs a paid license, so it was not an option on the free tier. This turned out to be the right call anyway, explained in the design decisions below.
 
**Admin role plan**
 
A least-privilege role plan was written to the staff of the IT department so no one has more access than they need:
 
| Role | Purpose |
|---|---|
| User Administrator | Create and manage users |
| Helpdesk Administrator | Reset passwords and basic support |
| Groups Administrator | Manage group membership |
| Authentication Administrator | Manage sign-in and authentication methods |

The actual role assignments were done after the IT staff were synced to the cloud

## Step 3 - Directory synchronization
 
This step connects the two directories so on-premises users appear in the cloud and sign in with the same password. For this I used Microsoft's Entra Cloud Sync.

**Why Cloud Sync**

Cloud Sync was selected over Entra Connect to demonstrate a modern hybrid identity architecture, since Entra Connect is retiring September 30, 2026. 

**What was set up**
 
- The provisioning agent was installed on `NAJM-DC01` and registered.
<p align="center"><img width="2068" height="1137" alt="on-prem-cloud-sync-agent-config" src="https://github.com/user-attachments/assets/3f7f3bbf-c7a1-4e9c-a520-34ab418f0b9c" /></p>

- In the cloud portal, sync was configured, with the agent synced as **active**.
 <p align="center"><img width="1998" height="576" alt="cloud-sync-agent" src="https://github.com/user-attachments/assets/7f6220b5-65aa-44b8-adba-f4cb3ac128a7" /></p>

- **Password Hash Synchronization** was turned on so users keep the same password in both places.
<p align="center">https://github.com/user-attachments/assets/babf5d65-7c30-41ce-9706-3fee4e5ec8a8</p>

- Sync was verified by confirming the on-premises users now appear as synced accounts in Entra ID.
<p align="center"><img width="2268" height="635" alt="synced-users" src="https://github.com/user-attachments/assets/08898a92-9d7d-4040-9284-7ed8d94b7bb2" /></p>


