## My SOP to Building and Validating an Active Directory Lab in Azure

## Overview

This SOP walks through building a fully functional Active Directory environment in Azure: a domain controller, a domain-joined client machine, an OU structure with users and role-based security groups, DNS configuration, Group Policy, and delegated administration. It's written so a team member can follow it step by step and reproduce the environment reliably.

**End state:**
- 1 domain controller + 1 domain-joined client VM
- OU structure organized by location (Houston, Tokyo), each with Users, Groups, and Workstations containers
- 6 users across 2 locations, assigned to role-based security groups (Help Desk, IT Support, Accounting)
- Domain-wide password policy enforced via Group Policy
- Password-reset permissions delegated to Help Desk (least-privilege access, not Domain Admin)

## Prerequisites

- Active Azure subscription with permissions to create resource groups and VMs
- Basic familiarity with the Azure Portal and Remote Desktop
- No prior Active Directory experience required — each concept is explained inline

## Table of Contents

1. [Phase 1 — Environment Foundation](#phase-1--environment-foundation)
2. [Phase 2 — Deploy the Domain Controller](#phase-2--deploy-the-domain-controller)
3. [Phase 3 — Install Active Directory](#phase-3--install-active-directory)
4. [Phase 4 — Build the OU, User, and Group Structure](#phase-4--build-the-ou-user-and-group-structure)
5. [Phase 5 — Deploy and Join the Client VM](#phase-5--deploy-and-join-the-client-vm)
6. [Phase 6 — Group Policy and Delegated Administration](#phase-6--group-policy-and-delegated-administration)
7. [Phase 7 — Recovery Options and Cleanup](#phase-7--recovery-options-and-cleanup)
8. [Cautionary Notes](#cautionary-notes)
9. [Tips for Efficiency](#tips-for-efficiency)

---

## Phase 1 — Environment Foundation

### 1. Create the Azure lab foundation [1:08](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=68)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/e6d192f4-2837-4b64-9c96-14348fe014cb" />

- Sign in to the **Azure Portal**.
- Create a **new Resource Group** to contain all lab assets (e.g. `ADLab`).
- Treat the resource group as a single folder for everything related to this lab — it makes cleanup trivial later.

---

## Phase 2 — Deploy the Domain Controller

### 2. Deploy the domain controller virtual machine [2:37](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=157)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/0d9c8cb9-83e2-43b1-94fd-acfc23ad3e87" />

- Create the first **Windows Server virtual machine** (e.g. `ControllerVM` or `DC01`).
- Select the correct region; leave availability settings at default unless redundancy is required.
- Image: **Windows Server 2025 Datacenter (64-bit, Gen 2)**.
- Choose the lowest-cost VM size suitable for a lab environment.
- Set the administrator username and a strong password.
- Allow **RDP** access for initial administration.
- Keep disk settings at default unless the lab requires changes.

### 3. Configure networking and cost controls [6:32](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=392)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/463be332-5c07-41d9-ae0d-3ab1fca676fa" />

- Use **one virtual network and one subnet** shared by both VMs.
- Enable deletion of the public IP and NIC when the VM is deleted.
- Configure **auto-shutdown** (e.g. 2:00 PM daily) with an email notification, to control cost.
- Review settings and create the VM.

### 4. Set the domain controller's private IP to static [8:38](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=518)
![generated-image-at-00:08:38](https://loom.com/i/34b9f3b4e0174a8585aa44ab3569e367?workflows_screenshot=true)

- Open the domain controller resource → **Networking** → **NIC** → **IP configuration**.
- Change the private IP assignment from dynamic to **static**.
- Save before continuing — a static IP is required so the client VM can reliably point its DNS here later.

### 5. Connect to the domain controller [9:27](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=567)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/61460d5c-291e-4b4c-a726-48bc5f7f4a0d" />

- Select **Connect** on the VM resource and use the private IP shown.
- Connect via **Remote Desktop Connection** (Windows) or the **Windows App** (other platforms).
- Recommended settings: **single display** if using multiple monitors, **clipboard redirection** for pasting commands.
- Sign in with the Azure-created administrator credentials.

---

## Phase 3 — Install Active Directory

### 6. Install AD DS and promote the server [12:19](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=739)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/26358834-3ac2-449b-97bb-5b699b39b680" />

- Open **Server Manager** → **Add Roles and Features** → install **Active Directory Domain Services (AD DS)**.
- Use the post-install notification flag to start **Promote this server to a domain controller**.
- Restart if prompted, then continue after reboot.
- Create a **new forest** with an internal domain name (e.g. `lab.local`) — never a public domain you don't own.
- Set the **Directory Services Restore Mode (DSRM)** password and store it securely.
- Run the prerequisite check and complete the promotion; allow the server to restart.

### 7. Log in using the new domain format [17:14](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=1034)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/bd21daf1-cc07-4d9f-b6b1-03eadc81c3ad" />

- Sign in using **NetBIOS domain \ username** format, e.g. `LAB\ControllerVM`.
- Use the same password created in Azure.
- Confirm the domain controller is active under the new domain context.

---

## Phase 4 — Build the OU, User, and Group Structure

### 8. Create the OU structure [18:09](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=1089)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/29ec0ce3-daf9-4990-a184-2f4d11c7d1f2" />

- Open **Active Directory Users and Computers (ADUC)** and enable **Advanced Features**.
- Create a top-level OU, `Branches`.
- Under it, create location OUs: `Houston`, `Tokyo`.
- Under each location, create sub-OUs: `Users`, `Groups`, `Workstations`.
- Leave accidental-deletion protection enabled unless you intentionally need to remove an OU.

### 9. Create users in each location OU [28:52](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=1732)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/ba7ab4c6-8204-44ec-b54e-75f4b80df3fc" />

- Create users via **GUI** or **PowerShell** — three per location.
- **Houston:** Alice Johnson, Bob Martinez, Christopher Walker
- **Tokyo:** Kenji Sato, Yuki Tanaka, Hana Ito
- Use a consistent naming and password standard; verify each user lands in the correct OU.

### 10. Create security groups for role-based access [36:18](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=2178)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/aa5e2d53-8174-4671-9574-69a1d778489e" />

- Create three **Security**-type groups in each location OU: `Help Desk`, `IT Support`, `Accounting`.
- Keep names consistent across locations (append a location tag if a naming conflict arises).
- Confirm groups appear under the correct location/group OU.

### 11. Assign users to security groups [50:43](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3043)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b7dc7d9e-e57f-44cc-9428-57b550c3ffa7" />

- Open each user's **Properties → Member Of** tab and add the correct group.
- Example: Alice Johnson → Help Desk, Bob Martinez → Accounting, Christopher Walker → IT Support.
- Repeat for Tokyo users; verify membership after saving.

---

## Phase 5 — Deploy and Join the Client VM

### 12. Deploy the client virtual machine [41:06](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=2466)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/87ec4566-6a0f-4097-a58f-48bfa05a7208" />

- Create a second VM (`ClientVM`) in the **same resource group**, same region/image/size for consistency.
- Set a new admin username and password.
- Use the **same virtual network and subnet** as the domain controller.
- Enable auto-shutdown; review and create.

### 13. Configure DNS on the client VM [44:44](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=2684)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/22712893-c33e-4d7d-a49d-0abe7f0b3e36" />

- Open **Network Connections → adapter's IPv4 properties**.
- Change DNS from automatic to **manual**; set **Preferred DNS server** to the domain controller's static private IP.
- **This step is required** — incorrect DNS is the most common cause of domain-join failure.

### 14. Join the client VM to the domain [47:37](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=2857)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/30702d45-c1d3-40ba-a4bf-b04dca31e11c" />

- **System Properties → Change → Domain**, enter the domain name (e.g. `lab.local`).
- Authenticate with domain controller credentials; confirm and restart.
- After reboot, verify the machine shows as domain-joined.

### 15. Move the client computer object into the correct OU [49:37](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=2977)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/11df3561-83c8-4f13-983e-51bc990b2958" />

- In ADUC, locate the client object in the default **Computers** container.
- Move it into the correct OU (e.g. `Houston > Workstations`) — Group Policy won't apply correctly if it's left in the default container.

### 16. Enable Remote Desktop access for test users [53:02](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3182)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/b988a702-30eb-491e-8ec2-d53096f9b7a8" />

- Domain users can't RDP in by default. Add the relevant users or groups (Help Desk, IT Support, Accounting) to the **Remote Desktop Users** group.
- Verify membership before testing logins.

### 17. Test domain login from the client VM [54:24](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3264)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/125dc3b4-bc31-451c-9979-9325f77d534a" />

- Sign in using domain format, e.g. `LAB\Alice Johnson`, with the user's password.
- A successful login validates the domain join, DNS configuration, and RDP permissions all at once.

---

## Phase 6 — Group Policy and Delegated Administration

### 18. Configure a baseline domain password policy [55:29](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3329)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/c416a2fe-2919-4e64-9f3e-04f1246d1383" />

- Open **Group Policy Management** → edit **Default Domain Policy**.
- Navigate: `Computer Configuration → Policies → Windows Settings → Security Settings → Account Policies → Password Policy`.
- Set: minimum password age **30 days**, minimum password length **12 characters**.
- Save and apply.

### 19. Delegate password-reset tasks to Help Desk [56:41](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3401)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/2137e916-318a-4ee1-a158-98c8038d7fcb" />

- In ADUC, right-click the relevant OU (e.g. `Houston > Users`) → **Delegate Control**.
- Add the **Help Desk** group; assign password reset and "must change password at next logon."
- Complete the wizard and verify the delegation.
- **Why this matters:** it lets Help Desk handle routine resets without holding full Domain Admin rights — least-privilege access in practice.

---

## Phase 7 — Recovery Options and Cleanup

### 20. Enable recovery options and clean up [58:59](https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e?t=3539)
<img width="1920" height="1080" alt="image" src="https://github.com/user-attachments/assets/46a5f389-0601-438e-842b-0c5211e126f3" />

- In **Active Directory Administrative Center**, enable the **Recycle Bin** (if available) to recover accidentally deleted objects.
- Use **Attribute Editor** and object properties for troubleshooting.
- When finished, delete the **resource group** in Azure to remove all lab resources at once, and confirm deletion to avoid ongoing charges.

---

## Cautionary Notes

- Use an **internal domain name** (e.g. `lab.local`); never a public domain you don't own.
- Store the **DSRM password** securely — it's critical for disaster recovery.
- Set the domain controller's private IP to **static** before joining any clients.
- The client VM must use the **domain controller as its DNS server** — incorrect DNS is the most common domain-join failure cause.
- Don't leave domain-joined computers in the default **Computers** container if Group Policy needs to apply.
- Be careful deleting OUs or users — accidental deletion can disrupt the whole lab.
- Verify group membership before testing Remote Desktop logins.
- Shut down or delete Azure resources when finished to avoid unnecessary cost.

## Tips for Efficiency

- Use **one resource group** for the entire lab so cleanup is a single step.
- Use **PowerShell ISE** for faster OU, user, and group creation.
- Keep naming consistent across all objects — VMs, OUs, users, groups.
- Save frequently used credentials and IPs in a secure note.
- Enable **clipboard redirection** in the RDP client to speed up command entry.
- Use **auto-shutdown** on both VMs to control cost during idle time.
- Build order: OU structure → users and groups → client join → access testing.
- Use **Recycle Bin** and **Attribute Editor** for faster troubleshooting and recovery.

---

**Full walkthrough (Loom):** <https://loom.com/share/c1c79b17ca5e4ec3bf6a210d0ad19f3e>
