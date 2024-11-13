# Commandes PowerShell

Faire une VM nommée Master qu'on va utiliser comme parent

```powershell
New-VM -Name "Master" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -NewVHDPath "D:\Hyper-V\Master\Master.vhdx" -NewVHDSizeBytes 200GB -SwitchName "Interne"
Set-VM -Name "Master" -ProcessorCount 4 -CheckpointType Disabled
# Get-VMIntegrationService -VMName "Master" pour avoir la liste de tous les services

Enable-VMIntegrationService -VMName "Master" -Name "Interface de services d’invité"
Add-VMDvdDrive -VMName "Master" -Path "D:\ISO\WindowsServer2022.iso"
Set-VMFirmware -VMName "Master" -FirstBootDevice (Get-VMDvdDrive -VMName "Master")
```

On peut aussi faire des variables avec par exemple:

```powershell
&iso = "D:\ISO\WindowsServer2022.iso"
```

ou plus intéressant

```powershell
$boot = Get-VMDvdDrive -VMName "Master"
```

Dans Windows Server, pour préparer une "image" que vont utiliser les enfants

`C:\Windows\System32\Sysprep\sysprep.exe /oobe /generalize /shutdown`

## Ici, on garde juste le disque parent qu'on a mis en lecture seule et on a supprimé la VM de l'interface Hyper-V

Création de 5 enfants DC-01, DC-02, Hote-01, Hote-02, Hote-03 qui vont être liés à Master en mode differencing qui permet de gagner la place de ce qui était deja sur le Master (ici, Windows Server -> en gros, l'OS est partagé)

```powershell
New-VHD -Path D:\Hyper-V\DC-01\DC-01.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\DC-02\DC-02.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-01\Hote-01.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-02\Hote-02.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
New-VHD -Path D:\Hyper-V\Hote-03\Hote-03.vhdx -ParentPath D:\Hyper-V\Master\Master.vhdx -Differencing
```

```powershell
New-VM -Name DC-01 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\DC-01\DC-01.vhdx
New-VM -Name DC-02 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\DC-02\DC-02.vhdx
New-VM -Name Hote-01 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-01\Hote-01.vhdx
New-VM -Name Hote-02 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-02\Hote-02.vhdx
New-VM -Name Hote-03 -MemoryStartupBytes 2GB -Generation 2 -Path D:\Hyper-V\ -SwitchName Interne -VHDPath D:\Hyper-V\Hote-03\Hote-03.vhdx
```

Retire les snapshot des machines et mets les processeurs à 2

```powershell
Set-VM -Name Hote-01, Hote-02, Hote-03, DC-01, DC-02 -ProcessorCount 2 -CheckpointType Enabled
```

Ajoute l'interface de services pour les machines

```powershell
Enable-VMIntegrationService -VMName Hote-01, Hote-02, Hote-03, DC-01, DC-02 *interface*
```

## Démarrer une VM

```powershell
Start-VM -Name DC-01
vmconnect DC-01 DC-01
```

## Changer le nom de la machine et son adressage IP

```powershell
Get-NetAdapter # Pour récupérer le nom de l'interface
Rename-NetAdapter -Name Ethernet -NewName Interne # Optionnel: Renommer la carte réseau

New-NetIPAddress -InterfaceAlias "Ethernet" -IPAddress 10.144.0.1 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name "Ethernet" -ComponentID ms_tcpip6
Rename-Computer -NewName "DC-01" -Restart
```

- Installer ADDS

```powershell
Install-WindowsFeature AD-Domain-Services -IncludeAllSubFeature -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "form-it.lab" -InstallDNS

# Plein d'autres paramétrages sont possibles, mais puisque qu'on prends les éléments par défaut on en a pas besoin
Install-ADDSForest -DomainName "form-it.lab" -DomainNetbiosName "FORM-IT" -ForestMode "WinThreshold" -DomainMode "WinThreshold" -InstallDNS -SafeModeAdministratorPassword (ConvertTo-SecureString "Azerty123" -AsPlainText -Force)
```

## Installation et configuration de DHCP

```powershell
Install-WindowsFeature DHCP -IncludeAllSubFeature -IncludeManagementTools
Add-DhcpServerInDC -DnsName "DC-01.form-it.lab" -IpAddress 10.144.0.1
```

- Autoriser l'administrateur

```powershell
Add-DhcpServerSecurityGroupMember -ComputerName "FORM-IT" -UserName "administrateur"
```

- Créer l'étendue DHCP

```powershell
Add-DhcpServerv4Scope -Name "Etage1" -Description "10.144.0.1" -StartRange 10.144.0.1 -EndRange 10.144.0.254 -SubnetMask 255.255.255.0 -State Active
```

- Ajouter les exclusions

```powershell
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.1 -EndRange 10.144.0.30
Add-DhcpServerv4ExclusionRange -ScopeId 10.144.0.0 -StartRange 10.144.0.200 -EndRange 10.144.0.254
```

- Configurer les options DHCP

```powershell
Set-DhcpServerv4OptionValue -ScopeId 10.144.0.0 -Router 10.144.0.254 -DnsServer 10.144.0.1, 10.144.0.2
```

- Configurer le reverse Dns

```powershell
Add-DnsServerPrimaryZone -NetworkID 10.144.0.0 -ReplicationScope Domain
```
