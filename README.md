# Le script

## => PC Physique

## Récupérer les cartes réseaux pour faire la VMSwitch Externe (Net-AdapterName)

```powershell
Get-NetAdapter
```

## Faire les VMSwitch

```powershell
New-VMSwitch -Name Externe -NetAdapterName "Wi-Fi"
New-VMSwitch -Name Interne -SwitchType Internal
New-VMSwitch -Name Pulsation -SwitchType Private
New-VMSwitch -Name MPIO1 -SwitchType Private
New-VMSwitch -Name MPIO2 -SwitchType Private
```

## Créer la VM Parent

```powershell
New-VM -Name "Master" -MemoryStartupBytes 16GB -Generation 2 -Path "D:\Hyper-V\" -NewVHDPath "D:\Hyper-V\Master\Master.vhdx" -NewVHDSizeBytes 200GB -SwitchName Interne
Set-VM -Name "Master" -ProcessorCount 8 -CheckpointType Disabled
Enable-VMIntegrationService -VMName "Master" *interface*
Add-VMDvdDrive -VMName "Master" -Path "D:\ISO\WindowsServer2022.iso"
Set-VMFirmware -VMName "Master" -FirstBootDevice (Get-VMDvdDrive -VMName "Master")
```

## => VM Master

### Préparer la VM pour faire les enfants

```powershell
Start-Process "C:\Windows\System32\Sysprep\sysprep.exe" -ArgumentList "/oobe /generalize /shutdown" -NoNewWindow -Wait
```

## => PC Physique

### Créer les 5 disques pour les enfants

```powershell
New-VHD -Path "D:\Hyper-V\DC-01\DC-01.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\DC-02\DC-02.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-01\Hote-01.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-02\Hote-02.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-03\Hote-03.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
```

### Créer les VM enfants

```powershell
New-VM -Name "DC-01" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-01\DC-01.vhdx" -SwitchName Interne
New-VM -Name "DC-02" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-02\DC-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-01" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-01\Hote-01.vhdx" -SwitchName Interne
New-VM -Name "Hote-02" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-02\Hote-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-03" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-03\Hote-03.vhdx" -SwitchName Interne
```

### Changer le nombre de processeurs et enlever les checkpoints (snapshots)

```powershell
Set-VM -Name "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02" -ProcessorCount 4 -CheckpointType Disable
```

### Activer les services invités

```powershell
Enable-VMIntegrationService -VMName "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02" *interface*
```

## => VM DC-01

### Changer les paramètres réseau

```powershell
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.1 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
```

### Désactiver l'ipv6

```powershell
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
```

### Renommer la machine et redémarrer

```powershell
Rename-Computer -NewName "DC-01" -Restart
```

### Faire l'ADDS

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeAllSubFeature -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "form-it.lab" -DomainNetbiosName "FORM-IT" -ForestMode WinThreshold -DomainMode WinThreshold -InstallDNS
```

### Faire le DHCP et les scope

```powershell
Install-WindowsFeature DHCP -IncludeAllSubFeature -IncludeManagementTools
Add-DhcpServerInDC -DnsName "DC-01.form-it.lab" -IpAddress 10.144.0.1
Add-DhcpServerv4Scope -Name "Scope1" -Description "10.144.0.1/24" -StartRange 10.144.0.1 -EndRange 10.144.0.254 -SubnetMask 255.255.255.0 -State Active
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.1 -EndRange 10.144.0.30
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.200 -EndRange 10.144.0.254
Set-DhcpServerv4OptionValue -ScopeId 10.144.0.0 -Router 10.144.0.254 -DnsServer 10.144.0.1, 10.144.0.2 -Force
```

### Faire la plage inversée

```powershell
Add-DnsServerPrimaryZone -Name "0.144.10.in-addr.arpa" -ReplicationScope "Domain"
Add-DnsServerResourceRecordPtr -Name "1" -PtrDomainName "DC-01.form-it.lab" -ZoneName "0.144.10.in-addr.arpa"
```

### OU, Groupes, Users active directory

```powershell
# OU
New-ADOrganizationalUnit -Name "Direction"
New-ADOrganizationalUnit -Name "Commerciaux"
New-ADOrganizationalUnit -Name "RH"
New-ADOrganizationalUnit -Name "Marketing"
New-ADOrganizationalUnit -Name "Vente"

# Group
New-ADGroup -Name "Directeur" -GroupScope Global -GroupCategory Security -Path "OU=Direction,DC=form-it,DC=lab"
New-ADGroup -Name "Negociation" -GroupScope Global -GroupCategory Security -Path "OU=Commerciaux,DC=form-it,DC=lab"
New-ADGroup -Name "Formation" -GroupScope Global -GroupCategory Security -Path "OU=RH,DC=form-it,DC=lab"
New-ADGroup -Name "Communication" -GroupScope Global -GroupCategory Security -Path "OU=Marketing,DC=form-it,DC=lab"
New-ADGroup -Name "Vendeur" -GroupScope Global -GroupCategory Security -Path "OU=Vente,DC=form-it,DC=lab"

# User
New-ADUser -Name "Arnold" -SamAccountName "Arnold" -Path "OU=Direction,DC=form-it,DC=lab" -AccountPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Marwa" -SamAccountName "Marwa" -Path "OU=Commerciaux,DC=form-it,DC=lab" -AccountPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Manel" -SamAccountName "Manel" -Path "OU=RH,DC=form-it,DC=lab" -AccountPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Faical" -SamAccountName "Faical" -Path "OU=Marketing,DC=form-it,DC=lab" -AccountPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force) -Enabled $true
New-ADUser -Name "Eliott" -SamAccountName "Eliott" -Path "OU=Vente,DC=form-it,DC=lab" -AccountPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force) -Enabled $true

# User -> Group
Add-ADGroupMember -Identity "Directeur" -Members "Arnold"
Add-ADGroupMember -Identity "Negociation" -Members "Marwa"
Add-ADGroupMember -Identity "Formation" -Members "Manel"
Add-ADGroupMember -Identity "Communication" -Members "Faical"
Add-ADGroupMember -Identity "Vendeur" -Members "Eliott"
```

## => PC Physique

### Créations de 9 disques 4To:

```powershell
New-VHD -Path "D:\Hyper-V\Hote-03\dd1.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd2.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd3.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd4.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd5.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd6.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd7.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd8.vhdx" -SizeBytes 4TB -Dynamic
New-VHD -Path "D:\Hyper-V\Hote-03\dd9.vhdx" -SizeBytes 4TB -Dynamic
```

### Ajout des disques à Hote-03:

```powershell
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd1.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd2.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd3.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd4.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd5.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd6.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd7.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd8.vhdx"
Add-VMHardDiskDrive -VMName "Hote-03" -Path "D:\Hyper-V\Hote-03\dd9.vhdx"
```

## => VM Hote-03

### Installer iSCSI

```powershell
Install-WindowsFeature iSCSITarget-VSS-VDS, FS-iSCSITarget-Server -IncludeAllSubFeature -IncludeManagementTools
```

### Faire le pool

```powershell
Get-PhysicalDisk
Get-StorageSubSystem
New-StoragePool -FriendlyName "PoolMiroirTriple" -StorageSubsystemFriendlyName *Hote-03* -PhysicalDisks (Get-PhysicalDisk | Where-Object CanPool -eq True)
```

### Faire le disque virtuel

```powershell
New-VirtualDisk -StoragePoolFriendlyName "PoolMiroirTriple" -FriendlyName "DisqueVirtuel128TO" -Size 128TB -ProvisioningType Thin -ResiliencySettingName "Mirror" -NumberOfColumns 3
$disk = Get-VirtualDisk -FriendlyName "DisqueVirtuel128TO" | Get-Disk
Initialize-Disk -Number $disk.Number
```

### Faire une partition du disque

```powershell
New-Partition -DiskNumber $disk.Number -Size (64TB) -AssignDriveLetter
Format-Volume -DriveLetter D -FileSystem NTFS -NewFileSystemLabel "Volume64TO"
```

### Préparer la cible iSCSI

```powershell
New-IscsiServerTarget -TargetName "CibleISCSI" -InitiatorIds "IPAddress:10.144.0.10", "IPAddress:10.144.0.20", "IPAddress:10.144.1.10", "IPAddress:10.144.1.20", "IPAddress:10.144.2.10", "IPAddress:10.144.2.20"
```

### Créer les 3 Disques iSCSI et le Quorum

```powershell
for ($i = 1; $i -le 3; $i++) {
    New-IscsiVirtualDisk -Path "D:\DisqueISCSI_$i.vhdx" -Size 15TB
}
New-IscsiVirtualDisk -Path "D:\DisqueISCSI_Quorum.vhdx" -Size 100GB
```

### Ajouter les cibles à ces disques

```powershell
for ($i = 1; $i -le 3; $i++) {
    Add-IscsiVirtualDiskTargetMapping -TargetName "CibleISCSI" -Path "D:\DisqueISCSI_$i.vhdx"
}
Add-IscsiVirtualDiskTargetMapping -TargetName "CibleISCSI" -Path "D:\DisqueISCSI_Quorum.vhdx"
```
