# Lab 2: Spoke VNets Deployment and VNet Peering

## Lab Overview

**Duration:** 30 minutes  
**Difficulty:** Intermediate  
**Prerequisites:** Completed Lab 1

### Objective

Deploy three spoke VNets representing Redwood Industries' multi-tier application architecture (frontend, backend, database), deploy test VMs in each spoke, and establish VNet peering to the hub. This creates the foundation for the hub-spoke topology.

### What You'll Build

By the end of this lab, you will have:

- ✅ Three resource groups for spoke organization
- ✅ Frontend VNet (10.101.0.0/16) with two Ubuntu VMs
- ✅ Backend VNet (10.102.0.0/16) with Ubuntu VM
- ✅ Database VNet (10.103.0.0/16) with Ubuntu VM
- ✅ VNet peering from all spokes to hub
- ✅ Basic connectivity between hub and all spokes

### Architecture

After Lab 2:

```text
                 Hub VNet (10.100.0.0/16)
              ┌────────────────────────────┐
              │  FortiGate HA Cluster      │
              │  ┌──────────────────────┐  │
              │  │ FGT-A ◄────► FGT-B   │  │
              │  └──────────────────────┘  │
              │                            │
              │  Internal LB: 10.100.2.4   │
              └────────────────────────────┘
                   ▲         ▲         ▲
              (Peering) (Peering) (Peering)
                   │         │         │
        ┌──────────┘         │         └──────────┐
        │                    │                    │
┌───────▼────────┐  ┌────────▼───────┐  ┌────────▼───────┐
│ Frontend VNet  │  │ Backend VNet   │  │ Database VNet  │
│ 10.101.0.0/16  │  │ 10.102.0.0/16  │  │ 10.103.0.0/16  │
│                │  │                │  │                │
│ ┌────────────┐ │  │ ┌────────────┐ │  │ ┌────────────┐ │
│ │ 2 x Web-VM │ │  │ │ App-VM     │ │  │ │ DB-VM      │ │
│ │ .1.4 , 1.5 │ │  │ │ .1.4       │ │  │ │ .1.4       │ │
│ └────────────┘ │  │ └────────────┘ │  │ └────────────┘ │
└────────────────┘  └────────────────┘  └────────────────┘

Note: No internet access or inter-spoke connectivity yet
UDRs and firewall policies configured in Labs 3-4
```

### Business Context

Redwood Industries is migrating their three-tier application to Azure:

- **Frontend VNet:** Web servers serving customer requests
- **Backend VNet:** Application logic and business services  
- **Database VNet:** SQL databases and data storage

Security requirements demand:
- Complete network segmentation between tiers
- All inter-tier traffic must flow through FortiGate for inspection
- No direct communication between tiers (zero-trust model)
- No direct communication between VMs in the same tiers (zero-trust model)
- Centralized logging and visibility

---

## Part A: Create Spoke Resource Groups

### Step 1: Create Frontend Resource Group

1. **Navigate to Resource Groups:**
   - Azure Portal → Resource groups
   - Click **+ Create**

2. **Configure Frontend RG:**
   - **Subscription:** Your subscription
   - **Resource group:** `Redwood-Frontend-RG`
   - **Region:** `Canada Central`
   - Click **Review + create**
   - Click **Create**

### Step 2: Create Backend Resource Group

1. **Create Second RG:**
   - Click **+ Create** again
   - **Resource group:** `Redwood-Backend-RG`
   - **Region:** `Canada Central`
   - Click **Review + create** → **Create**

### Step 3: Create Database Resource Group

1. **Create Third RG:**
   - Click **+ Create**
   - **Resource group:** `Redwood-Database-RG`
   - **Region:** `Canada Central`
   - Click **Review + create** → **Create**

### Validation

- ✅ Three resource groups created
- ✅ All in Canada Central region
- ✅ Names follow pattern: Redwood-[Tier]-RG

---

## Part B: Create Frontend VNet and VMs

### Step 4: Create Frontend VNet

#### 4.1 Navigate to VNet Creation

1. **From Redwood-Frontend-RG:**
   - Click the resource group
   - Click **+ Create**
   - Search: `virtual network`
   - Click  → **Create > Virtual network**

#### 4.2 Configure Frontend VNet Basics

1. **Basics Tab:**
   - **Subscription:** Your subscription
   - **Resource group:** `Redwood-Frontend-RG`
   - **Name:** `Redwood-Frontend-VNet`
   - **Region:** `Canada Central`
   - Click **Next: Security**

2. **Security Tab:**
   - Keep all disabled (no Bastion, DDoS, Firewall)
   - Click **Next**

#### 4.3 Configure Frontend IP Addresses

1. **IP Addresses Tab:**
   - **IPv4 address space:** `10.101.0.0/16`
   - Delete default subnet if present

2. **Create Protected Subnet:**
   - Click **+ Add subnet**
   - **Subnet Name:** `Protected-Subnet`
   - **Subnet Starting address:** `10.101.1.0`
   - **Subnet Size:** `/24 (256 addresses)`
   - Click **Add**

3. Click **Review + create** → **Create**

### Step 5: Create Frontend VMs

#### 5.1 Start VM-1 Creation

1. **From Redwood-Frontend-RG:**
   - Click **+ Create**
   - Search: `virtual machine`
   - Click **Virtual machine > Create**

#### 5.2 Configure VM Basics

1. **Basics Tab:**
   - **Virtual machine name:** `Redwood-Frontend-VM-1`
   - **Region:** `Canada Central`
   - **Availability options:** `No infrastructure redundancy required`
   - **Security type:** `Standard`
   - **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2`
   - **VM architecture:** `x64`
   - **Size:** `Standard_B1s` (1 vCPU, 1 GB RAM)
     - Click **See all sizes** if not visible
     - Filter for B-series, select B1s

#### 5.3 Configure Authentication

1. **Administrator account:**
   - **Authentication type:** `Password`
   - **Username:** `azureuser`
   - **Password:** Create strong password (save it!)
   - **Confirm password:** Re-enter password

2. **Inbound port rules:**
   - **Public inbound ports:** `None`
   - ⚠️ **Critical:** No public IP - access via hub only

3. Click **Next: Disks >**

#### 5.4 Configure Disks

1. **Disks Tab:**
   - **OS disk type:** `Standard SSD (locally-redundant storage)`
   - Keep other defaults

2. Click **Next: Networking >**

#### 5.5 Configure Networking

1. **Networking Tab:**
   - **Virtual network:** `Redwood-Frontend-VNet`
   - **Subnet:** `Protected-Subnet (10.101.1.0/24)`
   - **Public IP:** `None` ⚠️
   - **NIC network security group:** `Basic`
   - **Public inbound ports:** `None`

2. Click **Review + create**

#### 5.6 Review and Create

1. **Verify Configuration:**
   - Resource group: Redwood-Frontend-RG
   - Region: Canada Central
   - No public IP
   - Size: Standard_B1s

2. Click **Create**

#### 5.7 Start VM-2 creation

- Once the firt vm is created, create a second vm using exactly the same parameters.  
- Use the **Virtual machine name**: `Redwood-Frontend-VM-2` for your second vm.

### Validation

- ✅ Frontend VNet created (10.101.0.0/16)
- ✅ Protected-Subnet created (10.101.1.0/24)
- ✅ Frontend VMs deploying (no public IP)

---

## Part C: Create Backend VNet and VM

### Step 6: Create Backend VNet

Follow the same process as Frontend, with these values:

1. **VNet Configuration:**
   - **Resource group:** `Redwood-Backend-RG`
   - **Name:** `Redwood-Backend-VNet`
   - **Region:** `Canada Central`
   - **IPv4 address space:** `10.102.0.0/16`
   - **Subnet name:** `Protected-Subnet`
   - **Subnet Starting address:** `10.102.1.0`
   - **Subnet Size:** `/24 (256 addresses)`

### Step 7: Create Backend VM

1. **VM Configuration:**
   - **Resource group:** `Redwood-Backend-RG`
   - **Virtual machine name:** `Redwood-Backend-VM`
   - **Region:** `Canada Central`
   - **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2`
   - **Size:** `Standard_B1s`
   - **Username:** `azureuser`
   - **Password:** Same as frontend (or different, your choice)
   - **Virtual network:** `Redwood-Backend-VNet`
   - **Subnet:** `Protected-Subnet (10.102.1.0/24)`
   - **Public IP:** `None` ⚠️

### Validation

- ✅ Backend VNet created (10.102.0.0/16)
- ✅ Backend VM deploying (no public IP)

---

## Part D: Create Database VNet and VM

### Step 8: Create Database VNet

1. **VNet Configuration:**
   - **Resource group:** `Redwood-Database-RG`
   - **Name:** `Redwood-Database-VNet`
   - **Region:** `Canada Central`
   - **IPv4 address space:** `10.103.0.0/16`
   - **Subnet name:** `Protected-Subnet`
   - **Subnet range:** `10.103.1.0/24`

### Step 9: Create Database VM

1. **VM Configuration:**
   - **Resource group:** `Redwood-Database-RG`
   - **Virtual machine name:** `Redwood-Database-VM`
   - **Region:** `Canada Central`
   - **Image:** `Ubuntu Server 24.04 LTS - x64 Gen2`
   - **Size:** `Standard_B1s`
   - **Username:** `azureuser`
   - **Password:** Same as others
   - **Virtual network:** `Redwood-Database-VNet`
   - **Subnet:** `Protected-Subnet (10.103.1.0/24)`
   - **Public IP:** `None` ⚠️

### Validation

- ✅ Database VNet created (10.103.0.0/16)
- ✅ Database VM deploying (no public IP)

> [!TIP]
> While VMs are deploying (2-3 minutes each), this is a good time to review the architecture diagram and understand how the hub-spoke topology will work.

---

## Part E: Establish VNet Peering

### Understanding VNet Peering

**What is VNet Peering?**

- Direct connection between two VNets in Azure
- Traffic uses Azure backbone (never traverses internet)
- Low latency, high bandwidth
- No gateway required

**Hub-Spoke Topology:**

- Each spoke peers with hub
- Spokes do NOT peer with each other
- All inter-spoke traffic forced through hub FortiGate
- Centralized security inspection point

### Step 10: Create Hub-to-Frontend Peering

#### 10.1 Navigate to Hub VNet

1. **Go to Hub VNet:**
   - Navigate to **Redwood-Hub-RG**
   - Click **Redwood-Hub-VNet**
   - Click **Settings** → **Peerings** in left menu

#### 10.2 Add Frontend Peering

1. **Create Peering:**
   - Click **+ Add**

2. **Remote virtual network (Frontend VNet):**
   - **Peering link name:** `Frontend-to-Hub`
   - **Subscription:** Your subscription
   - **Virtual network:** `Redwood-Frontend-VNet`
   - Allow `Redwood-Frontend-VNet` to access `Redwood-Hub-VNet`: `Allow (default)`
   - Allow `Redwood-Frontend-VNet` to receive forwarded traffic from `Redwood-Hub-VNet`: `Allow (default)`

3. **Local virtual network (Hub VNet):**
   - **Peering link name:** `Hub-to-Frontend`
   - Allow `Redwood-Hub-VNet` to access `Redwood-Frontend-VNet`: `Allow (default)`

4. Click **Add**

#### 10.3 Verify Peering Status

1. **Check Status:**
   - **Perring sync status** shows `Fully Synchornized`
   - **Peering state** should change from `Updating` to `Connected`
   - This may take 30-60 seconds
   - Refresh page if needed

### Step 11: Create Hub-to-Backend Peering

1. **Still in Redwood-Hub-VNet Peerings:**
   - Click **+ Add**

2. **Configure:**
   - **Remote VNet (Backend) peering link name:** `Backend-to-Hub`
   - **Virtual network:** `Redwood-Backend-VNet`
   - **Local VNet (Hub) peering link name:** `Hub-to-Backend`
   - **Allow traffic options:** Same as Frontend (both directions allowed)

3. Click **Add**

### Step 12: Create Hub-to-Database Peering

1. **Create Third Peering:**
   - Click **+ Add**

2. **Configure:**
   - **Remote VNet (Database) peering link name:** `Database-to-Hub`
   - **Virtual network:** `Redwood-Database-VNet`
   - **Local VNet (Hub) peering link name:** `Hub-to-Database`
   - **Allow traffic options:** Same as previous

3. Click **Add**

### Step 13: Verify All Peerings

1. **Check Peering Summary:**
   - Navigate to **Redwood-Hub-VNet** → **Peerings**
   - You should see 3 peerings:
     - Hub-to-Frontend: Fully Synchronized - Connected
     - Hub-to-Backend: Fully Synchronized - Connected
     - Hub-to-Database: Fully Synchronized - Connected

### Validation

- ✅ Three peerings established from hub
- ✅ All show **Peering state** `Connected`
- ✅ Bidirectional traffic allowed

---

## Part F: Verify VM IP Addresses

### Step 14: Record VM IP Addresses

#### 14.1 Frontend VM IP

1. **Navigate to Frontend VM 1:**
   - Go to **Redwood-Frontend-RG**
   - Click **Redwood-Frontend-VM-1**
   - In **Overview**, note **Private IP address**
   - Should be: `10.101.1.4`
   - Write this down

2. **Navigate to Frontend VM 2:**
   - Go to **Redwood-Frontend-RG**
   - Click **Redwood-Frontend-VM-2**
   - In **Overview**, note **Private IP address**
   - Should be: `10.101.1.5`
   - Write this down


#### 14.2 Backend VM IP

1. **Navigate to Backend VM:**
   - Go to **Redwood-Backend-RG**
   - Click **Redwood-Backend-VM**
   - Note **Private IP address**
   - Should be: `10.102.1.4`
   - Write this down

#### 14.3 Database VM IP

1. **Navigate to Database VM:**
   - Go to **Redwood-Database-RG**
   - Click **Redwood-Database-VM**
   - Note **Private IP address**
   - Should be: `10.103.1.4`
   - Write this down

### Why .4 Not .1?

Azure reserves first 4 IPs in each subnet:
- `.0` = Network address
- `.1` = Default gateway (Azure's virtual router)
- `.2` & `.3` = Azure DNS
- `.255` = Broadcast address

First available IP for VMs is `.4`

---

## Part G: Initial Connectivity Testing

### Step 15: Test Hub-to-Spoke Connectivity

Since we have no UDRs configured yet, traffic uses Azure's default routing (direct VNet peering).

#### 15.1 Access FortiGate CLI

**Option A: Web GUI Console (Easier):**

1. **Login to FortiGate A:**
   - Navigate to `https://[FGT-A-Mgmt-Public-IP]`
   - Login with `fortiuser` credentials

2. **Open CLI Console:**
   - Click user icon (top right)
   - Click **CLI Console**
   - Terminal window opens

**Option B: SSH (Advanced):**

```bash
ssh fortiuser@[FGT-A-Mgmt-Public-IP]
```

#### 15.2 Test Connectivity from FortiGate

**From FortiGate A CLI:**

```bash
# Test Frontend VM 1
execute ping 10.101.1.4

# Test Frontend VM 2
execute ping 10.101.1.5

# Test Backend VM  
execute ping 10.102.1.4

# Test Database VM
execute ping 10.103.1.4
```

**Expected Results:**
- ✅ All pings should succeed
- This proves VNet peering is working
- Traffic uses Azure backbone routing

### Step 16: Verify No Internet Access (Expected)

#### 16.1 Access Frontend VM

Since VMs have no public IPs, we need to access them through Azure Serial Console or by SSH from FortiGate.

**Method 1: Azure Serial Console (Recommended for this test):**

1. **Navigate to Frontend VM:**
   - Go to **Redwood-Frontend-RG** → **Redwood-Frontend-VM**
   - Click **Help** → **Serial console** in left menu
   - Login with `azureuser` credentials

2. **Test Internet:**
   ```bash
   ping -c 3 8.8.8.8
   ```

**Expected Result:**
- ❌ Ping fails (no internet access)
- This is correct - no UDRs or firewall policies configured yet

**Method 2: SSH Through FortiGate:**

From FortiGate CLI:
```bash
# SSH to Frontend VM from FortiGate
execute ssh azureuser@10.101.1.4
# Enter VM password when prompted

# Once connected to VM:
ping -c 3 8.8.8.8
# Should fail - no internet route
```

### Step 17: Test Inter-Spoke Connectivity

#### 17.1 Test Frontend to Backend

**From Frontend VM:**

```bash
ping -c 3 10.102.1.4
```

**Expected Result:**

- ❌ **Fails** (no route to Backend VNet)
- **Why:** VNet peering is non-transitive in Azure
- Frontend has peering to Hub, Backend has peering to Hub
- But Frontend and Backend have no direct peering or route to each other
- **This is correct behavior** - demonstrates need for UDRs in Lab 3

#### 17.2 Test Frontend to Database

**From Frontend VM:**

```bash
ping -c 3 10.103.1.4
```

**Expected Result:**
- ❌ **Fails** (same reason as above)
- Spokes cannot communicate without UDRs forcing traffic through FortiGate
  
---

## Lab 2 Complete! 🎉

### What You've Accomplished

You have successfully deployed the spoke infrastructure for Redwood Industries:

✅ **Three Resource Groups:** Frontend, Backend, Database  
✅ **Three Spoke VNets:**
   - Frontend VNet (10.101.0.0/16)
   - Backend VNet (10.102.0.0/16)
   - Database VNet (10.103.0.0/16)

✅ **Three Test VMs:**
   - Frontend VM 1 (10.101.1.4)
   - Frontend VM 2 (10.101.1.5)
   - Backend VM (10.102.1.4)
   - Database VM (10.103.1.4)

✅ **Hub-Spoke Topology:**
   - Three VNet peerings established
   - Hub can reach all spokes
   - Spokes cannot reach each other (for now)

### Architecture Review

```text
Current State After Lab 2:

        Hub VNet (10.100.0.0/16)
     ┌──────────────────────────┐
     │  FortiGate HA Cluster    │
     │  Internal LB: 10.100.2.4 │
     └──────────────────────────┘
          ▲         ▲         ▲
          │    Peerings       │
          │                   │
    ┌─────┴────┐  ┌────┴────┐  ┌────┴─────┐
    │ Frontend │  │ Backend │  │ Database │
    │ .101.0/16│  │.102.0/16│  │.103.0/16 │
    │          │  │         │  │          │
    │ VM: .1.4 │  │ VM: .1.4│  │ VM: .1.4 │
    └──────────┘  └─────────┘  └──────────┘

Current Connectivity:
✓ Hub ↔ All Spokes (via peering)
✗ Frontend ↔ Backend (no route - peering is non-transitive)
✗ Frontend ↔ Database (no route - peering is non-transitive)
✗ Backend ↔ Database (no route - peering is non-transitive)
✗ Internet access (not configured)


After Lab 3 (UDRs):
Traffic will be forced through FortiGate
Connectivity will break temporarily
Lab 4 will restore it with firewall policies
```

### Key Takeaways

1. **Hub-Spoke Design:**
   - Scalable: Can add 100+ spoke VNets
   - Centralized: All security in hub
   - Non-transitive: Spokes can't reach each other directly

2. **VNet Peering is Non-Transitive:**
   - Each spoke peers only with hub
   - Spokes do NOT have routes to each other
   - This is Azure's default security posture
   - UDRs in Lab 3 will enable spoke-to-spoke via FortiGate

3. **No Public IPs:**
   - Spokes completely isolated from internet
   - Access only through hub FortiGate
   - Follows zero-trust principles


4. **Current Limitations:**
   - No spoke-to-spoke connectivity (peering is non-transitive)
   - No traffic inspection (spokes isolated)
   - No internet access
   - **Labs 3-4 will enable connectivity with security!**

### Information to Record

Ensure you have recorded these IP addresses:

- [ ] Internal Load Balancer: **10.100.2.4** (from Lab 1)
- [ ] Frontend VM 1: **10.101.1.4**
- [ ] Frontend VM 2: **10.101.1.5**
- [ ] Backend VM: **10.102.1.4**
- [ ] Database VM: **10.103.1.4**
- [ ] All VM passwords saved securely

### Next Steps

Ready for **Lab 3: User-Defined Routes Configuration!**

In Lab 3, you will:
- Create route tables for all three spoke VNets
- Configure UDRs pointing to Internal LB (10.100.2.4)
- Force all traffic through FortiGate cluster
- Understand traffic flow patterns
- *Note: Connectivity will break temporarily until Lab 4*

---

## Troubleshooting Reference

### Issue: VNet Peering Shows "Updating" Forever

**Solutions:**
1. Wait 2 minutes and refresh page
2. Check both VNets are in same Azure region
3. Delete peering and recreate
4. Verify no overlapping address spaces

### Issue: Can't Ping Spoke VMs from FortiGate

**Checklist:**
1. Verify VNet peering status is "Connected"
2. Check VM is running (Azure Portal → VM status)
3. Verify correct IP address (should be .1.4 in each spoke)
4. Check NSG on VM NIC allows ICMP (auto-configured, should allow)

### Issue: VM Shows Wrong IP Address

**Expected:** VMs get first available IP (.4) in subnet

**If different:**
1. Verify VM is in correct VNet/subnet
2. Check if multiple NICs attached
3. IP may be .5 or .6 if .4 was already taken

### Issue: Can't Access Serial Console

**Solutions:**
1. Verify Boot diagnostics enabled on VM
2. Try "Restart" on VM to reset console
3. Use SSH from FortiGate as alternative
4. Check browser popup blocker

### Issue: Inter-Spoke Connectivity Fails

**Checklist:**
1. Verify VNet peering allows forwarded traffic (both directions)
2. Check NSGs don't block traffic
3. Verify VMs are running
4. Test from FortiGate first to isolate issue

---

## Understanding What's Next

### Current Traffic Flow

**Hub to Spoke:**

```text
FortiGate → Azure peering → Spoke VM
(Direct route, no inspection)
```

**Spoke to Spoke:**

**Spoke to Spoke:**

```text
Frontend VM → ❌ NO ROUTE ❌
(VNet peering is non-transitive - spokes cannot reach each other)
```

### After Lab 3 (UDRs Added)

**Hub to Spoke:**

```text
FortiGate → Internal LB → FortiGate → Hub peering → Spoke VM
(Traffic hairpins through FortiGate for inspection)
```

**Spoke to Spoke:**

```text
Frontend VM → UDR → Internal LB → FortiGate inspection → 
Internal LB → Hub peering → Backend VM
(Full inspection path)
```

### Why Add UDRs?

**Without UDRs:**

- Traffic uses shortest Azure path
- Bypasses FortiGate completely
- No security inspection
- No logging or visibility

**With UDRs:**

- Force all traffic to Internal LB (10.100.2.4)
- Internal LB sends to active FortiGate
- Complete traffic inspection
- Full logging and control

---

**End of Lab 2**  
*Estimated completion time: 30 minutes*  
*Next: Lab 3 - User-Defined Routes Configuration*

---

*Lab Guide Version 1.0 - December 2024*  
*Questions? Ask your instructor or refer to the troubleshooting section.*
