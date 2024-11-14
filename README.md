# Script Hyper-V

### 1. Création d'une VM parent

```powershell
New-VM -Name "Main" -MemoryStartupBytes 4GB -Generation 2 -Path "D:\Hyper-V\" -NewVHDPath "D:\Hyper-V\Main\Main.vhdx" -NewVHDSizeBytes 200GB -SwitchName Interne
Set-VM -Name "Main" -ProcessorCount 4 -CheckpointType Disabled
Enable-VMIntegrationService -VMName "Main"
Add-VMDvdDrive -VMName "Main" -Path "D:\ISO\WindowsServer2022.iso"
Set-VMFirmware -VMName "Main" -FirstBootDevice (Get-VMDvdDrive -VMName "Main")
```

### 2. SysPrep

Sur la machine parent:

```powershell
Start-Process "C:\Windows\System32\Sysprep\sysprep.exe" -ArgumentList "/oobe /generalize /shutdown" -NoNewWindow -Wait
```

### 3. Création des VM enfants

Une fois que la VM "Main" est généralisée, tu peux créer les disques différenciés :

```powershell
New-VHD -Path "D:\Hyper-V\DC-01\DC-01.vhdx" -ParentPath "D:\Hyper-V\Main\Main.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\DC-02\DC-02.vhdx" -ParentPath "D:\Hyper-V\Main\Main.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-01\Hote-01.vhdx" -ParentPath "D:\Hyper-V\Main\Main.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-02\Hote-02.vhdx" -ParentPath "D:\Hyper-V\Main\Main.vhdx" -Differencing
New-VHD -Path "D:\Hyper-V\Hote-03\Hote-03.vhdx" -ParentPath "D:\Hyper-V\Main\Main.vhdx" -Differencing
```

```powershell
New-VM -Name "DC-01" -MemoryStartupBytes 2GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-01\DC-01.vhdx" -SwitchName Interne
New-VM -Name "DC-02" -MemoryStartupBytes 2GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\DC-02\DC-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-01" -MemoryStartupBytes 2GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-01\Hote-01.vhdx" -SwitchName Interne
New-VM -Name "Hote-02" -MemoryStartupBytes 2GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-02\Hote-02.vhdx" -SwitchName Interne
New-VM -Name "Hote-03" -MemoryStartupBytes 2GB -Generation 2 -Path "D:\Hyper-V\" -VHDPath "D:\Hyper-V\Hote-03\Hote-03.vhdx" -SwitchName Interne
```

```powershell
Set-VM -Name "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02" -ProcessorCount 2 -CheckpointType Disable
Enable-VMIntegrationService -VMName "Hote-01", "Hote-02", "Hote-03", "DC-01", "DC-02"
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
```

- Hote-01:

```powershell
Rename-Computer -NewName "Hote-01" -Restart
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.10 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
```

- Hote-02:

```powershell
Rename-Computer -NewName "Hote-02" -Restart
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.20 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
```

- Hote-03:

```powershell
Rename-Computer -NewName "Hote-03" -Restart
Rename-NetAdapter -Name Ethernet -NewName Interne
New-NetIPAddress -InterfaceAlias Interne -IPAddress 10.144.0.30 -PrefixLength 24 -DefaultGateway 10.144.0.254
Set-DnsClientServerAddress -InterfaceAlias Interne -ServerAddresses 10.144.0.1, 10.144.0.2
Disable-NetAdapterBinding -Name Interne -ComponentID ms_tcpip6
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
Add-ADGroupMember -Identity "Direction" -Members "Arnold"
Add-ADGroupMember -Identity "Negociation" -Members "Marwa"
Add-ADGroupMember -Identity "Formation" -Members "Manel"
Add-ADGroupMember -Identity "Communication" -Members "Faical"
Add-ADGroupMember -Identity "Vendeur" -Members "Eliott"
```
