# Understanding DNS Hierarchy: From Root to Application Endpoint

**A Practical Guide to DNS Resolution and Azure DNS Management**

---

## Table of Contents

1. [Introduction](#introduction)
2. [DNS Hierarchy Basics](#dns-hierarchy-basics)
3. [How DNS Resolution Works](#how-dns-resolution-works)
4. [Azure DNS Zones Explained](#azure-dns-zones-explained)
5. [Delegation vs Flat Structure](#delegation-vs-flat-structure)
6. [Cross-Subscription DNS Management](#cross-subscription-dns-management)
7. [Practical Implementation Guide](#practical-implementation-guide)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Introduction

When you type `order.eus.contoso.azure.com` into your browser, a complex but elegant series of DNS lookups happens behind the scenes. This guide will demystify that process and show you how to implement it in Azure DNS.

### What You'll Learn

- How DNS hierarchy works from root (`.`) to your application endpoint
- The difference between DNS delegation and flat record structures
- How to manage DNS across multiple Azure subscriptions
- Practical Azure CLI commands for DNS management
- Best practices for enterprise DNS architecture

---

## DNS Hierarchy Basics

### The DNS Tree Structure

DNS (Domain Name System) is organized as a hierarchical tree, much like a file system:

![DNS Hierarichy Diagram](https://github.com/PiyushMittl/Others/blob/main/azure-dns-blog/images/1.dns-parent-chield-hierarichy.png)

**Full FQDNs:**
- `order.eus.contoso.azure.com`
- `payment.eus.contoso.azure.com`
- `order.sea.contoso.azure.com`
- `payment.sea.contoso.azure.com`

### Breaking Down a Fully Qualified Domain Name (FQDN)

Let's dissect `order.eus.contoso.azure.com`:

| Level | Component | Type | Managed By |
|-------|-----------|------|------------|
| 5 | `order` | A record name | You |
| 4 | `eus` | Sub-subdomain (region) | You |
| 3 | `contoso` | Subdomain | You (your zone) |
| 2 | `azure` | Second-level domain | Microsoft |
| 1 | `com` | Top-level domain (TLD) | Verisign |
| 0 | `.` (implicit) | Root | ICANN/IANA |

**Reading right-to-left:** Each component is more specific than the one to its right.

---

## How DNS Resolution Works

### The Complete Journey

When you query `order.eus.contoso.azure.com`, here's the step-by-step resolution process:

#### Step 1: Root DNS Servers

```
Client Query: order.eus.contoso.azure.com
    ↓
Root DNS Server (.):
    "I don't know about order.eus.contoso.azure.com,
     but I know who handles .com domains"
    ↓
Response: Here are the .com TLD name servers:
    - a.gtld-servers.net
    - b.gtld-servers.net
    - ... (13 root servers)
```

#### Step 2: TLD Name Servers (.com)

```
Client Query: order.eus.contoso.azure.com
    ↓
.com TLD Server (a.gtld-servers.net):
    "I don't know about order.eus.contoso.azure.com,
     but I know who handles azure.com"
    ↓
Looks up: NS record for "azure"
    ↓
Response: Here are the azure.com name servers:
    - ns1-microsoft.azure-dns.com
    - ns2-microsoft.azure-dns.net
    - ...
```

#### Step 3: Second-Level Domain (azure.com)

```
Client Query: order.eus.contoso.azure.com
    ↓
azure.com Server (ns1-microsoft.azure-dns.com):
    "I don't know about order.eus.contoso.azure.com,
     but I have an NS record for contoso"
    ↓
Looks up: NS record for "contoso"
    ↓
Response: Here are the contoso.azure.com name servers:
    - ns1-01.azure-dns.com
    - ns2-01.azure-dns.net
    - ns3-01.azure-dns.org
    - ns4-01.azure-dns.info
```

**🔑 Critical Point:** This is where delegation happens! The `azure.com` zone contains an NS (Name Server) record that points to YOUR zone's name servers.

#### Step 4: Your DNS Zone (contoso.azure.com)

```
Client Query: order.eus.contoso.azure.com
    ↓
contoso.azure.com Server (ns1-01.azure-dns.com):
    "Let me check my records..."
    ↓
Looks up: NS record for "eus"
    ↓
Found: NS record pointing to eus.contoso.azure.com name servers
    - ns1-09.azure-dns.com
    - ns2-09.azure-dns.net
    - ns3-09.azure-dns.org
    - ns4-09.azure-dns.info
    ↓
Response: "Ask ns1-09.azure-dns.com (eus zone name servers)"
```

**🔑 Another Delegation:** The `contoso.azure.com` zone contains an NS record for "eus" that delegates to the regional zone.

#### Step 5: Regional DNS Zone (eus.contoso.azure.com)

```
Client Query: order.eus.contoso.azure.com
    ↓
eus.contoso.azure.com Server (ns1-09.azure-dns.com):
    "Let me check my records..."
    ↓
Looks up: A record for "order"
    ↓
Found: A record = 13.77.50.25
    ↓
Response: 13.77.50.25
    ✅ RESOLUTION COMPLETE
```

#### Step 6: Connection Established

```
Client now knows: order.eus.contoso.azure.com = 13.77.50.25
    ↓
TCP/HTTPS connection to 13.77.50.25
    ↓
Application Gateway receives request
    ↓
Routes to backend (AKS cluster)
    ↓
Response returned to client
```

### Visual Timeline

```
Time  | Action                                    | Server
------|-------------------------------------------|------------------
T+0ms | Query: order.eus.contoso.azure.com     | Local DNS Resolver
T+5ms | → Root DNS                                | a.root-servers.net
T+8ms | ← .com TLD servers                        | 
T+10ms| → .com TLD                                | a.gtld-servers.net
T+15ms| ← azure.com servers                       |
T+18ms| → azure.com                               | ns1-microsoft...
T+22ms| ← contoso.azure.com servers            |
T+25ms| → contoso.azure.com                    | ns1-01.azure-dns.com
T+28ms| ← eus.contoso.azure.com servers        |
T+30ms| → eus.contoso.azure.com                | ns1-09.azure-dns.com
T+33ms| ← IP: 13.77.50.25                         |
T+35ms| ✅ Resolution complete                     |
```

**Total time:** ~35ms (with caching, subsequent queries: <1ms)

---

## The Critical Rule: NS Delegation

### ⚠️ Understanding Name Server Delegation

**THE GOLDEN RULE:** Whenever you create a child DNS zone (subdomain), you **MUST** add its name servers to the parent zone as NS records.

### Why This Is Absolutely Required

![DNS NS Hierarichy Diagram](https://github.com/PiyushMittl/Others/blob/main/azure-dns-blog/images/1.dns-parent-chield-hierarichy.png)

```
Child Zone Created → Azure Assigns Name Servers → You MUST Add to Parent

Without this step:
❌ DNS queries stop at the parent zone
❌ The child zone is NEVER queried
❌ Your records exist but are "orphaned" (unreachable)
❌ Users get NXDOMAIN errors
```

### The Step-by-Step Process

#### Step 1: Create Child Zone (Azure Assigns Name Servers)

```powershell
# Create the child zone
az network dns zone create \
  --name eus.contoso.azure.com \
  --resource-group regional-dns-rg \
  --subscription RegionalSubscription

# Azure automatically assigns 4 name servers:
# Output:
# {
#   "nameServers": [
#     "ns1-09.azure-dns.com",
#     "ns2-09.azure-dns.net",
#     "ns3-09.azure-dns.org",
#     "ns4-09.azure-dns.info"
#   ]
# }
```

**What happened:**
- ✅ Child zone `eus.contoso.azure.com` created
- ✅ Azure assigned 4 unique name servers
- ❌ **BUT: Parent zone doesn't know about it yet!**

#### Step 2: Get the Child Zone's Name Servers

```powershell
# Retrieve the name servers
$childNameServers = az network dns zone show \
  --name eus.contoso.azure.com \
  --resource-group regional-dns-rg \
  --subscription RegionalSubscription \
  --query nameServers -o tsv

# Output:
# ns1-09.azure-dns.com
# ns2-09.azure-dns.net
# ns3-09.azure-dns.org
# ns4-09.azure-dns.info

Write-Host "Child zone name servers:"
$childNameServers | ForEach-Object { Write-Host "  - $_" }
```

#### Step 3: Add NS Records to Parent Zone (THE CRITICAL STEP!)

```powershell
# In the PARENT zone (contoso.azure.com), create NS record for "eus"
az network dns record-set ns create \
  --resource-group global-dns-rg \
  --zone-name contoso.azure.com \
  --name eus \
  --ttl 3600 \
  --subscription GlobalSubscription

# Add EACH of the child zone's name servers
foreach ($ns in $childNameServers) {
    az network dns record-set ns add-record \
      --resource-group global-dns-rg \
      --zone-name contoso.azure.com \
      --record-set-name eus \
      --nsdname $ns \
      --subscription GlobalSubscription

    Write-Host "✓ Added NS: $ns"
}
```

**What this does:**

```
Parent Zone: contoso.azure.com
Now contains:
  NS record: "eus" →
    - ns1-09.azure-dns.com
    - ns2-09.azure-dns.net
    - ns3-09.azure-dns.org
    - ns4-09.azure-dns.info
```

**Meaning:** "For any queries about `*.eus.contoso.azure.com`, go ask these name servers"

#### Step 4: Verify Delegation Works

```powershell
# Test NS record exists in parent
nslookup -type=NS eus.contoso.azure.com

# Expected output:
# eus.contoso.azure.com nameserver = ns1-09.azure-dns.com
# eus.contoso.azure.com nameserver = ns2-09.azure-dns.net
# ... (all 4 name servers)

# Test A record resolution through the delegation
nslookup order.eus.contoso.azure.com

# Should resolve correctly!
```

### Visual: Before vs After

#### ❌ BEFORE (Delegation Missing)

```
Query: order.eus.contoso.azure.com
  ↓
contoso.azure.com zone:
  - Checks for "order.eus" record → NOT FOUND
  - Checks for "eus" NS record → NOT FOUND ❌
  - Returns: NXDOMAIN (no such domain)

Child zone (eus.contoso.azure.com):
  - Contains: A record "order" → 13.77.50.25
  - But NEVER QUERIED (orphaned!)

Result: ❌ DNS resolution fails
```

#### ✅ AFTER (Delegation Added)

```
Query: order.eus.contoso.azure.com
  ↓
contoso.azure.com zone:
  - Checks for "order.eus" record → NOT FOUND
  - Checks for "eus" NS record → FOUND! ✅
  - Returns: "Ask ns1-09.azure-dns.com (child zone name servers)"
  ↓
eus.contoso.azure.com zone (ns1-09.azure-dns.com):
  - Checks for "order" record → FOUND!
  - Returns: 13.77.50.25

Result: ✅ DNS resolution succeeds
```

### Real-World Example: Your Contoso Application Setup

**Scenario:** You want to create separate zones for each region.

```
Global Zone: contoso.azure.com
  ↓ (Must contain NS delegations for each region)

Regional Zones:
  - eus.contoso.azure.com (East US)
  - ne.contoso.azure.com (North Europe)
  - eus2.contoso.azure.com (East US 2)
```

**Complete Script:**

```powershell
# Define regions
$regions = @(
    @{Name="eus"; Sub="eastus-PROD-contosos"},
    @{Name="ne"; Sub="northeurope-PROD-contosos-Buildout1"},
    @{Name="eus2"; Sub="eastus2-PROD-contosos-Buildout1"}
)

$parentZone = "contoso.azure.com"
$parentSub = "Global-DNS-contosos"
$parentRG = "global-azuredns-contosos"

foreach ($region in $regions) {
    Write-Host "`n=== Processing region: $($region.Name) ===" -ForegroundColor Cyan

    # Step 1: Create regional zone
    Write-Host "[1] Creating child zone..." -ForegroundColor Yellow
    az network dns zone create \
      --name "$($region.Name).$parentZone" \
      --resource-group "$($region.Name)-azuredns-contosos" \
      --subscription $region.Sub

    # Step 2: Get child zone name servers
    Write-Host "[2] Getting child zone name servers..." -ForegroundColor Yellow
    $childNS = az network dns zone show \
      --name "$($region.Name).$parentZone" \
      --resource-group "$($region.Name)-azuredns-contosos" \
      --subscription $region.Sub \
      --query nameServers -o tsv

    Write-Host "Child zone name servers:" -ForegroundColor Green
    $childNS | ForEach-Object { Write-Host "  - $_" }

    # Step 3: Create NS delegation in parent zone
    Write-Host "[3] Creating NS delegation in parent zone..." -ForegroundColor Yellow

    # Create NS record set
    az network dns record-set ns create \
      --resource-group $parentRG \
      --zone-name $parentZone \
      --name $region.Name \
      --ttl 3600 \
      --subscription $parentSub 2>$null

    # Add each name server
    foreach ($ns in $childNS) {
        az network dns record-set ns add-record \
          --resource-group $parentRG \
          --zone-name $parentZone \
          --record-set-name $region.Name \
          --nsdname $ns \
          --subscription $parentSub

        Write-Host "  ✓ Added: $ns" -ForegroundColor Green
    }

    Write-Host "✅ Delegation complete for $($region.Name)" -ForegroundColor Green
}

# Step 4: Verify all delegations
Write-Host "`n=== Verifying Delegations ===" -ForegroundColor Cyan
foreach ($region in $regions) {
    Write-Host "`nChecking: $($region.Name).$parentZone" -ForegroundColor Yellow
    nslookup -type=NS "$($region.Name).$parentZone"
}
```

### Common Mistake: Forgetting NS Delegation

```powershell
# ❌ WRONG: Creating child zone without updating parent

# Create child zone
az network dns zone create --name eus.contoso.azure.com ...

# Create A record in child zone
az network dns record-set a add-record \
  --zone-name eus.contoso.azure.com \
  --name order \
  --ipv4-address 13.77.50.25 ...

# ❌ FORGOT THIS STEP: Add NS records to parent!
# Result: order.eus.contoso.azure.com returns NXDOMAIN
```

### Key Takeaway

```
CREATE CHILD ZONE
    ↓
AZURE ASSIGNS NAME SERVERS (4 of them)
    ↓
YOU MUST ADD THESE TO PARENT ZONE AS NS RECORDS ← THIS IS MANDATORY!
    ↓
DNS DELEGATION WORKS
```

**Without this step, your child zone is invisible to the DNS system!**

---

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

---

*Happy DNS-ing! 🚀*
