# Script Hyper-V

### 0. Créer VM Switch

```powershell
Get-NetAdapter
New-VMSwitch -Name Externe -NetAdapterName "Wi-Fi"
New-VMSwitch -Name Interne -SwitchType Internal
New-VMSwitch -Name Pulsation -SwitchType Private
New-VMSwitch -Name MPIO1 -SwitchType Private
New-VMSwitch -Name MPIO2 -SwitchType Private
```

### 1. Création d'une VM parent

```powershell
New-VM -Name "Master" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -NewVHDPath "D:\Hyper-V\Master\Master.vhdx" -NewVHDSizeBytes 200GB -SwitchName Interne
Set-VM -Name "Master" -ProcessorCount 4 -CheckpointType Disabled
Enable-VMIntegrationService -VMName "Master" *interface*
Add-VMDvdDrive -VMName "Master" -Path "D:\ISO\WindowsServer2022.iso"
Set-VMFirmware -VMName "Master" -FirstBootDevice (Get-VMDvdDrive -VMName "Master")
```

### 2. SysPrep

Sur la machine parent:

```powershell
Start-Process "C:\Windows\System32\Sysprep\sysprep.exe" -ArgumentList "/oobe /generalize /shutdown" -NoNewWindow -Wait
```

### 3. Création des VM enfants

Une fois que la VM "Master" est généralisée, tu peux créer les disques différenciés :

```powershell
New-VHD -Path "D:\Hyper-V\DC-01\DC-01.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\DC-02\DC-02.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-01\Hote-01.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-02\Hote-02.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-03\Hote-03.vhdx" -ParentPath "D:\Hyper-V\Master\Master.vhdx" -Differencing
```

```powershell
New-VM -Name "DC-01" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-01\DC-01.vhdx" -SwitchName Interne
New-VM -Name "DC-02" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-02\DC-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-01" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-01\Hote-01.vhdx" -SwitchName Interne
New-VM -Name "Hote-02" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-02\Hote-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-03" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-03\Hote-03.vhdx" -SwitchName Interne
```

```powershell
Set-VM -Name "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02" -ProcessorCount 4 -CheckpointType Disable
Enable-VMIntegrationService -VMName "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02" *interface*
```

### 4. Configurations IP des machines

- DC-01:

```powershell
Rename-Computer -NewName "DC-01" -Restart
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.1 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
```

- DC-02:

```powershell
Rename-Computer -NewName "DC-02" -Restart
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.2 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6

Add-Computer -DomainName "form-it.lab" -Credential "FORM-IT\Administrateur" -Restart
Install-ADDSDomainController -DomainName "form-it.lab"

Add-DhcpServerv4Failover -ComputerName "DC-01" -PartnerServer "DC-02" -Name "Failover-Partner" -LoadBalancePercent 50 -SharedSecret "Azerty123"  -ScopeId 10.144.0.0
```

- Hote-01:

```powershell
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.10 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
Rename-Computer -NewName "Hote-01" -Restart
```

- Hote-02:

```powershell
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.20 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
Rename-Computer -NewName "Hote-02" -Restart
```

- Hote-03:

```powershell
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.30 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
Rename-Computer -NewName "Hote-03" -Restart
```

### 5. ADDS DC-01

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeAllSubFeature -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "form-it.lab" -DomainNetbiosName "FORM-IT" -ForestMode WinThreshold -DomainMode WinThreshold -InstallDNS
```

### 6. DHCP DC-01

```powershell
Install-WindowsFeature DHCP -IncludeAllSubFeature -IncludeManagementTools
Add-DhcpServerInDC -DnsName "DC-01.form-it.lab" -IpAddress 10.144.0.1
# Plage
Add-DhcpServerv4Scope -Name "Scope1" -Description "10.144.0.1/24" -StartRange 10.144.0.1 -EndRange 10.144.0.254 -SubnetMask 255.255.255.0 -State Active
# Exclusions
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.1 -EndRange 10.144.0.30
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.200 -EndRange 10.144.0.254
# DNS, Gateway
Set-DhcpServerv4OptionValue -ScopeId 10.144.0.0 -Router 10.144.0.254 -DnsServer 10.144.0.1, 10.144.0.2 -Force
```

# Abandon

New-ADGroup -Name "Administrateurs DHCP" -Path "CN=Users,DC=form-it,DC=lab" -GroupScope Global -GroupCategory Security -Description "Groupe des administrateurs DHCP" -ErrorAction SilentlyContinue
New-ADGroup -Name "Utilisateurs DHCP" -Path "CN=Users,DC=form-it,DC=lab" -GroupScope Global -GroupCategory Security -Description "Groupe des utilisateurs DHCP" -ErrorAction SilentlyContinue

### 6. DNS Plage inversée

```powershell
Add-DnsServerPrimaryZone -Name "0.144.10.in-addr.arpa" -ReplicationScope "Domain"
Add-DnsServerResourceRecordPtr -Name "1" -PtrDomainName "DC-01.form-it.lab" -ZoneName "0.144.10.in-addr.arpa"
```

### 7. OU, Groupes, Users active directory

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

### 8. Ajout de disques

- Créations de 9 disques 4To:

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

- Ajout des disques à Hote-03:

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

### 9. iSCSI

Hote 03

- Ajouté les fonctionnalités iSCSI :

```powershell
Install-WindowsFeature iSCSITarget-VSS-VDS, FS-iSCSITarget-Server -IncludeAllSubFeature -IncludeManagementTools
```

- Faire un pool:

```powershell
# Récupérer les informations des disques présents sur la machine
Get-PhysicalDisk
# Filtrer les disques qui peuvent être Pool
$disks = Get-PhysicalDisk | Where-Object CanPool -eq True

# Pour connaitre le nom
Get-StorageSubSystem
# Créer un pool de stockage avec ces disques
New-StoragePool -FriendlyName "Pool" -StorageSubsystemFriendlyName "Storage Spaces*" -PhysicalDisks $disks
```

```powershell
# Étape 3 : Créer un espace de stockage avec le provisionnement fin
New-VirtualDisk -StoragePoolFriendlyName "Pool" -FriendlyName "VirtualDisk" -Size 128TB -ResiliencySettingName Mirror -NumberOfColumns 3 -ProvisioningType Thin
```

faire un disque virtuel dans le volume
Creer cibles iscsi

Hote 01:
Ajouter cartes réseaux renommés avec ip et masque
ajout des fontionnalités clustering de basculement, equilibrage de la charge réseau et MPIO

```powershell
Install-WindowsFeature -Name Failover-Clustering, RSAT-NLB, Multipath-IO -IncludeManagementTools -IncludeAllSubFeature
```

outils -> mpio
outils -> initiateur iscsi
lier hote3->hote1
demarrer disques
pareil pour hote 2

faire un test avec
failover cluster manager

Avant de créer le cluster
vérifier zone dns inversée (DC-01)
nom 144.10.in-addr...
-> nouvel hote
cluster-failover = 10.144.0.144
cluster-nbl = 10.144.0.200

Un fois cluster fait, ajouter disques pour le cluster, virtualisation imbriquée, activer l'usurpation adresse mac
ajouter les disques partagés dans le cluster, hote1 a access, hote2 aussi, ajouter cluster disques dans gestionnaire du cluster de basculement (ajouter au volume partagé)
configurer le quorum...

Eteindre Hote 1/hote 2 pour activer virtualisation imbriquée, et usurpation
puis redemarrer et ajouter hyperv
