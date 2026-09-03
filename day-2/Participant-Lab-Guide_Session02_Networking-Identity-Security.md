# Participant Lab Guide
## Session 2 — Azure Networking, Identity & Security
### BFL Azure Cloud & Microsoft AI Foundry Training Program

**Author:** Nived Varma

> **Follow-along steps only.** Full explanations and narration are covered live by the trainer — this document exists so you can keep pace without transcribing the screen.
>
> **Naming convention for this lab:** every resource you create is prefixed `trn_<your-participant-id>-<resource-name>`. Replace `<your-participant-id>` with the ID assigned to you at the start of the program (e.g., `trn_bfl07-rg-bfl-hub`). Do **not** reuse the trainer's `trainer_` names — they belong to a separate resource group.

---

### Step 0 — Resource Groups

```powershell
New-AzResourceGroup -Name "trn_<your-participant-id>-rg-bfl-hub" -Location "East US"
New-AzResourceGroup -Name "trn_<your-participant-id>-rg-bfl-spoke" -Location "East US"
```

**Portal steps:**
1. **Resource groups** → **+ Create** → Resource group: `trn_<your-participant-id>-rg-bfl-hub` → Region: **East US** → **Review + create** → **Create**.
2. Repeat for `trn_<your-participant-id>-rg-bfl-spoke`.

---

### Step 1 — Hub VNet

```powershell
New-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-hub-vnet" `
  -ResourceGroupName "trn_<your-participant-id>-rg-bfl-hub" -Location "East US" `
  -AddressPrefix "10.10.0.0/16"

Add-AzVirtualNetworkSubnetConfig -Name "shared-services-subnet" `
  -AddressPrefix "10.10.1.0/24" `
  -VirtualNetwork (Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-hub-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-hub") |
  Set-AzVirtualNetwork
```

**Portal steps:**
1. **Virtual networks** → **+ Create** → Resource group `trn_<your-participant-id>-rg-bfl-hub` → Name `trn_<your-participant-id>-bfl-hub-vnet` → Region **East US**.
2. **IP Addresses** → address space `10.10.0.0/16` → **+ Add a subnet** → Name `shared-services-subnet` → range `10.10.1.0/24` → **Add**.
3. **Review + create** → **Create**.

---

### Step 2 — Spoke VNet with Web and Data Subnets

```powershell
New-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-spoke-vnet" `
  -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" -Location "East US" `
  -AddressPrefix "10.20.0.0/16"

$spoke = Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-spoke-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke"
Add-AzVirtualNetworkSubnetConfig -Name "web-subnet" -AddressPrefix "10.20.1.0/24" -VirtualNetwork $spoke
Add-AzVirtualNetworkSubnetConfig -Name "data-subnet" -AddressPrefix "10.20.2.0/24" -VirtualNetwork $spoke
$spoke | Set-AzVirtualNetwork
```

**Portal steps:**
1. **Virtual networks** → **+ Create** → Resource group `trn_<your-participant-id>-rg-bfl-spoke` → Name `trn_<your-participant-id>-bfl-spoke-vnet` → Region **East US**.
2. **IP Addresses** → address space `10.20.0.0/16` → **+ Add a subnet** twice: `web-subnet` (`10.20.1.0/24`), `data-subnet` (`10.20.2.0/24`).
3. **Review + create** → **Create**.

---

### Step 3 — NSGs and Tier-to-Tier Rules

```powershell
$nsgWeb = New-AzNetworkSecurityGroup -Name "nsg-web" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" -Location "East US"
$nsgWeb | Add-AzNetworkSecurityRuleConfig -Name "Allow-HTTPS-Inbound" -Priority 100 `
  -Direction Inbound -Access Allow -Protocol Tcp -SourceAddressPrefix Internet -SourcePortRange * `
  -DestinationAddressPrefix * -DestinationPortRange 443
$nsgWeb | Set-AzNetworkSecurityGroup

$nsgData = New-AzNetworkSecurityGroup -Name "nsg-data" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" -Location "East US"
$nsgData | Add-AzNetworkSecurityRuleConfig -Name "Allow-SQL-From-Web" -Priority 100 `
  -Direction Inbound -Access Allow -Protocol Tcp -SourceAddressPrefix "10.20.1.0/24" -SourcePortRange * `
  -DestinationAddressPrefix * -DestinationPortRange 1433
$nsgData | Set-AzNetworkSecurityGroup

$spoke = Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-spoke-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke"
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $spoke -Name "web-subnet" -AddressPrefix "10.20.1.0/24" -NetworkSecurityGroup $nsgWeb
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $spoke -Name "data-subnet" -AddressPrefix "10.20.2.0/24" -NetworkSecurityGroup $nsgData
$spoke | Set-AzVirtualNetwork
```

**Portal steps:**
1. **Network security groups** → **+ Create** → RG `trn_<your-participant-id>-rg-bfl-spoke` → Name `nsg-web` → **Create**. Repeat for `nsg-data`.
2. `nsg-web` → **Inbound security rules** → **+ Add** → Source **Any**, Destination port `443`, Protocol **TCP**, Action **Allow**, Priority `100`, Name `Allow-HTTPS-Inbound` → **Add**.
3. `nsg-data` → **Inbound security rules** → **+ Add** → Source **IP Addresses** `10.20.1.0/24`, Destination port `1433`, Protocol **TCP**, Action **Allow**, Priority `100`, Name `Allow-SQL-From-Web` → **Add**.
4. `trn_<your-participant-id>-bfl-spoke-vnet` → **Subnets** → `web-subnet` → NSG = `nsg-web` → **Save**. Repeat for `data-subnet` with `nsg-data`.

---

### Step 4 — VNet Peering (Hub ↔ Spoke)

```powershell
$hub = Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-hub-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-hub"
$spoke = Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-spoke-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke"

Add-AzVirtualNetworkPeering -Name "hub-to-spoke" -VirtualNetwork $hub -RemoteVirtualNetworkId $spoke.Id
Add-AzVirtualNetworkPeering -Name "spoke-to-hub" -VirtualNetwork $spoke -RemoteVirtualNetworkId $hub.Id
```

**Portal steps:**
1. `trn_<your-participant-id>-bfl-hub-vnet` → **Peerings** → **+ Add**.
2. Link name (this → remote): `hub-to-spoke`. Link name (remote → this): `spoke-to-hub`.
3. Remote virtual network: `trn_<your-participant-id>-bfl-spoke-vnet` → **Add**.

---

### Step 5 — Private DNS Zone

```powershell
New-AzPrivateDnsZone -Name "bfl.internal" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-hub"

New-AzPrivateDnsVirtualNetworkLink -ZoneName "bfl.internal" `
  -ResourceGroupName "trn_<your-participant-id>-rg-bfl-hub" -Name "link-to-spoke" `
  -VirtualNetworkId (Get-AzVirtualNetwork -Name "trn_<your-participant-id>-bfl-spoke-vnet" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke").Id
```

**Portal steps:**
1. **Private DNS zones** → **+ Create** → RG `trn_<your-participant-id>-rg-bfl-hub` → Name `bfl.internal` → **Create**.
2. Open the zone → **Virtual network links** → **+ Add** → Link name `link-to-spoke` → Virtual network `trn_<your-participant-id>-bfl-spoke-vnet` → **OK**.

---

### Step 6 — Standard Load Balancer (Web Tier)

```powershell
$pip = New-AzPublicIpAddress -Name "trn_<your-participant-id>-lb-web-pip" `
  -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" -Location "East US" -Sku Standard -AllocationMethod Static

$feConfig = New-AzLoadBalancerFrontendIpConfig -Name "fe-web" -PublicIpAddress $pip
$beConfig = New-AzLoadBalancerBackendAddressPoolConfig -Name "be-web"
$probe = New-AzLoadBalancerProbeConfig -Name "https-probe" -Protocol Tcp -Port 443 -IntervalInSeconds 15 -ProbeCount 2
$rule = New-AzLoadBalancerRuleConfig -Name "lb-rule-https" -FrontendIpConfiguration $feConfig `
  -BackendAddressPool $beConfig -Probe $probe -Protocol Tcp -FrontendPort 443 -BackendPort 443

New-AzLoadBalancer -Name "trn_<your-participant-id>-lb-web" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" `
  -Location "East US" -Sku Standard -FrontendIpConfiguration $feConfig -BackendAddressPool $beConfig -Probe $probe -LoadBalancingRule $rule
```

**Portal steps:**
1. **Load balancers** → **+ Create** → RG `trn_<your-participant-id>-rg-bfl-spoke` → Name `trn_<your-participant-id>-lb-web` → SKU **Standard** → Type **Public**.
2. **Frontend IP configuration** → **+ Add** → Name `fe-web` → new Public IP `trn_<your-participant-id>-lb-web-pip` (Standard) → **Add**.
3. **Backend pools** → **+ Add** → Name `be-web` → associate to `web-subnet` → **Add**.
4. **Health probes** → **+ Add** → Name `https-probe` → TCP `443` → **Add**.
5. **Load balancing rules** → **+ Add** → Name `lb-rule-https` → Frontend `fe-web` → Backend `be-web` → TCP `443` → Probe `https-probe` → **Add**.
6. **Review + create** → **Create**.

---

### Step 7 — Entra ID App Registration & RBAC Role Assignment

```powershell
$app = New-AzADApplication -DisplayName "trn_<your-participant-id>-bfl-loan-webapp"
New-AzADServicePrincipal -ApplicationId $app.AppId

New-AzRoleAssignment -SignInName "<your-upn>@bflcorp.onmicrosoft.com" `
  -RoleDefinitionName "Reader" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke"
```

**Portal steps:**
1. **Microsoft Entra ID** → **App registrations** → **+ New registration** → Name `trn_<your-participant-id>-bfl-loan-webapp` → Single tenant → **Register**.
2. `trn_<your-participant-id>-rg-bfl-spoke` → **Access control (IAM)** → **+ Add** → **Add role assignment** → Role **Reader** → Members: yourself → **Review + assign**.

---

### Step 8 — System-Assigned Managed Identity + Key Vault Access

```powershell
New-AzKeyVault -Name "trn-<your-participant-id>-kv-web" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" `
  -Location "East US" -EnableRbacAuthorization

Update-AzVM -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke" `
  -VM (Get-AzVM -Name "trn_<your-participant-id>-vm-web" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke") -IdentityType SystemAssigned

$vm = Get-AzVM -Name "trn_<your-participant-id>-vm-web" -ResourceGroupName "trn_<your-participant-id>-rg-bfl-spoke"
$kv = Get-AzKeyVault -VaultName "trn-<your-participant-id>-kv-web"

New-AzRoleAssignment -ObjectId $vm.Identity.PrincipalId -RoleDefinitionName "Key Vault Secrets User" -Scope $kv.ResourceId
```

**Portal steps:**
1. **Key vaults** → **+ Create** → RG `trn_<your-participant-id>-rg-bfl-spoke` → Name `trn-<your-participant-id>-kv-web` → **Permission model: Azure RBAC** → **Create**.
2. Open `trn_<your-participant-id>-vm-web` → **Identity** → **System assigned** → **On** → **Save**.
3. `trn-<your-participant-id>-kv-web` → **Access control (IAM)** → **+ Add** → **Add role assignment** → Role **Key Vault Secrets User** → Members → **Managed identity** → select your VM → **Review + assign**.

> **Note:** Key Vault names are globally unique and cannot contain underscores — a hyphenated form is used here (`trn-<id>-kv-web`) instead of the resource-group naming pattern.

---

### Step 9 — Enable Microsoft Defender for Cloud

```powershell
Set-AzSecurityPricing -Name "KeyVaults" -PricingTier "Standard"
Get-AzSecurityTask
```

**Portal steps:**
1. **Microsoft Defender for Cloud** → **Environment settings** → your subscription → **Defender plans** → toggle **Key Vaults** → **On** → **Save**.
2. **Overview** → view your **Secure Score** → open two or three **Recommendations** for your resource group.

---

### Step 10 — Clean-up

```powershell
Remove-AzResourceGroup -Name "trn_<your-participant-id>-rg-bfl-spoke" -Force
Remove-AzResourceGroup -Name "trn_<your-participant-id>-rg-bfl-hub" -Force
```

**Portal steps:**
1. **Resource groups** → open `trn_<your-participant-id>-rg-bfl-spoke` → **Delete resource group** → type the name to confirm → **Delete**.
2. Repeat for `trn_<your-participant-id>-rg-bfl-hub`.
