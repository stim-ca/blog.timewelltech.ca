---
title: "VSphere 8.0 Continued"
date: 2023-11-13T11:11:03-08:00
draft: false
---

## Prerequisites

- Configured ESXi Host
- Basic PowerShell knowledge
- PowerCli Installed
- Windows Server 2022 Evaluation ISO
  - Download [link](https://www.microsoft.com/en-us/evalcenter/download-windows-server-2022)
- VMWare's VMRC Installed
  - Download [link](https://customerconnect.vmware.com/en/downloads/details?downloadGroup=VMRC1201&productId=974)


## Connecting to ESXi host

```powershell
$eSXi01Root = Get-Credential root

Connect-VIServer -Server '10.0.1.50' -Credential $eSXi01Root
```

## Copying ISO to Datastore

### Get-PSDrive
To find out the datastore path we're going to use `Get-PSDrive`.

```PowerShell
Get-PSDrive
```
```Shell
Name           Used (GB)     Free (GB) Provider      Root
----           ---------     --------- --------      ---
Alias                                  Alias
C                 418.38         46.61 FileSystem    C:\  
Cert                                   Certificate   \
D                   8.85         50.89 FileSystem    D:\
Env                                    Environment
Function                               Function
HKCU                                   Registry      HKEY_CURRENT_USER
HKLM                                   Registry      HKEY_LOCAL_MACHINE
Variable                               Variable
vi                                     VimInventory  \LastConnectedVCenterServer
vis                                    VimInventory  \
vmstore                                VimDatastore  \LastConnectedVCenterServer
vmstores                               VimDatastore  \
WSMan                                  WSMan
```

### Make a Directory to Store the ISOs 

We're going to make a new folder named `iso` on the datastore `data` that we setup in the last post.
You can use `tab` completion to find your correct path.

```powershell
mkdir vmstores:\10.0.1.50@443\ha-datacenter\data\iso
```

### Transferring the ISO to the ESXi Host

The command we're going to use is `Copy-DatastoreItem`. It requires an `Item` source and a `Destination` path.

```PowerShell
Copy-DatastoreItem -Item 'C:\Temp\ISO\Windows Server 2022.iso' -Destination 'vmstores:\10.0.1.50@443\ha-datacenter\data\iso\Windows Server 2022.iso' -Force
```

![Datastore Progress Bar]()

The transfer time will depend on your network and disk speeds.

## Provisioning a Virtual Machine

### Gathering Required Information

`GuestID`; Information can be found on the vSphere API documentation [link.](https://developer.vmware.com/apis/1355/vsphere/)

- Click `All Enumerations`
- In the `-- Quick Index --` bar search for `VirtualMachineGuestOsIdentifier` and a list will appear.


`NetworkName`; We can get this the `Get-VirtualNetwork` command.

### New-VM
We're going to use a PowerShell splat to keep the code easier to read. To find all available properties you can use the `Get-Help` command or the [VMWare docs](https://developer.vmware.com/docs/powercli/latest/vmware.vimautomation.core/commands/new-vm/).

```PowerShell
Get-help New-VM -Full
```

```PowerShell
$NewVM_Properties = @{
    CD             = $true
    CoresPerSocket = 4
    Datastore      = 'data'
    DiskGB         = 50
    GuestId        = 'windows2022srvNext_64Guest'
    MemoryGB       = 16
    Name           = 'TTE-DC-01'
    NetworkName    = 'VM Network'
    Notes          = 'Primary Domain Controller for AD.TimewellTech.ca'
    NumCpu         = 4
}

New-VM @NewVM_Properties
```
```Shell
Name                 PowerState Num CPUs MemoryGB
----                 ---------- -------- --------
TTE-DC-01            PoweredOff 4        16.000
```

### Add a Secondary HDD to the VM for User Data

```powershell
$NewHardDisk_Properties = @{
    CapacityGB = '40'
    Datastore = 'data'
    Persistence = 'Persistent'
    VM = "$($NewVM_Properties.Name)" 
    # I can't remember what this is called, but it allows us to get the VMName
}

New-HardDisk @NewHardDisk_Properties
```
```Shell
CapacityGB      Persistence                                                    Filename
----------      -----------                                                    --------
40.000          Persistent                            [data] TTE-DC-01/TTE-DC-01_1.vmdk
```

### Connect ISO to New VM's CD Drive

Use Dot notation to get the VM Name.

[data] is how you have to reference the datastore I found this out by looking at the GUI

![GUI Data Path]()

```powershell
Get-CDDrive -VM $NewVM_Properties.Name | 
    Set-CDDrive -IsoPath '[data] iso/Windows Server 2022.iso' -StartConnected $true
```
```Shell
Confirm
Are you sure you want to perform this action?
Performing the operation "Setting IsoPath: [data] iso/Windows Server 2022.iso, StartConnected: True, NoMedia: False." on    
target "CD/DVD drive 1".
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "Y"):
```

Press `Enter` to continue.

```Shell
IsoPath              HostDevice                             RemoteDevice
-------              ----------                             ------------
[data] iso/Window...
```

### Start the Virtual Machine & Open a Virtual Console

```Powershell
Start-VM -VM $newVMArguments.Name
```
```Shell
Name                 PowerState Num CPUs MemoryGB
----                 ---------- -------- --------
TTE-DC-01            PoweredOn  4        16.000
```

You must have VMRC installed on your local machine, otherwise head to your webgui.

```PowerShell
Open-VMConsoleWindow -VM $newVMArguments.Name
```



