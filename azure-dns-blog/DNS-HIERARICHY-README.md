# Understanding DNS Hierarchy: From Root to Application Endpoint

**A Practical Guide to DNS Resolution and Azure DNS Management**

---

## Table of Contents

1. [Introduction](#introduction)
2. [DNS Hierarchy Basics](#dns-hierarchy-basics)
3. [How DNS Resolution Works](#how-dns-resolution-works)

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

![DNS NS Hierarichy Diagram](https://github.com/PiyushMittl/Others/blob/main/azure-dns-blog/images/2.dns-parent-chield-ns-hierarichy.png)

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
