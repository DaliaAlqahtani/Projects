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
- The tenant is named **Najm Horizon Consulting**.
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

**Why I used Cloud Sync ?**

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

**Admin roles assignment**

the roles planned in Step 2 were assigned to the IT staff

- User Administrator: Nawaf AlRashidi (IT Manager)
- Helpdesk Administrator: Omar AlQahtani and Turki AlEnezi (Helpdesk Specialist and IT Support Technician) 
- Groups Administrator: Walid AlRuwaili (Identity & Access Admin)
- Authentication Administrator: Tariq AlZahrani (Cybersecurity Analyst)

<p  align="center"><img width="1800" height="681" alt="user-role-assignments" src="https://github.com/user-attachments/assets/4fd63ee2-82a6-4648-a1bb-f82a8ae30c9a" /></p>
 
**Mapping synced users to cloud groups**

I used a PowerShell script that read which OU each user belonged to on-premises and added the matching cloud user to the matching group, matching them by sign-in name.
<p  align="center"><img width="1473" height="996" alt="populate-security-groups" src="https://github.com/user-attachments/assets/ded5a747-af0d-4e73-a7ac-7f19149e9453" /></p>
<p  align="center"><img width="1567" height="1065" alt="security-groups-members(script)" src="https://github.com/user-attachments/assets/749576c2-43e4-4d8a-9dc9-6a024a522060" /></p>


## Step 4 - HR database and the Logic App

**Choosing the data source**

Since this is a lab environment without access to enterprise HR platforms such as SAP SuccessFactors or Workday, I implemented a lightweight HR system using Azure SQL Database as the data source.

**Governance and naming**

Resources were placed under a clear structure: a management group (`mg-najmhorizon`) over the subscription, with all resources in one resource group (`rg-najmhorizon-prod-itn-001`). Names follow the Microsoft Cloud Adoption Framework pattern with environment (`prod`) and region (`itn`) tags:

| Resource | Name |
|---|---|
| Management group | `mg-najmhorizon` |
| Resource group | `rg-najmhorizon-prod-itn-001` |
| SQL server | `sql-najmhorizon-prod-itn` |
| SQL database | `sqldb-najmhorizon-hr-prod-itn` |
| Logic App | `logic-najmhorizon-onboarding-prod-itn` |

**The Employees table**
 
The table used an auto-incrementing ID as its primary key. That column is what the Logic App watches to spot new rows, so it is required, not optional.

```sql
CREATE TABLE dbo.Employees (
    EmployeeID    INT IDENTITY(1,1) PRIMARY KEY,
    FirstName     NVARCHAR(100) NOT NULL,
    LastName      NVARCHAR(100) NOT NULL,
    Department    NVARCHAR(50)  NOT NULL,
    JobTitle      NVARCHAR(100) NOT NULL,
    PersonalEmail NVARCHAR(256) NOT NULL
);
```

<p  align="center"><img width="2560" height="1199" alt="table-created" src="https://github.com/user-attachments/assets/0c533133-0e90-4bc6-be5c-fdde9a91aed3" /></p>

**The Logic App**

The purpose of the Logic App is that after a new row is inserted into the table, the Logic App is triggered, it generates the user's UPN and a temporary password, creates the user in Entra ID through Microsoft Graph and map it to the right department, then finally sends a welcome email done in [Step 5](#step-5---department-routing-and-welcome-email).
 
A Consumption Logic App was created in the same region and resource group as the database. It was given a **system-assigned managed identity**, and that identity was granted the Microsoft Graph permission `User.ReadWrite.All` through PowerShell.

First, the system-assigned managed identity was enabled

<p  align="center">https://github.com/user-attachments/assets/4121e618-6d0a-4c9a-a969-133b81aa0fe0</p>

Then, the graph permission granted

```powershell 
# --- Variables ---
$logicAppName = "logic-najmhorizon-onboarding-prod-itn"   # the MI shares the Logic App's name
$graphAppId   = "00000003-0000-0000-c000-000000000000"     # Microsoft Graph AppId
$permission   = "User.ReadWrite.All"

# --- Service principal of the Logic App's managed identity ---
$msi = Get-MgServicePrincipal -Filter "displayName eq '$logicAppName'"

# --- Microsoft Graph service principal in the tenant ---
$graphSp = Get-MgServicePrincipal -Filter "appId eq '$graphAppId'"

# --- Find the application permission (app role) ---
$appRole = $graphSp.AppRoles |
    Where-Object { $_.Value -eq $permission -and $_.AllowedMemberTypes -contains "Application" }

# --- Assign it to the managed identity ---
New-MgServicePrincipalAppRoleAssignment `
    -ServicePrincipalId $msi.Id `
    -PrincipalId        $msi.Id `
    -ResourceId         $graphSp.Id `
    -AppRoleId          $appRole.Id
```

The grant can be verified with the `User.ReadWrite.All` permission:

<p align="center"><img width="2560" height="1031" alt="verify-grant-managed-identity-permissions" src="https://github.com/user-attachments/assets/b30ebcb7-e5aa-4b01-ac70-a8a4661eea82" /></p>

The workflow:

1. **Trigger:** the SQL "When an item is created" trigger, watching `dbo.Employees` every 3 minutes.
<p align="center"><img width="329" height="500" alt="trigger-configuration" src="https://github.com/user-attachments/assets/7cfa6f0a-8ec7-493d-bdbf-21ef80076c03" /></p>

2. **Build the values:** four small Compose steps build the sign-in name, the display name, and a temporary password:
   - Mail nickname: first and last name, lowercased and joined with a dot.
   - Sign-in name (UPN): the mail nickname plus `@najmhorizon.onmicrosoft.com`.
   - Display name: first and last name with a space.
   - Temporary password: a fixed prefix plus part of a generated GUID, which always meets the complexity rules and is different every time.

  <p align="center">https://github.com/user-attachments/assets/2c97f370-9c81-4f45-adc9-a83d7d524111</p>

3. **Create the user:** an HTTP call to the Microsoft Graph users endpoint, authenticated with the managed identity, with the new user set to change their password at first sign-in using `forceChangePasswordNextSignIn: true` in the HTTP body.

<p align="center"><img width="529" height="500" alt="logic-app-http-action" src="https://github.com/user-attachments/assets/2ddac2c8-e8d1-4eb4-9d6e-3c9f450340f6" /></p>


 
## Step 5 - Department routing and welcome email

This step extends the same Logic App. For users to be mapped to their corresponding department, I added another graph permission `GroupMember.ReadWrite.All` granted to the same managed identity with the same PowerShell script as before.

**Routing by department**
 
A Switch control looks at the Department value from the new row and branches into one of four cases (IT, Finance, Sales, Marketing). Each case adds the newly created user to the matching `SG-NJH-*` group with an HTTP call to Graph. Only the group ID changes between the four cases; everything else is identical.

First, I used a script to view each group's object ID

<p align="center"><img width="900" height="500" alt="security-groups-object-id" src="https://github.com/user-attachments/assets/2fc6be8b-ce1f-4721-85ec-4310b0c7207b" /></p>

Then the Switch with the four cases and the HTTP call for each department

<p align="center">https://github.com/user-attachments/assets/826a0e85-e042-4a34-89e9-316284d6c099</p>


**Welcome email**
 
The last action emails the new hire their details.

The sender used the **Outlook.com connector**. The email is sent to the personal email address from the table row and includes a short HTML template with the company logo, the person's display name, job title, department, new sign-in name, and temporary password.

<p align="center"><img width="1889" height="1166" alt="welcome-email-action" src="https://github.com/user-attachments/assets/4ea13962-13a0-49b1-942e-ca745b17ad07" /></p>


## Testing and verification

Because pre-existing rows do not fire the trigger, the Logic App was finished and enabled first, then a single test row was inserted to the **Employees** table so it would actually run:
 
```sql
INSERT INTO dbo.Employees (FirstName, LastName, Department, JobTitle, PersonalEmail)
VALUES ('Khalid', 'AlDosari', 'IT', 'Network Engineer', 'dalianaalmanea+khalid@gmail.com');
```
<p align="center"><img width="2554" height="906" alt="new-hire-row" src="https://github.com/user-attachments/assets/bcec8d47-c687-41de-ae8c-852be61b0da4" /></p>

**Note** I used my personal email so I can view the email message I've created.

The full run succeeded end to end:

<p align="center"><img width="2560" height="1042" alt="logic-app-run" src="https://github.com/user-attachments/assets/7675a25f-a642-4a82-8db8-85cad07faaba" /></p>


- The new row fired the trigger within 3 minutes interval.
- A new user, `khalid.aldosari@najmhorizon.onmicrosoft.com`, was created in Entra ID.
- Because the department was IT, the user was added to `SG-NJH-IT`.

 <p align="center"><img width="2269" height="723" alt="new-hire-view" src="https://github.com/user-attachments/assets/9aca959c-323b-42ec-abba-5f67efba3673" /></p>

- A welcome email arrived at the personal address with the sign-in name and temporary password.

  <p align="center"><img width="2560" height="1182" alt="welcome-email" src="https://github.com/user-attachments/assets/4c0f0d2b-968c-4988-b0d7-81a1f4de95f2" /></p>

  ## Design decisions and trade-offs
  
  **Managed identity instead of a stored secret.** The Logic App authenticates to Microsoft Graph using a managed identity that Azure rotates automatically. The common approach stores a secret string that expires, has to be rotated, and can be leaked. The managed identity removes that secret entirely and improves security.

  **Cloud Sync instead of the classic Connect tool.** The classic installer is retiring soon, and Cloud Sync is the current supported path, so this was both the necessary and the modern choice.

  **Global Administrator kept cloud-only.** The top admin account is not a synced identity. If the on-premises environment were ever compromised, a synced admin account would hand that control straight to the cloud. Keeping it cloud-only contains the damage.



