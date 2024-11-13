# Scripts Hyper-V

## Création de la VM 'Master' pour l'utiliser comme parent

```powershell
New-VM -Name "Master" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -NewVHDPath "D:\Hyper-V\Master\Master.vhdx" -NewVHDSizeBytes 200GB -SwitchName "Interne"
Set-VM -Name "Master" -ProcessorCount 4 -CheckpointType Disabled
```

## Activer les service invités (comme les tools de VMWare)

```powershell
Enable-VMIntegrationService -VMName "Master" -Name "Interface de services d’invité"
Add-VMDvdDrive -VMName "Master" -Path "D:\ISO\WindowsServer2022.iso"
Set-VMFirmware -VMName "Master" -FirstBootDevice (Get-VMDvdDrive -VMName "Master")
```

## Exemples de variables

```powershell
$iso = "D:\ISO\WindowsServer2022.iso"
$boot = Get-VMDvdDrive -VMName "Master"
```

## Preparer l'image avec Sysprep

`C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown`
Ici on va supprimer mettre le disque en lecture seule et supprimer la VM du gestionnaire Hyper-V

## Créer les différents disques pour en faire des VM enfants

```powershell
New-VHD -Path D:\Hyper-V\DC-01\DC-01.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\DC-02\DC-02.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-01\Hote-01.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-02\Hote-02.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-03\Hote-03.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
```

## Création des VM enfants

```powershell
New-VM -Name DC-01 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\DC-01\DC-01.vhdx
New-VM -Name DC-02 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\DC-02\DC-02.vhdx
New-VM -Name Hote-01 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-01\Hote-01.vhdx
New-VM -Name Hote-02 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-02\Hote-02.vhdx
New-VM -Name Hote-03 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-03\Hote-03.vhdx
```

## Modifier processeurs virtuels et enlever les snapshots

```powershell
Set-VM -Name Hote-01, Hote-02, Hote-03, DC-01, DC-02 -ProcessorCount 2 -CheckpointType Disable
```

## Mettre les services invités

```powershell
Enable-VMIntegrationService -VMName Hote-01, Hote-02, Hote-03, DC-01, DC-02 *interface*
```

## Démarrer VM et la lancer

```powershell
Start-VM -Name DC-01
vmconnect localhost DC-01
```

## Assigner l'IP, la passerelle, le DNS et changer le nom de la machine

```powershell
Get-NetAdapter ## To retrieve the adapter name
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.144.0.1 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
Rename-Computer -NewName "DC-01" -Restart
```

## Installer ADDS

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeAllSubFeature -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "form-it.lab" -InstallDNS
Install-ADDSForest -DomainName "form-it.lab" -DomainNetbiosName "FORM-IT" -ForestMode "WinThreshold" -DomainMode "WinThreshold" -InstallDNS -SafeModeAdministratorPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force)
```

## Installer et configurer DHCP

```powershell
Install-WindowsFeature DHCP -IncludeAllSubFeature -IncludeManagementTools
Add-DhcpServerInDC -DnsName "DC-01.form-it.lab" -IpAddress 10.144.0.1
Add-DhcpServerv4Scope -Name "Etage1" -Description "10.144.0.1" -StartRange 10.144.0.1 -EndRange 10.144.0.254 -SubnetMask 255.255.255.0 -State Active
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.1 -EndRange 10.144.0.30
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.200 -EndRange 10.144.0.254
Set-DhcpServerv4OptionValue -ScopeId 10.144.0.0 -Router 10.144.0.254 -DnsServer 10.144.0.1, 10.144.0.2
```

## Configure reverse DNS

```powershell
Add-DnsServerPrimaryZone -NetworkID 10.144.0.0 -ReplicationScope Domain
```
