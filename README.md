# Hyper-V Windows VM Migration to OCI 


## VM Requirements

- Hyper-V VMs in **GEN1 (BIOS) (preferred) or GEN2 (UEFI)**
- Boot Volume size: **<400 GB**
- Additional block volumes **cannot be imported**, the data must be imported separately.
- The VM **must not be encrypted**.
- **Remove ISCSI controllers** (if any).
- The boot process **must not require additional data volumes** to be present for a successful boot.
- The network configuration **must not hardcode the MAC address** for the network interface.



## How to Prepare the Hyper-V VM Before Migration

### 1. Install VirtIO Drivers and Restart the VM.

[Download VirtIO drivers](./V1043005-01.zip)
### 2. Configure Network and Remote Access

Run the following commands from **PowerShell with Admin rights**:

#### Enable DHCP on ETHERNET NIC:
```powershell
Get-NetAdapter | Where-Object { $_.Status -eq "Up" } | Set-NetIPInterface -Dhcp Enabled
Get-NetAdapter | Where-Object { $_.Status -eq "Up" } | Set-DnsClientServerAddress -ResetServerAddresses
Get-NetAdapter | Where-Object { $_.Status -eq "Up" } | ForEach-Object {Remove-NetRoute -InterfaceIndex $_.ifIndex -DestinationPrefix "0.0.0.0/0" -Confirm:$false}
```

#### Enable RDP for the Administrator account:  

```powershell
Set-ItemProperty -Path "HKLM:\System\CurrentControlSet\Control\Terminal Server" -Name "fDenyTSConnections" -Value 0
Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
Add-LocalGroupMember -Group "Remote Desktop Users" -Member "Administrator"
```
*(Replace "Administrator" with the correct Admin name if needed)*

#### Disable "Require computers to use Network Level Authentication to connect"

```powershell
Set-ItemProperty -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp" -Name "UserAuthentication" -Value 0
```

#### Create a Windows Firewall rule to allow inbound RDP (TCP 3389):
```powershell
New-NetFirewallRule -DisplayName "Allow RDP from Any" -Direction Inbound -Protocol TCP -LocalPort 3389 -Action Allow -RemoteAddress Any -Profile Any
```

#### Disable Windows Firewall (or any local Firewall):
```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled false
```

#### Determine whether the current Windows license type is a volume license

```powershell
Get-CimInstance -ClassName SoftwareLicensingProduct | where {$_.PartialProductKey} | select ProductKeyChannel
```

*If the license is not a volume license, after you import the image, you will update the license type.*

## Export the VM

### 1. Convert the VHDX to QCOW2 from a third-party instance (e.g., the Hyper-V server)

#### Install Pacman on Windows:
[Download Pacman installer](https://www.msys2.org/)

#### Install QEMU from Pacman:
```shell
pacman -S mingw-w64-x86_64-qemu
```

#### Convert VHDX to QCOW2 using CMD:
```shell
c:\msys64\mingw64\bin\qemu-img.exe convert -f vhdx -O qcow2 ..\SOURCE_DISK.vhdx ..\DESTINATION_DISK.qcow2
```

### 2. Upload the `.qcow2` file to an OCI Object Storage bucket.

#### Using the OCI Console

```
Storage > Object Storage & Archive Storage > Buckets > bucket_name > Upload	
```

#### Using the OCI CLI

```shell
oci os object put -bn <destination_bucket_name> --file <path_to_the_VMDK_or_QCOW2_file>
```

## Create a Custom Image

```
Compute > Custom images > Import image	
```

- Create a **custom image** from the uploaded `.qcow2` located in the bucket:
  - **Operating System**: Windows
  - **Operating System Version**: Server 2022 Standard *(use the correct edition)*
  - **Import from an Object Storage bucket**
  - **Image Type**: QCOW2
  - **Launch Mode**: Emulated Mode

- **Edit the custom image capabilities after provisioning:**
  - **Preferred firmware**: Use **BIOS** (if GEN1), use **UEFI** (if GEN2).
  - **Preferred launch mode**: **EMULATED (Preferred)** or **PARAVIRTUALIZED** *(Provisioning may take longer, ~10-15 min)*.
  - **AMD SEV**: Disabled
  - **Secure Boot**: Disabled
  - **Preferred network attachment type**: PARAVIRTUALIZED
  - **Preferred boot volume type**: PARAVIRTUALIZED
  - **Preferred local data volume type**: PARAVIRTUALIZED
  - **Preferred remote data volume type**: PARAVIRTUALIZED


## Create the Instance

- Create the **instance from the Custom Image**:
  - **Specify the custom boot volume size**:  
    - **USE THE MINIMUM DISK SIZE INDICATED UNDER "BOOT VOLUME SIZE (GB)"**, e.g., **127 GB**.
    - **DO NOT USE THE DEFAULT SIZE (50GB).**


## After the Migration

- Once you have successfully verified **RDP connectivity** to the VM:
  - **Backup the VM**
  - **Enable the local firewall and configure Windows Defender or a third-party firewall**:

```powershell
Set-NetFirewallProfile -Profile Domain,Public,Private -Enabled true
```


## Additional Information

- [Oracle Cloud Infrastructure - Importing a Custom Windows Image](https://docs.oracle.com/en-us/iaas/Content/Compute/Tasks/importingcustomimagewindows.htm)
