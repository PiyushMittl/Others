## Azure DNS Zones Explained

### What is a DNS Zone?

A **DNS zone** is a management boundary in Azure that represents a domain (like `contoso.azure.com`). It's a container for DNS records.

### Zone Types

| Type | Description | Example |
|------|-------------|---------|
| **Public Zone** | Accessible from the internet | `contoso.azure.com` |
| **Private Zone** | Only accessible from specified VNets | `internal.company.local` |

### Zone Components

When you create a zone in Azure, you get:

1. **The Zone Resource** - Container for DNS records
2. **Name Servers** - 4 Azure DNS servers assigned automatically
3. **SOA Record** - Start of Authority (auto-created)
4. **NS Records** - Name server records (auto-created)

### Creating a Zone in Azure

```powershell
# Create a public DNS zone
az network dns zone create \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos

# Azure automatically assigns 4 name servers:
# ns1-01.azure-dns.com
# ns2-01.azure-dns.net
# ns3-01.azure-dns.org
# ns4-01.azure-dns.info
```

**What happens behind the scenes:**

```
Azure DNS Service:
  1. Allocates 4 geographically distributed name servers
  2. Creates SOA record (metadata about the zone)
  3. Creates NS records pointing to the 4 name servers
  4. Synchronizes across Azure's global DNS infrastructure
  5. Returns: Zone ID and name servers
```

---

## Delegation vs Flat Structure

You have two architectural choices for organizing DNS records:

### Option 1: Flat Structure (Recommended)

**All records in a single zone:**

```
Zone: contoso.azure.com (in Global-DNS-contosos subscription)
├─ A: order.eus → 13.77.50.25
├─ A: order.ne → 52.178.17.25
├─ A: order.eus2 → 20.44.10.25
├─ A: order.we → 52.166.34.25
├─ A: app.eus → 13.77.50.26
└─ CNAME: www → order.eus
```

**Advantages:**
- ✅ Simple management (one zone)
- ✅ Fewer DNS lookups (faster)
- ✅ Easier to audit/monitor
- ✅ Lower cost (one zone instead of many)

**Disadvantages:**
- ❌ All eggs in one basket
- ❌ Single point of failure (mitigated by Azure SLA)
- ❌ Requires cross-subscription RBAC if deployments are regional

**Implementation:**

```powershell
# Create all records in one zone
$regions = @(
    @{Name="ccy"; IP="13.77.50.25"},
    @{Name="ne"; IP="52.178.17.25"},
    @{Name="eus2"; IP="20.44.10.25"}
)

foreach ($region in $regions) {
    az network dns record-set a add-record \
      --resource-group global-azuredns-contosos \
      --zone-name contoso.azure.com \
      --record-set-name "order.$($region.Name)" \
      --ipv4-address $region.IP \
      --subscription Global-DNS-contosos
}
```

**DNS Resolution (Flat):**
```
Query: order.ne.contoso.azure.com
  ↓ Root → .com → azure.com → contoso.azure.com
  ↓ Returns: 52.178.17.25
  ✅ 4 hops total
```

---

### Option 2: Delegated Structure

**Separate zones for each region:**

```
Zone: contoso.azure.com (Global)
├─ NS: ccy → ns1-09.azure-dns.com (delegates to ccy zone)
├─ NS: ne → ns1-17.azure-dns.com (delegates to ne zone)
└─ NS: eus2 → ns1-23.azure-dns.com (delegates to eus2 zone)

Zone: eus.contoso.azure.com (East US subscription)
├─ A: order → 13.77.50.25
└─ A: app → 13.77.50.26

Zone: ne.contoso.azure.com (North Europe subscription)
└─ A: order → 52.178.17.25

Zone: eus2.contoso.azure.com (East US 2 subscription)
└─ A: order → 20.44.10.25
```

**Advantages:**
- ✅ Regional autonomy (each region manages own zone)
- ✅ Isolation (issues in one region don't affect others)
- ✅ No cross-subscription RBAC needed
- ✅ Clear ownership boundaries

**Disadvantages:**
- ❌ More complex to manage
- ❌ Additional DNS lookup (slower)
- ❌ Higher cost (multiple zones)
- ❌ More moving parts to monitor

**Implementation:**

```powershell
# Step 1: Create regional zone
az network dns zone create \
  --name ne.contoso.azure.com \
  --resource-group northeurope-azuredns-contosos \
  --subscription northeurope-PROD-contosos-Buildout1

# Step 2: Get regional zone's name servers
$regionalNS = az network dns zone show \
  --name ne.contoso.azure.com \
  --resource-group northeurope-azuredns-contosos \
  --subscription northeurope-PROD-contosos-Buildout1 \
  --query nameServers -o tsv

# Step 3: Create NS delegation in parent zone
az network dns record-set ns create \
  --resource-group global-azuredns-contosos \
  --zone-name contoso.azure.com \
  --name ne \
  --ttl 3600 \
  --subscription Global-DNS-contosos

foreach ($ns in $regionalNS) {
    az network dns record-set ns add-record \
      --resource-group global-azuredns-contosos \
      --zone-name contoso.azure.com \
      --record-set-name ne \
      --nsdname $ns \
      --subscription Global-DNS-contosos
}

# Step 4: Create A record in regional zone
az network dns record-set a add-record \
  --resource-group northeurope-azuredns-contosos \
  --zone-name ne.contoso.azure.com \
  --record-set-name order \
  --ipv4-address 52.178.17.25 \
  --subscription northeurope-PROD-contosos-Buildout1
```

**DNS Resolution (Delegated):**
```
Query: order.ne.contoso.azure.com
  ↓ Root → .com → azure.com → contoso.azure.com
  ↓ contoso.azure.com: "Ask ne.contoso.azure.com servers"
  ↓ ne.contoso.azure.com: Returns: 52.178.17.25
  ✅ 5 hops total (one extra delegation)
```

---

## Cross-Subscription DNS Management

### The Challenge

In enterprise environments, you often have:
- **Global DNS subscription** - Centralized DNS management
- **Regional subscriptions** - Application deployments per region

**Problem:** How do regional deployments create DNS records in the global zone?

**Solution:** Cross-subscription RBAC!

### Architecture

```
┌─────────────────────────────────────────────────────┐
│ Subscription: Global-DNS-contosos                │
│ ┌─────────────────────────────────────────────┐     │
│ │ DNS Zone: contoso.azure.com              │     │
│ │ ├─ A: order.ne → 52.x.x.x                   │     │
│ │ ├─ A: order.eus → 13.x.x.x                  │     │
│ │ └─ A: order.eus2 → 20.x.x.x                 │     │
│ └─────────────────────────────────────────────┘     │
│                       ↑                              │
│                       │ DNS Zone Contributor         │
│                       │ (RBAC Permission)            │
└───────────────────────┼──────────────────────────────┘
                        │
        ┌───────────────┼───────────────┐
        │               │               │
        ↓               ↓               ↓
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ North Europe  │ │ Central US    │ │ East US 2     │
│ Subscription  │ │ Subscription  │ │ Subscription  │
│               │ │               │ │               │
│ Identity:     │ │ Identity:     │ │ Identity:     │
│ ev2-ne        │ │ ev2-ccy       │ │ ev2-eus2      │
│               │ │               │ │               │
│ Creates A     │ │ Creates A     │ │ Creates A     │
│ record via    │ │ record via    │ │ record via    │
│ Ev2 deployment│ │ Ev2 deployment│ │ Ev2 deployment│
└───────────────┘ └───────────────┘ └───────────────┘
```

### Implementation Steps

#### Step 1: Create Global DNS Zone

```powershell
# Run this once in the global subscription
az network dns zone create \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos
```

#### Step 2: Get Regional Deployment Identities

```powershell
# For each region, find the deployment identity (Ev2, managed identity, service principal)
$regions = @(
    @{Name="North Europe"; Sub="northeurope-PROD-contosos-Buildout1"; Identity="ev2Identity-northeurope"; RG="app-prod-northeurope"},
    @{Name="Central US EUAP"; Sub="centraluseuap-PROD-contosos"; Identity="ev2Identity-ccy"; RG="app-prod-centraluseuap"}
)

foreach ($region in $regions) {
    $principalId = az identity show \
      --name $region.Identity \
      --resource-group $region.RG \
      --subscription $region.Sub \
      --query principalId -o tsv

    Write-Host "$($region.Name): $principalId"
}
```

#### Step 3: Grant Cross-Subscription Permissions

```powershell
# Get global DNS zone resource ID
$zoneId = az network dns zone show \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos \
  --query id -o tsv

# Grant DNS Zone Contributor role to each regional identity
foreach ($region in $regions) {
    $principalId = az identity show \
      --name $region.Identity \
      --resource-group $region.RG \
      --subscription $region.Sub \
      --query principalId -o tsv

    az role assignment create \
      --assignee $principalId \
      --role "DNS Zone Contributor" \
      --scope $zoneId

    Write-Host "✓ Granted $($region.Name) access to global DNS zone"
}
```

#### Step 4: Create Records from Regional Subscription

```powershell
# Now, from any regional subscription, you can create records!

# Set context to regional subscription
az account set --subscription "northeurope-PROD-contosos-Buildout1"

# Create A record in global zone (note: --subscription points to global)
az network dns record-set a add-record \
  --resource-group global-azuredns-contosos \
  --zone-name contoso.azure.com \
  --record-set-name order.ne \
  --ipv4-address 52.178.17.25 \
  --subscription Global-DNS-contosos

# ✅ Record created successfully from regional sub to global zone!
```

### RBAC Roles Explained

| Role | Permissions | Recommended For |
|------|-------------|-----------------|
| **DNS Zone Contributor** | Full access to DNS zones and all records | Deployment identities, automation |
| **Reader** | View zones and records only | Monitoring, auditing |
| **Network Contributor** | DNS + networking resources | Too broad, avoid |
| **Owner** | Full control + access management | Zone administrators only |

---

## Practical Implementation Guide

### Complete Example: Contoso Application DNS Setup

Let's implement a real-world scenario:

**Requirements:**
- Global DNS zone in `Global-DNS-contosos` subscription
- 3 regional deployments: North Europe, Central US EUAP, East US 2
- Each deployment creates regional A records
- Application Gateway provides the public IPs

#### Complete PowerShell Script

```powershell
# Save as: Setup-Contoso-DNS.ps1

param(
    [Parameter(Mandatory=$false)]
    [switch]$CreateZone = $false,

    [Parameter(Mandatory=$false)]
    [switch]$GrantPermissions = $false,

    [Parameter(Mandatory=$false)]
    [switch]$CreateRecords = $false,

    [Parameter(Mandatory=$false)]
    [switch]$Verify = $false,

    [Parameter(Mandatory=$false)]
    [switch]$All = $false
)

$ErrorActionPreference = "Stop"

# Configuration
$globalSub = "Global-DNS-contosos"
$globalRG = "global-azuredns-contosos"
$zoneName = "contoso.azure.com"

$regions = @(
    @{
        Name = "East US"
        Short = "eus"
        Subscription = "eastus-PROD-contosos"
        ResourceGroup = "app-prod-eastus"
        Identity = "ev2Identity-eus"
        AppGatewayIP = "13.77.50.25"
    },
    @{
        Name = "North Europe"
        Short = "ne"
        Subscription = "northeurope-PROD-contosos-Buildout1"
        ResourceGroup = "app-prod-northeurope"
        Identity = "ev2Identity-northeurope"
        AppGatewayIP = "52.178.17.25"
    },
    @{
        Name = "East US 2"
        Short = "eus2"
        Subscription = "eastus2-PROD-contosos-Buildout1"
        ResourceGroup = "app-prod-eastus2"
        Identity = "ev2Identity-eus2"
        AppGatewayIP = "20.44.10.25"
    }
)

Write-Host "=== Contoso Application DNS Setup ===" -ForegroundColor Cyan
Write-Host "Zone: $zoneName" -ForegroundColor Cyan
Write-Host "Global Subscription: $globalSub`n" -ForegroundColor Cyan

# Function: Create Global DNS Zone
function Create-GlobalZone {
    Write-Host "[1] Creating global DNS zone..." -ForegroundColor Yellow

    $zoneExists = az network dns zone show \
      --name $zoneName \
      --resource-group $globalRG \
      --subscription $globalSub 2>$null

    if ($zoneExists) {
        Write-Host "✓ Zone already exists: $zoneName" -ForegroundColor Green
    } else {
        az network dns zone create \
          --name $zoneName \
          --resource-group $globalRG \
          --subscription $globalSub | Out-Null

        Write-Host "✓ Created zone: $zoneName" -ForegroundColor Green
    }

    # Get name servers
    $nameServers = az network dns zone show \
      --name $zoneName \
      --resource-group $globalRG \
      --subscription $globalSub \
      --query nameServers -o tsv

    Write-Host "`nName servers for $zoneName" -ForegroundColor Cyan
    $nameServers | ForEach-Object { Write-Host "  - $_" }

    Write-Host "`n⚠️  ACTION REQUIRED: Update NS records in azure.com to point to these name servers" -ForegroundColor Yellow
}

# Function: Grant Cross-Subscription Permissions
function Grant-Permissions {
    Write-Host "`n[2] Granting cross-subscription permissions..." -ForegroundColor Yellow

    $zoneId = az network dns zone show \
      --name $zoneName \
      --resource-group $globalRG \
      --subscription $globalSub \
      --query id -o tsv

    foreach ($region in $regions) {
        Write-Host "`nProcessing $($region.Name)..." -ForegroundColor Cyan

        # Get identity principal ID
        $principalId = az identity show \
          --name $region.Identity \
          --resource-group $region.ResourceGroup \
          --subscription $region.Subscription \
          --query principalId -o tsv 2>$null

        if (!$principalId) {
            Write-Host "  ⚠️  Identity not found: $($region.Identity)" -ForegroundColor Yellow
            continue
        }

        # Check if role assignment already exists
        $existingRole = az role assignment list \
          --assignee $principalId \
          --scope $zoneId \
          --role "DNS Zone Contributor" 2>$null | ConvertFrom-Json

        if ($existingRole) {
            Write-Host "  ✓ Permission already exists" -ForegroundColor Green
        } else {
            az role assignment create \
              --assignee $principalId \
              --role "DNS Zone Contributor" \
              --scope $zoneId | Out-Null

            Write-Host "  ✓ Granted DNS Zone Contributor role" -ForegroundColor Green
        }

        Write-Host "  Identity: $principalId"
    }
}

# Function: Create DNS Records
function Create-Records {
    Write-Host "`n[3] Creating regional A records..." -ForegroundColor Yellow

    foreach ($region in $regions) {
        Write-Host "`nCreating record for $($region.Name)..." -ForegroundColor Cyan

        $recordName = "order.$($region.Short)"

        # Check if record exists
        $existingRecord = az network dns record-set a show \
          --resource-group $globalRG \
          --zone-name $zoneName \
          --name $recordName \
          --subscription $globalSub 2>$null | ConvertFrom-Json

        if ($existingRecord) {
            # Update existing record
            az network dns record-set a update \
              --resource-group $globalRG \
              --zone-name $zoneName \
              --name $recordName \
              --set aRecords[0].ipv4Address=$($region.AppGatewayIP) \
              --subscription $globalSub | Out-Null

            Write-Host "  ✓ Updated: $recordName.$zoneName → $($region.AppGatewayIP)" -ForegroundColor Green
        } else {
            # Create new record
            az network dns record-set a add-record \
              --resource-group $globalRG \
              --zone-name $zoneName \
              --record-set-name $recordName \
              --ipv4-address $region.AppGatewayIP \
              --subscription $globalSub | Out-Null

            Write-Host "  ✓ Created: $recordName.$zoneName → $($region.AppGatewayIP)" -ForegroundColor Green
        }
    }
}

# Function: Verify Setup
function Verify-Setup {
    Write-Host "`n[4] Verifying DNS setup..." -ForegroundColor Yellow

    # Get zone name servers
    $nameServers = az network dns zone show \
      --name $zoneName \
      --resource-group $globalRG \
      --subscription $globalSub \
      --query nameServers -o tsv

    $firstNS = $nameServers[0]

    foreach ($region in $regions) {
        $fqdn = "order.$($region.Short).$zoneName"
        Write-Host "`nTesting: $fqdn" -ForegroundColor Cyan

        # Query against zone's name servers directly
        $result = nslookup $fqdn $firstNS 2>&1

        if ($result -match $region.AppGatewayIP) {
            Write-Host "  ✅ Resolved to: $($region.AppGatewayIP)" -ForegroundColor Green
        } else {
            Write-Host "  ❌ Resolution failed" -ForegroundColor Red
            Write-Host "  Response: $result"
        }
    }

    # List all A records
    Write-Host "`nAll A records in zone:" -ForegroundColor Cyan
    az network dns record-set list \
      --resource-group $globalRG \
      --zone-name $zoneName \
      --subscription $globalSub \
      --query "[?type=='Microsoft.Network/dnszones/A'].{Name:name, IP:aRecords[0].ipv4Address, TTL:ttl}" \
      --output table
}

# Main execution
if ($All -or $CreateZone) {
    Create-GlobalZone
}

if ($All -or $GrantPermissions) {
    Grant-Permissions
}

if ($All -or $CreateRecords) {
    Create-Records
}

if ($All -or $Verify) {
    Verify-Setup
}

if (!$All -and !$CreateZone -and !$GrantPermissions -and !$CreateRecords -and !$Verify) {
    Write-Host "Usage:"
    Write-Host "  .\Setup-Contoso-DNS.ps1 -All                # Run all steps"
    Write-Host "  .\Setup-Contoso-DNS.ps1 -CreateZone         # Step 1 only"
    Write-Host "  .\Setup-Contoso-DNS.ps1 -GrantPermissions   # Step 2 only"
    Write-Host "  .\Setup-Contoso-DNS.ps1 -CreateRecords      # Step 3 only"
    Write-Host "  .\Setup-Contoso-DNS.ps1 -Verify             # Step 4 only"
}

Write-Host "`n=== Setup Complete ===" -ForegroundColor Green
```

#### Usage Examples

```powershell
# Run all setup steps
.\Setup-Contoso-DNS.ps1 -All

# Run steps individually
.\Setup-Contoso-DNS.ps1 -CreateZone
.\Setup-Contoso-DNS.ps1 -GrantPermissions
.\Setup-Contoso-DNS.ps1 -CreateRecords
.\Setup-Contoso-DNS.ps1 -Verify

# Verify only (after deployment)
.\Setup-Contoso-DNS.ps1 -Verify
```

---

## Best Practices

### 1. Zone Organization

✅ **DO:**
- Use a centralized global DNS zone for cross-region records
- Group related records in the same zone
- Use consistent naming conventions

❌ **DON'T:**
- Create separate zones for each environment in production (use labels instead)
- Mix internal and external DNS in the same zone

### 2. TTL (Time To Live) Strategy

```powershell
# For production endpoints
TTL = 3600 (1 hour)    # Balance between caching and flexibility

# For active deployments/migrations
TTL = 300 (5 minutes)  # Faster propagation

# For stable infrastructure
TTL = 86400 (24 hours) # Maximum caching
```

### 3. Security

```powershell
# Principle of least privilege
az role assignment create \
  --assignee <IDENTITY> \
  --role "DNS Zone Contributor" \        # ✅ Specific role
  --scope <ZONE_RESOURCE_ID>             # ✅ Zone-level scope

# ❌ DON'T grant at subscription level
# ❌ DON'T use "Contributor" or "Owner" roles
```

### 4. Monitoring

**Set up alerts for:**
- DNS query rate spikes
- Zone modification events
- Failed DNS resolutions

```powershell
# Enable diagnostic logs
az monitor diagnostic-settings create \
  --name "dns-zone-logs" \
  --resource <ZONE_RESOURCE_ID> \
  --logs '[{"category": "QueryCapture", "enabled": true}]' \
  --workspace <LOG_ANALYTICS_WORKSPACE_ID>
```

### 5. Disaster Recovery

**Always maintain:**
1. Export of all DNS records
2. Documentation of name server delegation
3. Backup zone in secondary subscription (optional)

```powershell
# Export DNS zone
az network dns zone export \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --file-name "contoso-backup-$(Get-Date -Format 'yyyyMMdd').txt"
```

---

## Troubleshooting

### Common Issues

#### Issue 1: DNS Not Resolving

**Symptoms:** `nslookup` returns `NXDOMAIN` or no records

**Diagnosis:**

```powershell
# Step 1: Check zone exists
az network dns zone show \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos

# Step 2: Check record exists
az network dns record-set a show \
  --name order.ne \
  --zone-name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos

# Step 3: Query zone name servers directly
$ns = az network dns zone show \
  --name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos \
  --query nameServers[0] -o tsv

nslookup order.ne.contoso.azure.com $ns
```

**Common Causes:**
- ❌ NS records in parent zone not updated
- ❌ Record doesn't exist
- ❌ Wrong zone queried

---

#### Issue 2: Cross-Subscription Permission Denied

**Symptoms:** `AuthorizationFailed` when creating records

**Diagnosis:**

```powershell
# Check role assignments
az role assignment list \
  --assignee <IDENTITY_PRINCIPAL_ID> \
  --scope "/subscriptions/GLOBAL_SUB/resourceGroups/global-azuredns-contosos/providers/Microsoft.Network/dnszones/contoso.azure.com" \
  --query "[].{Role:roleDefinitionName, Scope:scope}" \
  --output table
```

**Solution:**

```powershell
# Grant DNS Zone Contributor
az role assignment create \
  --assignee <IDENTITY_PRINCIPAL_ID> \
  --role "DNS Zone Contributor" \
  --scope <ZONE_RESOURCE_ID>

# Wait 5-10 minutes for RBAC propagation
```

---

#### Issue 3: Stale DNS Cache

**Symptoms:** DNS returns old IP address

**Diagnosis:**

```powershell
# Check current record
az network dns record-set a show \
  --name order.ne \
  --zone-name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos \
  --query "aRecords[0].ipv4Address"

# Check TTL
az network dns record-set a show \
  --name order.ne \
  --zone-name contoso.azure.com \
  --resource-group global-azuredns-contosos \
  --subscription Global-DNS-contosos \
  --query "ttl"
```

**Solution:**

```powershell
# Wait for TTL to expire, or:

# Flush local DNS cache (Windows)
ipconfig /flushdns

# Flush local DNS cache (Linux)
sudo systemd-resolve --flush-caches

# Query with no-cache
nslookup -type=A order.ne.contoso.azure.com 8.8.8.8
```

---

#### Issue 4: Wrong Zone Being Queried

**Symptoms:** DNS resolves to wrong IP or old regional zone

**Diagnosis:**

```powershell
# Check current NS records in parent zone
nslookup -type=NS contoso.azure.com

# Expected: Should show global zone name servers (ns1-01.azure-dns.com)
# Problem: Shows old regional zone name servers

# Verify parent zone delegation
az network dns record-set ns show \
  --name contoso \
  --zone-name azure.com \
  --resource-group <PARENT_RG> \
  --subscription <PARENT_SUB>
```

**Solution:**

```powershell
# Update NS records in parent zone to point to global zone
# See "How to Update Domain Registrar" section above
```

---

## Summary

### Key Takeaways

1. **DNS is hierarchical** - Each level delegates to the next
2. **Zones are management boundaries** - Group related records together
3. **Flat structure is simpler** - Recommended for most scenarios
4. **Cross-subscription works** - Use RBAC for centralized DNS with distributed deployments
5. **Name server delegation is critical** - The parent zone must point to your zone's name servers

### Your DNS Journey

```
Start: order.eus.contoso.azure.com (in browser)
  ↓
Root DNS → .com TLD → azure.com → contoso.azure.com
  ↓
End: 13.77.50.25 (your App Gateway)
  ↓
App Gateway → AKS Cluster → Your Application
  ↓
Response back to user
```

**Total time:** ~30ms (first query) → <1ms (cached)

---

## Additional Resources

### Documentation
- [Azure DNS Overview](https://docs.microsoft.com/azure/dns/dns-overview)
- [DNS Zones and Records](https://docs.microsoft.com/azure/dns/dns-zones-records)
- [Azure RBAC](https://docs.microsoft.com/azure/role-based-access-control/overview)

### Tools
- **Azure CLI:** `az network dns` commands
- **PowerShell:** Azure DNS management scripts
- **DNS Tools:** `nslookup`, `dig`, `host`

### Monitoring
- Azure Monitor for DNS query metrics
- Log Analytics for DNS query logs
- Azure Advisor for DNS best practices

---

## About This Guide

**Author:** Technical Documentation  
**Version:** 1.0  
**Last Updated:** 2026-05-06  
**Target Audience:** Cloud engineers, DevOps professionals, DNS administrators

**Feedback:** Please submit issues or improvements via your team's documentation process.
