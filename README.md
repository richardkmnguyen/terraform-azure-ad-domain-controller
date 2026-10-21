Watch me explain this lab: [LoomLink](https://www.loom.com/share/1540010d8ce54a4d80038355543fd087)
 
# terraform-azure-ad-domain-controller
 
Used Terraform to deploy a Windows Server 2022 VM in Azure and promote it to an Active Directory domain controller (`corp.richard.com`) in a single `terraform apply`, then worked through six real deployment errors (a PowerShell syntax mismatch, a DNS lookup failure, VM capacity and quota limits, a hypervisor generation conflict, and an extension polling bug) to get a working, verified domain controller built entirely from code.

# Lab 6: Infrastructure as Code with Terraform

**Terraform · Azure · Active Directory · PowerShell · Infrastructure as Code**
 
In Lab 1 I built a domain controller in Azure by hand. In this lab I rebuilt the same idea as code: one set of Terraform files that creates the network, the VM, and the Active Directory forest with no clicking through the portal. Getting a clean deployment took more than running the commands in the guide. Along the way I hit errors from my shell, my network, Azure's capacity and quota limits, and a Terraform provider quirk, and each one taught me something about how the cloud actually behaves. Below is what I built, what broke, and what I took from it.
 
This lab builds on my earlier ones: Lab 1 built a domain controller manually, Lab 2 was network level analysis, Lab 3 was log level monitoring, and this lab automates the environment so it can be created and destroyed on demand.
 
| | |
|---|---|
| **Platform** | Terraform and Azure CLI on a local Windows 10 PC (PowerShell), deploying a Windows Server 2022 Azure VM |
| **Time** | 4 hours |
 
---
 
## What Infrastructure as Code is for
 
Clicking through a portal is slow, hard to repeat, and impossible to review. **Infrastructure as Code (IaC)** describes infrastructure in text files so it can be created, changed, and destroyed the same way every time, and so the files themselves can be version controlled. Terraform is one of the most common IaC tools: you describe the end state you want, and Terraform works out what to create or change to get there.
 
The core Terraform workflow has four commands:
 
- **`terraform init`:** downloads the provider plugins (here, the Azure provider) the files need.
- **`terraform plan`:** compares the files to what already exists and shows exactly what would change, without changing anything.
- **`terraform apply`:** makes the changes.
- **`terraform destroy`:** removes everything the files created, which matters in the cloud because running resources bill by the hour.
---
 
## Architecture
 
```
My local Windows PC (PowerShell, Azure CLI, Terraform)
        │
        │  az login authenticates; Terraform calls the Azure management API
        ▼
Azure resource group: rg-ad-richard
        │
        ├── VNet 10.0.0.0/16  →  Subnet 10.0.1.0/24
        ├── Static public IP (Standard SKU)
        ├── NSG (inbound RDP 3389) attached to the NIC
        ├── NIC with static private IP 10.0.1.4
        └── Windows Server 2022 VM (ad-richard)
                  │
                  │  Custom Script Extension runs PowerShell on the VM
                  ▼
        Install AD DS role → Install-ADDSForest → DNS installed → automatic reboot
                  │
                  ▼
        Domain controller for corp.richard.com (NetBIOS: CORP)
```
 
| Component | Role |
|---|---|
| **Azure CLI** | Logs me into Azure; Terraform reuses that login so no credentials live in the files |
| **Terraform** | Reads the `.tf` files and builds the resources |
| **Azure subscription** | The billing account and boundary that the resources live in |
| **Custom Script Extension** | Runs a PowerShell command inside the VM after it is created, to install and configure AD DS |
| **Project directory** | Holds the `.tf` files and Terraform's state file |
 
Key concepts I needed:
 
- **Provider:** the plugin that lets Terraform talk to a platform. This lab uses `hashicorp/azurerm`.
- **Resource:** one thing Terraform manages, such as a VM or a subnet. Resources can reference each other, which is how Terraform knows the creation order.
- **Variables and `terraform.tfvars`:** `variables.tf` declares inputs and `terraform.tfvars` supplies the values, so the same code can be reused with different names, passwords, and domains.
- **State file (`terraform.tfstate`):** Terraform's record of what it has built. It is how `destroy` knows what to remove. It also contains secrets in plain text.
- **DSRM password:** the Directory Services Restore Mode password, separate from the admin password, used only for AD recovery operations.
- **Quota vs. capacity:** a quota is how much of a resource family my subscription is allowed to use in a region. Capacity is whether Azure has any of that hardware free right now. They fail with different errors.
- **Hypervisor generation:** Azure VM images and sizes are Gen 1 or Gen 2, and the image and the size have to match.
---
 
## What I built
 
### 1. Installed the prerequisites
On Windows 10 I installed the Azure CLI and Terraform, signed up for an Azure subscription, and created a project folder:

```powershell
winget install -e --id Microsoft.AzureCLI
```
> To install Azure CLI, run the command above first.

```powershell
az --version
az login
az account show
```
> After you install Azure CLI, run these commands to confirm your CLI is up to date, to login to your Azure account, and to confirm you're using the correct subscription.

```powershell
winget install Hashicorp.Terraform
```
> To install Terraform, run the command above.

```powershell
terraform -version
```
> Run this to verify that Terraform is installed and up to date.
 
<img width="1920" height="439" alt="AutomateLabPrereqs2" src="https://github.com/user-attachments/assets/da3dd929-1337-482a-9d6c-146fd3fc81e5" />

### 2. Scaffolded the project
I originally used scaffold commands written for bash based on the guide I was following, which failed in Windows PowerShell (details in the problems section). The PowerShell version:
 
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\repos\az-ad-vm"
Set-Location "$HOME\repos\az-ad-vm"
New-Item -ItemType File main.tf, variables.tf, outputs.tf, terraform.tfvars
```
 
### 3. Wrote the Terraform configuration
Four files make up the project:
 
- **`main.tf`:** the provider, the resource group, VNet, subnet, public IP, NSG, NIC, the Windows VM, and the Custom Script Extension.
- **`variables.tf`:** inputs for the name, region, passwords (marked `sensitive`), domain name, and NetBIOS name.
- **`terraform.tfvars`:** my values: domain `corp.richard.com`, NetBIOS `CORP`, and my own passwords.
- **`outputs.tf`:** the public IP, the domain name, and the admin username.
The extension is the part that makes this a domain controller. It installs the AD DS role and creates a new forest with DNS in one command:
 
```hcl
resource "azurerm_virtual_machine_extension" "ad_setup" {
  name                 = "install-ad-ds"
  virtual_machine_id   = azurerm_windows_virtual_machine.main.id
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.10"

  settings = jsonencode({
    commandToExecute = "powershell -ExecutionPolicy Unrestricted -Command \"Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools; Import-Module ADDSDeployment; Install-ADDSForest -DomainName '${var.domain_name}' -DomainNetbiosName '${var.domain_netbios}' -ForestMode 'WinThreshold' -DomainMode 'WinThreshold' -InstallDns:$true -SafeModeAdministratorPassword (ConvertTo-SecureString '${var.dsrm_password}' -AsPlainText -Force) -Force:$true\""
  })

  tags = var.tags

  lifecycle {
    ignore_changes = [settings]
  }
}
```
 
Two lines in that file came from fixing errors, not from the original guide: the `lifecycle` block at the bottom, and a `size` change on the VM. Both are explained below.
 
I then ran:
 
```powershell
terraform init
terraform fmt
terraform validate
```
 
`validate` returned `Success! The configuration is valid.`
 
<img width="1920" height="418" alt="AutomateLabCheckValidity" src="https://github.com/user-attachments/assets/15ad2af2-af32-40b9-a4c2-a1990187ae37" />

### 4. Deployed, and worked through the errors
`terraform plan` and `terraform apply` did not work on the first try. Six separate problems came up before the deployment finished. Each one is covered in the next section.
 
<img width="1921" height="1049" alt="AutomateLabTerraformPlan3andError" src="https://github.com/user-attachments/assets/f43d1f94-b37e-4fbe-bd65-c76da8980f60" />

> This was the first error I ran into involving the VM size I originally requested being unavailable.

### 5. Connected and verified the domain controller
I got the IP with `terraform output public_ip`, opened Remote Desktop Connection, and signed in as `CORP\adadmin` (the domain prefix, since the VM reboots into domain authentication after promotion). Then I ran the verification commands in PowerShell as Administrator:
 
```powershell
Get-Service NTDS | Select-Object Name, Status
Get-ADDomain
Get-ADDomainController -Filter *
Resolve-DnsName corp.richard.com
```
<img width="1919" height="105" alt="AutomateLabGetIP" src="https://github.com/user-attachments/assets/c9d39347-0ec9-44dc-90f4-954db6445e7e" />
<img width="887" height="551" alt="AutomateLabRDP2" src="https://github.com/user-attachments/assets/7d664e2e-c5ed-40a8-8c5e-6ada18fa6b1e" />
 
Results:
 
- `NTDS` was `Running`.
- `Get-ADDomain` showed `DNSRoot` and `Forest` as `corp.richard.com`, `NetBIOSName` as `CORP`, and the domain mode as `Windows2016Domain` (what `WinThreshold` maps to).
- `Get-ADDomainController` showed one DC, `ad-richard`, at `10.0.1.4`, running Windows Server 2022 Datacenter, as a global catalog and holding all five FSMO roles.
- `Resolve-DnsName` returned an A record for `corp.richard.com` pointing at `10.0.1.4`, so the DNS role installed with AD DS.
<img width="1907" height="1046" alt="AutomateLabVerification1" src="https://github.com/user-attachments/assets/3801dd19-817e-47d9-a071-ec50b312bcfe" />
<img width="1914" height="72" alt="AutomateLabVerification2Fix" src="https://github.com/user-attachments/assets/05014ee4-51fd-4695-a49e-2240299c9ad2" />
 
---
 
## The errors, one at a time
 
### PowerShell rejected the scaffold command
```
The token '&&' is not a valid statement separator in this version.
```
The guide I used was written for bash. Windows PowerShell does not support `&&`, and `touch` does not exist there. Because the line failed to parse, nothing ran. I replaced `mkdir -p`, `cd`, and `touch` with `New-Item` and `Set-Location`. The same issue applies to `\` line continuations in later commands, which PowerShell writes as a backtick or avoids by using one line.
 
### `terraform plan` failed with a resource provider error
```
Error: Encountered an error whilst ensuring Resource Providers are registered.
... lookup management.azure.com: no such host
```
The big block of text is about resource provider registration, but the useful line is the last one: `no such host`. That is a **DNS failure on my own PC**, not a permissions problem. I confirmed it with three checks, and a temporary DNS problem turned out to be the cause:
 
```powershell
Resolve-DnsName management.azure.com
Test-NetConnection management.azure.com -Port 443
az group list
```
 
All three succeeded when I re-ran them, and `terraform plan` then worked.
 
### `SkuNotAvailable` (409 Conflict)
```
Following SKUs have failed for Capacity Restrictions: Standard_D2s_v3
```
Azure had no capacity for that size in `eastus` at that moment. This is not a mistake in the files. The earlier resources (resource group, network, NIC) had already been created, so nothing needed cleanup. The fix was to choose a different VM size. I originally chose to switch to "Standard_D4alds_v7."
 
### `BadRequest`: size cannot boot Hypervisor Generation 1 (400)
```
The selected VM size 'Standard_D4alds_v7' cannot boot Hypervisor Generation '1'.
```
<img width="1919" height="253" alt="AutomateLabTerraformApplyFixError" src="https://github.com/user-attachments/assets/06b000da-29df-4aea-8adb-db061d3bbb2d" />

The Windows Server 2022 image in the guide is Gen 1, and newer sizes like `D4alds_v7` only boot Gen 2 images. The fix was to pick a size that supports the image (or use the `2022-datacenter-g2` image). It also showed me that a bigger, newer size is not automatically a better choice, since that one had double the vCPUs I needed. I opted to switch to "Standard_D2s_v5."

 
### "Family vCPUs are insufficient": a quota limit, not capacity
The size I tried next (Standard_D2s_v5) failed with a quota message. Quota is per subscription, per region, and per VM family, and a new subscription often has zero for many families. I checked what I was actually allowed to use:
 
```powershell
az vm list-usage --location eastus --output table
```
 
The table showed `Limit = 0` for the v5 families I had tried and `Limit = 10` for others, with 4 of 10 total regional vCPUs already used by my other labs. I picked **`Standard_D2s_v4`**: a 2 vCPU, 8 GB size from a family with quota, which supports Premium disks and boots the image. I then changed the size in the main.tf file using CodeVS.
 
```hcl
size = "Standard_D2s_v4"
```
<img width="1919" height="1048" alt="AutomateLabTerraformPlanSizeRealFix" src="https://github.com/user-attachments/assets/69d45881-2b57-4e34-90c1-5c082b29c899" />
 
### Extension error: `unimplemented polling status "Unknown"`
```
Error: creating/updating Extension ... polling after CreateOrUpdate:
internal-error: unimplemented polling status "Unknown"
```
<img width="613" height="106" alt="AutomateLabTerraformApplyFixFinalError22" src="https://github.com/user-attachments/assets/89012462-4848-46f9-ab1f-10f53b3e338a" />

This one is a provider quirk. The extension promotes the server to a domain controller, which reboots the VM partway through. While Terraform polls, Azure reports `Unknown`, which the provider does not handle, so Terraform gives up even though the install keeps running. The VM was created and the install was probably fine, so I did **not** re-run `apply`, since that could re-run the install on a server that was already a domain controller. I asked Azure what really happened:
 
```powershell
az vm extension show --resource-group rg-ad-richard --vm-name vm-ad-richard --name install-ad-ds --query "provisioningState" --output tsv
```
 
It returned `Succeeded`. Then `terraform plan` wanted to **create** the extension again, because Terraform never saved it to state. I imported the existing one instead:
 
```powershell
terraform import azurerm_virtual_machine_extension.ad_setup "/subscriptions/<subscription-id>/resourceGroups/rg-ad-richard/providers/Microsoft.Compute/virtualMachines/vm-ad-richard/extensions/install-ad-ds"
```
 
`Import successful!`

<img width="1920" height="815" alt="AutomateLabTerraformApplyFixFinalFinal" src="https://github.com/user-attachments/assets/e12bb066-1a54-4af4-a00d-7eb76369a0f4" />
 
### Two leftover diffs after the import
After importing, `terraform plan` still showed two changes:
<img width="1920" height="517" alt="AutomateLabVMAgentPlatform" src="https://github.com/user-attachments/assets/2668526d-b4ad-4b6a-9bed-6ce3f2a84068" />

1. **The VM:** `vm_agent_platform_updates_enabled = true -> false`. Azure had turned this setting on by default and my config did not mention it. I added `vm_agent_platform_updates_enabled = true` to the VM resource to match reality.
<img width="1919" height="1049" alt="AutomateLabTerraformVMCleanChange" src="https://github.com/user-attachments/assets/1feb0fc1-fac0-4bc3-a061-857465fd61a1" />

2. **The extension:** a change to the sensitive `settings`. Terraform does not have the sensitive settings in state after an import, so it shows a difference that is not real, and applying it could re-run the install script on a domain controller that was already built. Since the AD DS install is a one-time action, I told Terraform to stop watching those settings:
```hcl
lifecycle {
  ignore_changes = [settings]
}
```

My first attempt at this failed validation with `Unsupported attribute ... "settings"`. I had pasted the `lifecycle` block into the VM resource instead of the extension resource (both have a `tags = var.tags` line, and the VM's comes first), and a VM has no `settings` attribute. After moving it to the extension, `terraform validate` passed and `terraform plan` reported no changes.
<img width="1920" height="1049" alt="AutomateLabTerraformApplyFixFinal4Error" src="https://github.com/user-attachments/assets/92afe987-6717-4f11-8c32-759b4fdba1bf" />
<img width="435" height="124" alt="AutomateLabUnsupportedAttributeError" src="https://github.com/user-attachments/assets/542b2220-21fc-4f12-a07d-0bd5f9359e70" />
> I mistakenly placed the `lifecycle` block in the VM resource initially. This gave me the error above.
<img width="1919" height="1048" alt="AutomateLabTerraformApplyFixFinal4ErrorReal" src="https://github.com/user-attachments/assets/8362d893-e6bb-476c-ba81-cd53405d305b" />
<img width="1920" height="227" alt="AutomateLabUnsupportedAttributeErrorFixedReal" src="https://github.com/user-attachments/assets/14e58009-1b49-4109-8ad0-0b6c545eb1b7" />

> I moved the `lifecycle` block into the correct extension resource resolving the issue.
 
---
 
## Problems and fixes
 
| What broke | Cause | Fix |
|---|---|---|
| `&&` and `touch` errors in the scaffold command | The guide I followed was written for bash; Windows PowerShell does not support them | `New-Item` and `Set-Location` instead |
| `plan` failed with a resource provider registration error | Temporary DNS failure on my PC (`lookup management.azure.com: no such host`) | Confirmed with `Resolve-DnsName`, `Test-NetConnection`, and `az group list`, then re-ran `plan` |
| `SkuNotAvailable` for `Standard_D2s_v3` (409) | No capacity for that size in `eastus` | Chose a different VM size |
| `cannot boot Hypervisor Generation '1'` (400) | Gen 2 only size paired with a Gen 1 image | Used a size that supports the image |
| "Family vCPUs are insufficient" | Per-family quota of 0 on my subscription | Read `az vm list-usage` and picked `Standard_D2s_v4`, which had quota |
| `unimplemented polling status "Unknown"` | AD promotion reboots the VM during extension polling and the provider cannot handle `Unknown` | Checked the real status with `az vm extension show` instead of re-applying |
| `plan` wanted to create an extension that already existed | The failed run never saved it to state | `terraform import` |
| `plan` still showed changes after the import | Azure default on the VM, and sensitive `settings` not in state | Added `vm_agent_platform_updates_enabled = true` and `ignore_changes = [settings]` |
| `Unsupported attribute "settings"` on `validate` | `lifecycle` block pasted into the VM resource instead of the extension | Moved it into the extension resource |
 
---
 
## Key takeaways
 
1. **Read the last line of a long error.** The provider registration block was boilerplate, and the real cause (`no such host`) was a single line at the bottom.
2. **Quota and capacity are different problems.** "SkuNotAvailable" means Azure is out of hardware right now. "Family vCPUs are insufficient" means my subscription is not allowed that much. `az vm list-usage` answered the second one directly.
3. **When apply fails partway, check reality before re-running.** `az vm extension show` told me the install had succeeded. Re-running `apply` blindly could have re-run a domain promotion on an existing domain controller.
4. **Terraform state is the source of truth for Terraform, not for Azure.** When they disagree, I fix the state (`import`, `ignore_changes`) rather than rebuild the resource.
5. **Newer and bigger is not automatically compatible.** The image generation has to match the VM size.
6. **Know which resource a block belongs to.** The `lifecycle` mistake passed formatting but failed validation, and `validate` named the resource that was actually wrong.
If I did this again, I would run `az vm list-usage` and `az vm list-skus` before the first `apply` to pick a size I know I have quota and capacity for, and I would add the `lifecycle` block and the `vm_agent_platform_updates_enabled` setting from the start.
 
---
 
## Security notes
 
This is a lab configuration, and a few choices in it would not be acceptable in production:
 
- **RDP is open to the whole internet.** The NSG rule uses `source_address_prefix = "*"`. A domain controller should never be exposed like this. Restrict it to my own public IP or use Azure Bastion.
- **Secrets are in plain text locally.** `terraform.tfvars` and `terraform.tfstate` contain the admin and DSRM passwords, and the AutoLogon unattend content also embeds the admin password.
- **The extension script includes the DSRM password** in its command line, which is a known weakness of passing secrets this way.
---
 
## Verification
 
| Check | How I verified it |
|---|---|
| Tools installed and logged in | `az account show` and `terraform -version` both returned results |
| Configuration valid | `terraform validate` returned `Success! The configuration is valid.` |
| VM and network deployed | `terraform plan` refreshed all resources with no errors |
| AD DS install finished | `az vm extension show` returned `Succeeded` |
| Terraform matches Azure | After import and `lifecycle`, `terraform plan` reported no changes |
| AD service running | `Get-Service NTDS` returned `Running` |
| Domain exists | `Get-ADDomain` returned `corp.richard.com` with NetBIOS name `CORP` |
| DC is healthy | `Get-ADDomainController` returned `ad-richard` at `10.0.1.4`, global catalog, not read-only |
| DNS works | `Resolve-DnsName corp.richard.com` returned an A record for `10.0.1.4` |
 
---
 
## Teardown
 
Running resources bill by the hour, so when I finish with the lab:
 
```powershell
terraform destroy
```
 
It removes `rg-ad-richard` and everything in it. My other resource groups are not affected, because Terraform only manages what is in its own state.
 
---
 
## Next steps
 
- Restrict the RDP rule to my own IP, or replace it with Azure Bastion
- Move the passwords out of `terraform.tfvars` into environment variables (`TF_VAR_...`) or Azure Key Vault
- Extend the deployment to create OUs, users, and a second VM joined to the domain
- Add the Splunk forwarder and the Nessus scan from my other labs on top of this automated environment
---
 
**Status: complete.** Terraform deployed the network and VM → six deployment errors diagnosed and fixed → extension imported into state → domain controller verified (NTDS running, domain `corp.richard.com`, DNS resolving).
