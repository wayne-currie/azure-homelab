# 🏢 P2S Active Directory Homelab in Azure

> Building an enterprise-grade Active Directory environment using Azure infrastructure with Point-to-Site VPN connectivity

[![Azure](https://img.shields.io/badge/Azure-0078D4?style=for-the-badge&logo=microsoft-azure&logoColor=white)](https://azure.microsoft.com/)
[![Windows Server](https://img.shields.io/badge/Windows_Server_2022-0078D6?style=for-the-badge&logo=windows&logoColor=white)](https://www.microsoft.com/windows-server)
[![Active Directory](https://img.shields.io/badge/Active_Directory-0078D4?style=for-the-badge&logo=windows&logoColor=white)](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
[![PowerShell](https://img.shields.io/badge/PowerShell-5391FE?style=for-the-badge&logo=powershell&logoColor=white)](https://docs.microsoft.com/en-us/powershell/)

---


## 🎯 Project Overview

This project demonstrates the deployment of a fully functional **Active Directory Domain Services (AD DS)** environment hosted in Microsoft Azure, seamlessly connected to a local Windows 11 virtual machine through a secure **Point-to-Site VPN** tunnel.

### 🌟 Key Highlights

- ☁️ **Cloud-Hosted Domain Controller** - Windows Server 2022 in Azure
- 🔐 **Secure Connectivity** - Certificate-based P2S VPN
- 🌐 **Custom Domain** - homelab.local
- 📡 **Integrated DNS** - Azure VNet DNS configuration
- 🖥️ **Hybrid Setup** - Local VM joined to cloud domain

---

## 🏗️ Network Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Azure Cloud                             │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │          Virtual Network (10.0.0.0/16)                    │  │
│  │                                                           │  │
│  │  ┌──────────────────┐         ┌──────────────────────┐    │  │
│  │  │  GatewaySubnet   │         │   ServerSubnet       │    │  │
│  │  │  10.0.1.0/24     │         │   10.0.2.0/24        │    │  │
│  │  │                  │         │                      │    │  │
│  │  │  ┌────────────┐  │         │  ┌───────────────┐   │    │  │
│  │  │  │    VPN     │  │◄───────►│  │    Domain     │   │    │  │
│  │  │  │  Gateway   │  │         │  │  Controller   │   │    │  │
│  │  │  │            │  │         │  │  (DC01)       │   │    │  │
│  │  │  │ 10.0.0.4   │  │         │  │  10.0.2.10    │   │    │  │
│  │  │  └────────────┘  │         │  └───────────────┘        │  │
│  │  └──────────────────┘         └──────────────────────┘    │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                               ▲
                               │
                               │ Point-to-Site VPN
                               │ Client Pool: 10.1.0.0/24
                               │ Protocol: IKEv2
                               │
                               ▼
                    ┌──────────────────────┐
                    │   Local Environment  │
                    │                      │
                    │  ┌────────────────┐  │
                    │  │  Windows 11 VM │  │
                    │  │    (VMware)    │  │
                    │  │                │  │
                    │  │  Domain Member │  │
                    │  └────────────────┘  │
                    └──────────────────────┘
```

### 📊 Network Specifications

| Component | Address Space | Purpose |
|-----------|---------------|---------|
| **Azure VNet** | 10.0.0.0/16 | Primary virtual network |
| **GatewaySubnet** | 10.0.1.0/24 | VPN Gateway infrastructure |
| **ServerSubnet** | 10.0.2.0/24 | Domain Controller & servers |
| **VPN Client Pool** | 10.1.0.0/24 | Remote client addressing |

---

## ✅ Prerequisites

Before beginning this project, ensure you have:

- ☁️ **Azure Subscription** with appropriate permissions
- 💻 **VMware Workstation/Player** with Windows 11 VM
- 🔧 **PowerShell 5.1+** with administrative access
- 📚 **Basic networking knowledge** (TCP/IP, DNS, subnetting)
- 🎓 **Familiarity with Active Directory** concepts

---

## 🚀 Implementation Guide

### Step 1️⃣: Create Azure Virtual Network

Set up the isolated network infrastructure for your AD environment.

**Configuration:**

```yaml
Resource Group: EntraID-Homelab-RG
Virtual Network Name: AD-VNet
Region: [Your preferred region]
Address Space: 10.0.0.0/16

Subnets:
  - Name: GatewaySubnet
    Address Range: 10.0.1.0/24
    
  - Name: ServerSubnet
    Address Range: 10.0.2.0/24
```

<details>
<summary>📸 Click to view Azure Portal steps</summary>

1. Navigate to **Virtual Networks** in Azure Portal
2. Click **+ Create**
3. Configure basic settings with values above
4. Create both subnets during VNet creation
5. Review and create

</details>

---

### Step 2️⃣: Deploy VPN Gateway

Create a Virtual Network Gateway for secure remote connectivity.

**Gateway Configuration:**

| Setting | Value |
|---------|-------|
| **Name** | AD-VNet-Gateway |
| **Gateway Type** | VPN |
| **VPN Type** | Route-based |
| **SKU** | VpnGw1 |
| **Virtual Network** | AD-VNet |
| **Public IP** | Create new (AD-VPN-PIP) |

> ⏱️ **Deployment Time:** Approximately 30-45 minutes

---

### Step 3️⃣: Generate VPN Certificates

Create self-signed certificates for VPN authentication.

#### 📜 Step 3.1: Create Root Certificate

Open **PowerShell as Administrator** and run:

```powershell
# Create Self-Signed Root Certificate
$params = @{
    Type = 'Custom'
    Subject = 'CN=AzureRoot'
    KeySpec = 'Signature'
    KeyExportPolicy = 'Exportable'
    KeyUsage = 'CertSign'
    KeyUsageProperty = 'Sign'
    KeyLength = 2048
    HashAlgorithm = 'sha256'
    NotAfter = (Get-Date).AddMonths(24)
    CertStoreLocation = 'Cert:\CurrentUser\My' 
}
$cert = New-SelfSignedCertificate @params

# Export root certificate public key
Export-Certificate -Cert $cert -FilePath "C:\AzureRoot.cer"

Write-Host "✅ Root certificate created and exported to C:\AzureRoot.cer" -ForegroundColor Green
```

#### 📜 Step 3.2: Create Client Certificate

```powershell
# Create Client Certificate signed by Root
$clientParams = @{
    Type = 'Custom'
    Subject = 'CN=AzureClient'
    DnsName = 'AzureClient'
    KeySpec = 'Signature'
    KeyExportPolicy = 'Exportable'
    KeyLength = 2048
    HashAlgorithm = 'sha256'
    NotAfter = (Get-Date).AddMonths(18)
    CertStoreLocation = 'Cert:\CurrentUser\My'
    Signer = $cert
    TextExtension = @('2.5.29.37={text}1.3.6.1.5.5.7.3.2')
}
New-SelfSignedCertificate @clientParams

Write-Host "✅ Client certificate created successfully" -ForegroundColor Green
```

**📝 Important Notes:**

| Certificate Type | Storage Location | Purpose |
|-----------------|------------------|---------|
| **Root (.cer)** | Exported to C:\ | Upload to Azure VPN Gateway |
| **Client** | Windows Certificate Store | Authenticate VPN connections |

> 🔐 **Security:** The root certificate's private key remains secure on your machine. Only the public key (.cer) is uploaded to Azure.

---

### Step 4️⃣: Configure Point-to-Site VPN

Configure P2S settings in the Virtual Network Gateway.

**Configuration Steps:**

1. Navigate to your **VPN Gateway** in Azure Portal
2. Select **Point-to-site configuration** from the left menu
3. Configure the following settings:

```yaml
Address Pool: 10.1.0.0/24
Tunnel Type: 
  ☑ IKEv2
  ☑ OpenVPN (SSL)
Authentication Type: Azure certificate
```

4. **Upload Root Certificate:**
   - Click **+ Add root certificate**
   - Name: `AzureRootCert`
   - Upload the `C:\AzureRoot.cer` file content (base64)

5. Click **Save** and wait for configuration to complete

6. **Download VPN Client** package

<details>
<summary>💡 How to extract certificate data for Azure</summary>

```powershell
# Open the .cer file in Notepad
notepad C:\AzureRoot.cer

# Copy everything between these lines:
# -----BEGIN CERTIFICATE-----
# [certificate data]
# -----END CERTIFICATE-----

# Paste only the certificate data (without BEGIN/END lines) into Azure
```

</details>

---

### Step 5️⃣: Install VPN Client

Install the VPN client on your Windows 11 machine.

**Installation Process:**

1. **Download** the VPN client package from Azure Portal
   - Location: VPN Gateway → Point-to-site configuration → Download VPN client

2. **Extract** the downloaded ZIP file

3. **Navigate** to `WindowsAmd64` folder

4. **Run** `VpnClientSetupAmd64.exe` as Administrator

5. **Verify** installation:
   ```powershell
   Get-VpnConnection | Where-Object {$_.Name -like "*Azure*"}
   ```

> ✅ **Success Indicator:** You should see a new VPN connection in Windows Network Settings

---

### Step 6️⃣: Connect to Azure VPN

Establish your first VPN connection to Azure.

**Connection Steps:**

1. Open **Settings** → **Network & Internet** → **VPN**
2. Select the Azure VPN connection
3. Click **Connect**
4. Connection will authenticate using your client certificate

**Verify Connection:**

```powershell
# Check VPN connection status
Get-VpnConnection | Where-Object {$_.ConnectionStatus -eq "Connected"}

# Verify assigned IP address
ipconfig | Select-String "10.1.0"
```

**Expected Output:**

| Parameter | Value |
|-----------|-------|
| **Status** | ✅ Connected |
| **Assigned IP** | 10.1.0.x |
| **Gateway** | 10.0.0.4 |
| **Protocol** | IKEv2 |

![VPN Connection Status](https://github.com/user-attachments/assets/768f5c6a-b776-4187-9064-74847a66a125)

---

### Step 7️⃣: Deploy Windows Server 2022

Create the virtual machine that will host Active Directory.

**VM Specifications:**

```yaml
Virtual Machine Name: DC01
Resource Group: EntraID-Homelab-RG
Region: [Same as VNet]

Configuration:
  Size: Standard_B2s (2 vCPUs, 4 GB RAM)
  Image: Windows Server 2022 Datacenter
  OS Disk: 127 GB Premium SSD
  
Networking:
  Virtual Network: AD-VNet
  Subnet: ServerSubnet (10.0.2.0/24)
  Private IP: 10.0.2.10 (Static)
  Public IP: None (access via VPN only)
  
Security:
  Network Security Group: Create new (DC-NSG)
  Inbound Rules:
    - RDP (3389) from VPN subnet
    - DNS (53) from VPN subnet
    - AD-related ports from VNet
```

**Initial Configuration Checklist:**

After VM deployment, perform these steps:

- [ ] Set static private IP in Azure NIC settings
- [ ] Configure Windows Firewall for AD/DNS
- [ ] Disable IE Enhanced Security Configuration
- [ ] Install latest Windows Updates
- [ ] Set appropriate timezone
- [ ] Rename computer to DC01

```powershell
# Quick setup script (run on server)
# Rename computer
Rename-Computer -NewName "DC01" -Force

# Set timezone
Set-TimeZone -Id "Eastern Standard Time"

# Disable IE Enhanced Security
$AdminKey = "HKLM:\SOFTWARE\Microsoft\Active Setup\Installed Components\{A509B1A7-37EF-4b3f-8CFC-4F3A74704073}"
Set-ItemProperty -Path $AdminKey -Name "IsInstalled" -Value 0

# Install Windows Updates
Install-Module PSWindowsUpdate -Force
Get-WindowsUpdate -Install -AcceptAll -AutoReboot
```

![Windows Server VM Deployment](https://github.com/user-attachments/assets/630c10e4-15bf-4b2c-9db7-40d91ee84d89)

---

### Step 8️⃣: Configure Domain Controller

Promote Windows Server 2022 to an Active Directory Domain Controller.

#### 🔧 Phase 1: Install AD DS Role

```powershell
# Install Active Directory Domain Services
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools

Write-Host "✅ AD DS role installed successfully" -ForegroundColor Green
```

#### 🔧 Phase 2: Promote to Domain Controller

```powershell
# Define forest configuration
$DomainName = "homelab.local"
$DomainNetBIOS = "HOMELAB"
$SafeModePassword = ConvertTo-SecureString "YourStrongPassword123!" -AsPlainText -Force

# Promote server to Domain Controller
Install-ADDSForest `
    -DomainName $DomainName `
    -DomainNetbiosName $DomainNetBIOS `
    -ForestMode "WinThreshold" `
    -DomainMode "WinThreshold" `
    -InstallDns:$true `
    -SafeModeAdministratorPassword $SafeModePassword `
    -Force:$true

# Server will automatically restart
```

> ⚠️ **Important:** The server will restart automatically after promotion. Wait 5-10 minutes before reconnecting.

#### 🔧 Phase 3: Post-Promotion Configuration

After the server restarts, configure DNS and create OUs:

```powershell
# Configure DNS Forwarders
Add-DnsServerForwarder -IPAddress "8.8.8.8" -PassThru
Add-DnsServerForwarder -IPAddress "1.1.1.1" -PassThru

# Create Organizational Units
$OUs = @("Users", "Computers", "Groups", "Servers", "Workstations")
foreach ($OU in $OUs) {
    New-ADOrganizationalUnit -Name $OU -Path "DC=homelab,DC=local" -ProtectedFromAccidentalDeletion $true
}

# Verify DNS zones
Get-DnsServerZone

Write-Host "✅ Domain Controller configuration complete!" -ForegroundColor Green
```

#### ☁️ Phase 4: Update Azure VNet DNS

Configure Azure VNet to use the Domain Controller for DNS:

1. Navigate to **AD-VNet** in Azure Portal
2. Select **DNS servers** from left menu
3. Choose **Custom**
4. Add DNS server: `10.0.2.10`
5. Click **Save**

> 🔄 **Note:** You may need to restart VMs or renew VPN connection for DNS changes to take effect.

**Domain Configuration Summary:**

| Setting | Value |
|---------|-------|
| **Domain Name** | homelab.local |
| **NetBIOS Name** | HOMELAB |
| **Functional Level** | Windows Server 2016 |
| **DNS Server** | 10.0.2.10 |
| **Forest Root** | homelab.local |

![Domain Controller Configuration](https://github.com/user-attachments/assets/4920f984-aeb5-4398-ac98-9e548ccc2e19)

---

### Step 9️⃣: Join Windows 11 to Domain

Connect your local Windows 11 VM to the Azure-hosted domain.

#### 🔍 Phase 1: Verify Network Connectivity

**Test 1: Ping Domain Controller IP**

```powershell
# Ensure VPN is connected first
Test-Connection -ComputerName 10.0.2.10 -Count 4
```

✅ **Expected Result:** Successful replies from 10.0.2.10

![Ping to Azure Server](https://github.com/user-attachments/assets/b0c38e16-b5b3-42ff-927e-dd4e4f4bf529)

**Test 2: Ping Domain Name**

```powershell
Test-Connection -ComputerName homelab.local -Count 4
```

❌ **If this fails**, proceed to DNS configuration below.

#### 🔧 Phase 2: Configure DNS for VPN Interface

If you cannot resolve the domain name, manually configure DNS:

```powershell
# Step 1: Identify VPN interface
Get-NetAdapter | Where-Object {$_.Name -like "*p2s*" -or $_.InterfaceDescription -like "*VPN*"}

# Step 2: Set DNS to Domain Controller
# Replace 'p2sVPN-VNet1' with your actual VPN interface name
Set-DnsClientServerAddress -InterfaceAlias "p2sVPN-VNet1" -ServerAddresses "10.0.2.10"

# Step 3: Verify DNS configuration
Get-DnsClientServerAddress -InterfaceAlias "p2sVPN-VNet1"

# Step 4: Test DNS resolution
Resolve-DnsName homelab.local
nslookup homelab.local
```

<img width="973" height="497" alt="DNS Configuration" src="https://github.com/user-attachments/assets/b102a06d-3174-430c-b4b7-9a0ddfbbe567" />

#### 🏢 Phase 3: Join the Domain

**GUI Method:**

1. Press `Win + R`, type `sysdm.cpl`, press Enter
2. Click **Change** button
3. Select **Domain** radio button
4. Enter: `homelab.local`
5. Click **OK**
6. Enter credentials:
   - **Username:** `HOMELAB\Administrator`
   - **Password:** [Your domain admin password]
7. Click **OK** on success message
8. Click **OK** to restart

**PowerShell Method:**

```powershell
# Join domain via PowerShell
$DomainName = "homelab.local"
$Credential = Get-Credential -Message "Enter domain admin credentials (HOMELAB\Administrator)"

Add-Computer -DomainName $DomainName -Credential $Credential -Restart -Force
```

<img width="885" height="355" alt="Domain Join Dialog" src="https://github.com/user-attachments/assets/fd03877a-4f96-433b-8827-3453a70f1cd0" />

#### ✅ Phase 4: Verify Domain Membership

After restart:

```powershell
# Verify domain membership
Get-ComputerInfo | Select-Object CsDomain, CsDomainRole

# List domain users
Get-ADUser -Filter * -Server homelab.local

# Test domain authentication
Test-ComputerSecureChannel -Server DC01.homelab.local
```

<img width="1045" height="293" alt="Successful Domain Join" src="https://github.com/user-attachments/assets/66390d4a-6fdf-4918-88c6-11a775b79f52" />

**Success Indicators:**

- ✅ Computer shows "homelab.local" as domain
- ✅ Can log in with domain credentials
- ✅ Domain users appear in login screen
- ✅ Can access domain resources

---

## 🔧 Troubleshooting

<details>
<summary><b>VPN Connection Issues</b></summary>

**Problem:** Cannot connect to VPN

**Solutions:**
- Verify client certificate is installed in personal certificate store
- Check if root certificate is properly uploaded to Azure
- Ensure VPN gateway is in "Succeeded" state
- Try downloading and reinstalling VPN client package

```powershell
# Check certificates
Get-ChildItem -Path Cert:\CurrentUser\My | Where-Object {$_.Subject -like "*Azure*"}

# Test VPN connectivity
Test-NetConnection -ComputerName [VPN-Gateway-Public-IP] -Port 443
```

</details>

<details>
<summary><b>DNS Resolution Problems</b></summary>

**Problem:** Cannot resolve homelab.local

**Solutions:**
1. Verify Azure VNet DNS points to DC (10.0.2.10)
2. Manually set DNS on VPN interface
3. Clear DNS cache: `ipconfig /flushdns`
4. Restart VPN connection

```powershell
# Force DNS refresh
ipconfig /flushdns
ipconfig /registerdns

# Test DNS
nslookup homelab.local 10.0.2.10
```

</details>

<details>
<summary><b>Domain Join Failures</b></summary>

**Problem:** Cannot join domain

**Common Causes & Fixes:**

| Error | Cause | Solution |
|-------|-------|----------|
| "Domain cannot be contacted" | DNS issue | Configure DNS to point to DC |
| "Access denied" | Wrong credentials | Use HOMELAB\Administrator |
| "Network path not found" | Firewall blocking | Allow AD ports through NSG |

```powershell
# Test AD connectivity
Test-Connection DC01.homelab.local
nltest /dsgetdc:homelab.local
```

</details>

---

## 📚 Resources

### 📖 Official Documentation

| Resource | Description |
|----------|-------------|
| [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/) | Identity and access management |
| [Active Directory Domain Services](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview) | AD DS overview and guides |
| [Azure VPN Gateway](https://learn.microsoft.com/en-us/azure/vpn-gateway/) | VPN Gateway documentation |
| [PowerShell AD Module](https://learn.microsoft.com/en-us/powershell/module/activedirectory/) | PowerShell cmdlets for AD |

### 🔗 Additional Guides

- 🌐 [Configure Point-to-Site VPN to Azure VNet](https://www.cloudspress.com/how-to-configure-point-to-site-vpn-to-an-azure-vnet/)
- 🛡️ [Azure Network Security Best Practices](https://learn.microsoft.com/en-us/azure/security/fundamentals/network-best-practices)
- 📋 [Active Directory Best Practices](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/plan/security-best-practices/best-practices-for-securing-active-directory)

---

## 📊 Project Statistics

![Network Topology](https://img.shields.io/badge/Components-9-blue)
![Azure Resources](https://img.shields.io/badge/Azure_Resources-3-0078D4)
![Deployment Time](https://img.shields.io/badge/Deployment_Time-2--3_hours-orange)

---

<div align="center">
<sub>Last updated: January 2026</sub>
</div>
