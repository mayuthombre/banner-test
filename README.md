

# agl-windows-vm

## Terraform 0.12+ - Create an AGL Windows VM

## Version Compatibility
| Module version | Terraform version | AzureRM version |
| --- | --- | --- |
| >= 8.x.x | 0.12.x & 0.13.x | >= 2.0 |
| >= 9.x.x | >= 1.1.x | >= 2.0 |
| >= 10.x.x | >= 1.1.x | >= 3.0 |
| >= 11.x.x | >= 1.2.x | >= 3.0 |

* This Terraform module is for terraform 0.12 and later ONLY - if required refer to the terraform 0.11 version of this module [terraform-azure-rm-windows-vm/README.md](https://github.com/AGLEnergy/terraform-azurerm-windows-vm/blob/master/README.md)

* This module requires your TFE workspace to be built using at least version 3.2.1 to allow the use of the DSC functionKey.

This Terraform module creates an AGL standard Virtual Machine in Azure with AGL Security and Management agents installed. The virtual machine is joined to the AGL Domain in the OU specified by the `ou_tags` variable.

The module does not create nor expose a network security group. This would need to be defined separately as additional security rules on subnets in the deployed VM.

The module includes the following features - all of which are managed by Windows/Azure Desired State Configuration (DSC) to maintain the configuration standard:
* VM is joined to the AGL Active Directory domain in the OU specified
* Security and Operations agents installed
  * Qualys agent (security - vulnerability scanning)
  * CrowdStrike agent (security - endpoint protection)
  * Flexera agent (software / license management agent)
  * Splunk Universal Forwarder
  * Azure Monitor Agent (AMA) ( optional - via the `log_to_loganalytics` flag - default : `false` )
* IIS installed ( via the `install_iis` flag - default : `false` )
* Ability to install any Windows Feature via the `windows_feature_list` variable, eg `optional_windows_features = "RSAT-ADDS,Telnet-Client"`
* Note: when this module is used to deploy multiple VMs ( using the `vm_count` variable ) all VM's will necessarily be identically configured (eg VM size, number and size discs)
* Initialize Data Disks - This variable is used to prevent the CustomScript extension from formatting attached data discs. This is commonly used by the [agl-sql-vm module](https://github.com/AGLEnergy/terraform-azurerm-agl-sql-vm) because the SQL Extension Configuration performs the disc configuration during the SQL / Azure integration phase, the disc must remain attached but unformatted in this scenario.
* Note : Due to AGL security requirements the local administrator account ( specified by the `admin_username` parameter ) is immediately locked by Active Directory Group Policy as part of the domain join process. The `admin_password` therefore is only used once and does not need to be randomly generated or stored. It is ONLY used to diagnose domain join failures.

## AGL Virtual Machine Requirements
* Consult the comprehensive [IAAS Operations Guide](https://aglenergy.atlassian.net/wiki/spaces/CPE/pages/741901754/IaaS+Standard+Templates+-+Operations+Guide) has a wealth of deployment information including detailed pre-requisite steps.
* As per AGL security standards all VMs deployed must be domain joined and have required security agents deployed.
* Also as per AGL security standards all RDP access to virtual machines must be performed via CyberArk - check [here](https://agl-dwp.onbmc.com/dwp/app/#/knowledge/KBA00025206/rkm) for details on how to get your own team safe and how to use CyberArk to RDP into VMs

### Specific Details
* Pre-Requisite - Each team or workload will require **THEIR OWN** application or team specific OU to be created **BEFORE** deploying any Virtual Machines. Cloud Services will pre-load your CyberArk access group (eg CyberArk-Grp-xxx) into the access group as part of the onboarding process. This will allow login by your team from CyberArk into to any guest VMs that area created in the OU without any further manual tasks.
* The `ou_tags` variable is a **mandatory** 2 element map variable that specifies the Active Directory Organisational Unit that the Virtual Machine computer object will be created in. This must be **your** OU created as part of onboarding.
* If the COMPUTER object for the the Guest you are attempting to create exists elsewhere in Active Directory then the **Domain Join Will Fail**. This is intentional to avoid over-writing existing COMPUTER objects. If this occurs either request a new, valid hostname or if the computer object is invlid then raise a SR to Wintel to cleanup the stale object in AD.
* The `ScheduleType` tag is also mandatory as per the [tagging standards guide](https://aglenergy.atlassian.net/wiki/spaces/CPE/pages/702779781/Cloud+Tagging+Standards+-+User+Guide#Virtual-Machines-Tags) and must contain one of the following [values](https://aglenergy.atlassian.net/wiki/spaces/CPE/pages/702779781/Cloud+Tagging+Standards+-+User+Guide#ScheduleType-Values-Table): `AlwaysOn_24_7`,`Non_Prod_Std`,`WeekendShutdown`,`Non_Prod_Eco`,`Self_Serve` or `Custom`.

## Example Usage

```hcl
module "windows-vm" {
  source                        = "app.terraform.io/AGL/agl-windows-vm/azurerm"
  resource_group_name           = local.resource_group
  location                      = local.location
  size                          = "Standard_DS3_v2"
  hostname_prefix               = "AZSAWLAB"
  hostname_suffix_start_range   = "0001"
  vm_count                      = "1"
  network_interfaces = [
    {
      subnet_id                     = data.azurerm_subnet.subnet.id,
      private_ip_address_allocation = "dynamic",
      private_ip_address            = null,
      enable_accelerated_networking = false
    }
  ]
  admin_username                = "azureadmin"
  admin_password                = "ChangeMe!$"
  license_type                  =  "Windows_Server"
  availability_set_id           = "xxxx-xxxx-xx-xxxxxx"
  tags                          = var.tags
  ou_tags                       = var.ou_tags
  dsc_FunctionKey               = var.dsc_FunctionKey
  log_to_loganalytics           = true
  loganalytics_workspace_id     = var.my_la_id
  loganalytics_workspace_key    = var.my_la_key
  disk_alloc_unit_size          = 65536
  identity                      = [{type = "SystemAssigned"}]
  data_disks = [
    [
      {
        disk_size_gb              = 256
        storage_account_type      = "Standard_LRS"
        caching                   = "ReadOnly"
        create_option             = "Empty"
        source_resource_id        = ""
        write_accelerator_enabled = false
      },
      {
        disk_size_gb              = 120
        storage_account_type      = "Premium_LRS"
        caching                   = "None"
        create_option             = "Empty"
        source_resource_id        = ""
        write_accelerator_enabled = false
      }
    ],
    [
      {
        disk_size_gb              = 32
        storage_account_type      = "Standard_LRS"
        caching                   = "ReadOnly"
        create_option             = "Empty"
        source_resource_id        = ""
        write_accelerator_enabled = false
      },
      {
        disk_size_gb              = 64
        storage_account_type      = "Premium_LRS"
        caching                   = "None"
        create_option             = "Empty"
        source_resource_id        = ""
        write_accelerator_enabled = false
      }
    ]
  ]
}

module "windows-spot-vm" {
  source                        = "app.terraform.io/AGL/agl-windows-vm/azurerm"
  resource_group_name           = local.resource_group
  location                      = local.location
  size                          = "Standard_DS3_v2"
  hostname_prefix               = "AZSAWLAB"
  hostname_suffix_start_range   = "0001"
  vm_count                      = "1"
  network_interfaces = [
    {
      subnet_id                     = data.azurerm_subnet.subnet.id,
      private_ip_address_allocation = "dynamic",
      private_ip_address            = null,
      enable_accelerated_networking = false
    }
  ]
  admin_username                = "azureadmin"
  admin_password                = "ChangeMe!$"
  license_type                  =  "Windows_Server"
  availability_set_id           = "xxxx-xxxx-xx-xxxxxx"
  tags                          = var.tags
  ou_tags                       = var.ou_tags
  dsc_FunctionKey               = var.dsc_FunctionKey
  log_to_loganalytics           = true
  loganalytics_workspace_id     = var.my_la_id
  loganalytics_workspace_key    = var.my_la_key
  disk_alloc_unit_size          = 65536
  identity                      = [{type = "SystemAssigned"}]
  spot_configurtaion = {
    priority        = "Spot"
    eviction_policy = "Deallocate"
    max_bid_price   = -1
  }
}

```

The default image for this module will provision a default AGL Windows 2019 Datacenter virtual machine, to the latest version available from the Microsoft Azure Image Library

## Accelerated Networking and Availability Sets

See [Create a Windows virtual machine with Accelerated Networking using Azure PowerShell](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-powershell)
for considerations when Availability Sets and Accelerated Networking are both present on multiple VM's.

## Automation Account Hybrid Runbook Workers

(optional) The VM can be added to an existing Automation Account Hybrid Worker Group by specifying the following two settings:

- hybrid_runbook_worker_group_automation_account_name
- hybrid_runbook_worker_group_name

## Accelerated Networking and VM Sizes

Not all VM sizes support accelerated networking. See [Supported VM instances](https://docs.microsoft.com/en-us/azure/virtual-network/create-vm-accelerated-networking-powershell#supported-vm-instances)

## Disabling backup protection on the VM/Destroying the VM when the VM backup is protected

Disabling backup protection by changing the enable_backup to false will error out as shown below. This is due to 'soft delete' protection in the vault.
```
|Error: Error waiting for the Azure Backup Protected VM "VM;iaasvmcontainerv2;workspace-backup-rsv-testing-slesvm;AZSALLAB0002" to be false (Resource Group "management") to provision: timeout while waiting for state to become 'NotFound' (last state: 'Found', timeout: 30m0s|
```

[SoftDelete](https://docs.microsoft.com/en-us/azure/backup/backup-azure-security-feature-cloud)

Workaround is to manually remove Terraform State related with azurerm_backup_protected_vm resource address.
[details](https://github.com/terraform-providers/terraform-provider-azurerm/issues/4276)

### Azure Hybrid Runbook Worker
An Azure Hybrid Runbook Worker is an optional component of the Azure Monitoring Agent used to execute Runbooks in an environment. To deploy an Azure Hybrid Runbook worker on a guest VM then follow these steps:
* when deploying the guest, ensure `log_to_loganalytics` is true and that you have specified a valid Log Analytics workspace ID and key (mandatory as this installs the monitoring agent). Each subscription has a Log Analytics workspace created in the `management` resource group for this purpose.
* follow steps 3 through to 7 of [this](https://docs.microsoft.com/en-us/azure/automation/automation-windows-hrw-install#manual-deployment) guide, this verifies connectivity to Log Analytics and details how to configure the worker. You need to specify the Url and Key of your Automation Account to connect the elements together.

## Restoring data disk of a VM
- Manually restore the data disks referencing the [Azure portal link](https://docs.microsoft.com/en-us/azure/backup/backup-azure-arm-restore-vms#restore-disks) in an other Resource group
- Use the data module `azurerm_managed_disk` to get the details restored disk, [terraform reference](https://www.terraform.io/docs/providers/azurerm/d/managed_disk.html)
- Change the create_option to Restore and pass the source_resource_id argument which should be the id of the restored disk to the data\_disks JSON object variable as shown.<br> Example:<br>
```hcl
data "azurerm_managed_disk" "restored_data_disk" {
  name                = "testrestorevmvm-datadisk-000-20200403-021020"
  resource_group_name = azurerm_resource_group.main.name
}

data_disks = [
    [
      {
        disk_size_gb              = 256
        storage_account_type      = "Standard_LRS"
        caching                   = "ReadOnly"
        create_option             = "Restore"
        source_resource_id        = data.azurerm_managed_disk.restored_data_disk.source_resource_id
        write_accelerator_enabled = false
      },
      {
        disk_size_gb              = 120
        storage_account_type      = "Premium_LRS"
        caching                   = "None"
        create_option             = "Empty"
        source_resource_id        = ""
        write_accelerator_enabled = false
      }
    ]
  ]
```
- Execute terraform init, plan and apply. This operation will delete the corrupted disk and replaces with the restored disk
---
**NOTE**

Update the `create_option` and `source_resource_id` of the disk/s corresponding to the VM block/s for which restored disk has to be replaced.

---

## Restoring OS disk of a VM

Check the Quick reference guide for OS disk restore [Details:](https://aglenergy.atlassian.net/wiki/spaces/AAA/pages/833004016/OS+Disk+Restore+-+How+to+Guide)

## Expanding OS drive of a VM
The default OS drive is often 127 GB (some images have smaller OS disk sizes by default), we can expand the OS disk during the VM build by setting the variable `enable_os_disk_expansion` to true. [Details:](https://docs.microsoft.com/en-us/azure/virtual-machines/windows/expand-os-disk#expand-the-volume-within-the-os)

## Managed Identity Parameter

The managed_identity parameter, if defined, is passed as a list with a single object entry. This list can also be empty, if you wish to formally declare there is no managed identity for this VM instance.

The fields of the single object are

|Name | Type | Description |
|-----|------|-------------|
|type | string | Valid possible values are "SystemAssigned", "UserAssigned" and "SystemAssigned, UserAssigned" |
|identity_ids | list(string) | Default `[]`. A list of User Managed Identity ID's which should be assigned to the Windows Virtual Machine |

Note - if you are using a `type` of "SystemAssigned" only, there is no need to declare the `identity_ids` section.

The `managed_identity` output from the module is a list of identity objects of the following form

```json
[
  {
    "principal_id": "bbabb7ed-6272-41aa-92eb-258aa39b8556",
    "tenant_id": "123913b9-915d-4d67-aaf9-ce327e8fc59f",
    "type": "SystemAssigned"
  }
]
```
## Disc size and type changes
### Disc Size Changes
Terraform can directly increase the size of an OS or data disk simply by changing the value in the terraform code. Terraform will shutdown the guest, perform the size increase and start the guest. This can take upto 4-5 minutes depending on size increase. In testing it was noted that the C: partition was automatically extended to consume the new space.

### Disc Size Changes
Terraform CANNOT change the disc type without destroying the disc. To make this change:
1. Stop VM
1. Change disc type in the console
1. start VM
1. change TF code to match
1. terraform plan to validate - no changes should be presented

## Support for marketplace images

To use a marketplace image set the `marketplace_image` variable as `true` and pass the respective image details. Subscription needs to be enabled for programmatic deployment of the desired marketplace image.

## Support for Windows Server Core OS

To use a supported Windows Server core simply add `image_sku  = "2019-datacenter-core-g2"` to the VM Module parameters when building.

## Support for Windows 2012 R2 DataCenter

To use Windows Server 2012R2 simply add:
```
  image_publisher             = "MicrosoftWindowsServer"
  image_offer                 = "WindowsServer"
  image_sku                   = "2012-R2-Datacenter"
```

## Windows Feature Support
Use the `optional_windows_features` to specify a comma separated list of Windows Features ( as provided by the PowerShell `Get-WindowsFeature` cmdlet ) to be installed during build, eg:
```
  optional_windows_features = "Failover-Clustering,MSMQ"
```

## Windows Licensing (AHUB) Support and validation

To use the Windows hybrid benefit for licensing cost purpose, specify a value for license_type as Windows_Server, Windows_Client or None based on use case. Make sure to update AzureRM version to 2.29.0 or higher in order to use this feature (in earlier AzureRM release, resource (VM) get recreated) if license_type value is modified and in terraform lifecycle changes are not ignored for license_type. In the situation where the Cost Optimization team have updated the license_type outside of terraform for a VM, and you see the `license_type` value getting reverted, you can now maintain this in code .eg:
```
  license_type = "Windows_Server"
```
Please refer the [Confluence link](https://aglenergy.atlassian.net/wiki/spaces/CTOM/pages/2343536024/Azure+Hybrid+Benefit+Guidelines) to examin if VM is suitable or not for AHUB setting.

Possible value(s) validation is added in module code for license_type variable, to ensure that correct value is specified in code for license_Type. Make sure to update terraform version to tf13 or higher in order to use this custom validation feature as it was introduced from terraform 0.13.

## EnvOU validation

Possible value(s) validation is added in module code for environment ou (EnvOU) in ou_tags variable, to ensure that correct value is specified in code for EnvOU. Make sure to update terraform version to tf13 or higher in order to use this custom validation feature as it was introduced from terraform 0.13.

## Azure Spot VM

Using Azure Spot VMS allow to take advantage of unused capacity at a significant cost savings At any point in time when azure needs the capacity bacj, the azure infrasturcture willevict Azure Spot VMs. https://learn.microsoft.com/en-us/azure/virtual-machines/spot-vms. To opt azure Spot VM can be used agl-windows-vm module. introduced new attributes called `spot_configuration`. Set Priority flag as `Spot` to opt for Azure Spot vm and mention the required eviction_policy. By default priority will be set as `Regular`.

 To Conifgure the Spot stateful VM please use the terraform module https://github.com/AGLEnergy/terraform-spotinst-agl-stateful-node which uses Spot by NetApp to create the spot vm.
