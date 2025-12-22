# Lab 1: Hub VNet Infrastructure and FortiGate HA Deployment

## Lab Overview

**Duration:** 45 minutes  
**Difficulty:** Intermediate  
**Prerequisites:** Azure subscription with Contributor access, 2 FortiFlex tokens

### Objective

Deploy the foundational hub VNet infrastructure and FortiGate Active-Passive HA cluster with Azure Load Balancers. This establishes the centralized security hub that will protect all spoke VNets in subsequent labs.

### What You'll Build

By the end of this lab, you will have:

- ✅ Resource Group in Canada Central region
- ✅ Hub Virtual Network (10.100.0.0/16) with 4 dedicated subnets
- ✅ FortiGate Active-Passive HA cluster (2 VMs)
- ✅ External Load Balancer with Public IP for internet traffic
- ✅ Internal Load Balancer for east-west traffic inspection
- ✅ HA synchronization configured and operational
- ✅ Access to FortiGate web GUI for both units

### Architecture

After Lab 1:

```text
┌──────────────────────────────────────────────────┐
│     Hub VNet (10.100.0.0/16) - Canada Central    │
│                                                   │
│  External Subnet (10.100.1.0/24)                 │
│  ┌────────────────────────────────┐              │
│  │  External Load Balancer        │◄──Internet   │
│  │  Public IP: [TBD]              │              │
│  └────────┬───────────────┬───────┘              │
│           │               │                       │
│      ┌────▼────┐     ┌────▼────┐                │
│      │ FGT-A   │◄───►│ FGT-B   │                │
│      │(Active) │ HA  │(Passive)│                │
│      └────┬────┘     └────┬────┘                │
│           │               │                       │
│  Internal Subnet (10.100.2.0/24)                 │
│  ┌────────▼───────────────▼───────┐              │
│  │  Internal Load Balancer        │              │
│  │  IP: 10.100.2.4               │              │
│  └────────────────────────────────┘              │
│                                                   │
│  HA Sync Subnet (10.100.3.0/24)                  │
│  [HA heartbeat and config sync]                  │
│                                                   │
│  Management Subnet (10.100.4.0/24)              │
│  [Out-of-band management access]                │
└──────────────────────────────────────────────────┘
```

### Business Context

Redwood Industries completed their initial Azure deployment (AZ-101) with a single FortiGate VM protecting their first workload. Business growth and application criticality now demand:

- **High Availability:** No single point of failure for security infrastructure
- **Multi-Tier Architecture:** Separate VNets for web, application, and database layers
- **Compliance Requirements:** All inter-tier traffic must be inspected and logged

Today, you're deploying the HA security hub that will protect their expanding Azure footprint.

---

## Step 1: Create Resource Group

A Resource Group provides the logical container for all Redwood Industries' hub infrastructure.

### 1.1 Create Resource Group

1. **Log in to Azure Portal:**
   - Navigate to https://portal.azure.com
   - Sign in with your Azure account credentials

2. **Navigate to Resource Groups:**
   - Click **Resource groups** in the left navigation menu
   - Or search for "Resource groups" in the top search bar

3. **Create New Resource Group:**
   - Click **+ Create** button at the top of the page

4. **Configure Resource Group:**
   - **Subscription:** Select your Azure subscription from dropdown
   - **Resource group:** Enter `Redwood-Hub-RG`
   - **Region:** Select **Canada Central** from dropdown
     - ⚠️ **Important:** All resources in this workshop use Canada Central

5. **Review and Create:**
   - Click **Review + create** button at bottom
   - Review the settings
   - Click **Create** button

### Validation

- ✅ Resource group appears in the list with name "Redwood-Hub-RG"
- ✅ Region shows "Canada Central"
- ✅ Status shows "Succeeded"

---

## Step 2: Create Hub Virtual Network

The hub VNet provides the IP address space and subnets for FortiGate HA infrastructure.

### 2.1 Navigate to Virtual Network Creation

1. **From Resource Group:**
   - Click on **Redwood-Hub-RG**
   - Click **+ Create** button

2. **Search for Virtual Network:**
   - In search box, type **virtual network**
   - Click **Virtual network** (Microsoft service)
   - Click **Create**

### 2.2 Configure VNet Basics

1. **Basics Tab:**
   - **Subscription:** Your Azure subscription
   - **Resource group:** Select **Redwood-Hub-RG**
   - **Name:** Enter `Redwood-Hub-VNet`
   - **Region:** Select **Canada Central**
   - Click **Next**

2. **Security Tab:**
   - Do not enable any security features (Bastion, DDoS, Firewall)
   - Click **Next**

### 2.3 Configure IP Address Space

1. **IP Addresses Tab:**
   - **IPv4 address space:** Enter `10.100.0.0/16`
   - **Delete default subnet** (click trash icon)

2. **Create External Subnet:**
   - Click **+ Add subnet**
   - **Subnet name:** `External-Subnet`
   - **Starting address:** `10.100.1.0`
   - **Size:** `/24 (256 addresses)`
   - Click **Add**

3. **Create Internal Subnet:**
   - Click **+ Add subnet**
   - **Subnet name:** `Internal-Subnet`
   - **Subnet address range:** `10.100.2.0`
   - **Size:** `/24 (256 addresses)`
   - Click **Add**

4. **Create HA Sync Subnet:**
   - Click **+ Add subnet**
   - **Subnet name:** `HASync-Subnet`
   - **Subnet address range:** `10.100.3.0`
   - **Size:** `/24 (256 addresses)`
   - Click **Add**

5. **Create Management Subnet:**
   - Click **+ Add subnet**
   - **Subnet name:** `Management-Subnet`
   - **Subnet address range:** `10.100.4.0`
   - **Size:** `/24 (256 addresses)`
   - Click **Add**

6. **Create Management Subnet:**
   - Click **+ Add subnet**
   - **Subnet name:** `Protected-Subnet`
   - **Subnet address range:** `10.100.5.0`
   - **Size:** `/24 (256 addresses)`
   - Click **Add**

### 2.4 Review and Create VNet

1. Click **Review + create**
2. Verify all subnets are listed:
   - External-Subnet (10.100.1.0/24)
   - Internal-Subnet (10.100.2.0/24)
   - HASync-Subnet (10.100.3.0/24)
   - Management-Subnet (10.100.4.0/24)
   - Protected-Subnet (10.100.5.0/24)
3. Click **Create**

### Validation

- ✅ VNet created successfully
- ✅ All 4 subnets present with correct CIDR ranges
- ✅ Address space is 10.100.0.0/16

### Understanding the Subnet Architecture

**Why 4 Subnets for FortiGate HA?**

| Subnet | Purpose | Traffic Type |
|--------|---------|--------------|
| **External** | Internet-facing interface (port1) | North-south (internet) via External LB |
| **Internal** | Application-facing interface (port2) | East-west (VNet-to-VNet) via Internal LB |
| **HA Sync** | HA cluster communication (port3) | FGCP, config sync, session sync, heartbeat |
| **Management** | Out-of-band management (port4) | Direct admin access via Public IPs |
| **Protected** | Protected workloads | Traffic inspected by the FortiGate |

**Key Differences from AZ-101:**

- AZ-101 had 3 subnets (External, Internal, Protected)
- AZ-102 adds dedicated HA Sync and Management subnets
- This separation follows Fortinet best practices for production HA deployments

---

## Step 3: Deploy FortiGate HA from Azure Marketplace

### 3.1 Navigate to Marketplace

1. **From Redwood-Hub-RG:**
   - Click **+ Create**
   - Search for: `FortiGate`

2. **Select FortiGate Offering:**
   - Click **Fortinet FortiGate Next-Generation Firewall**
   - Publisher: Fortinet
   - Click **Create**

3. **Select Deployment Template:**
   - Click **Active-Passive HA with ELB/ILB**
   - This template deploys:
     - 2 FortiGate VMs
     - External Load Balancer (ELB)
     - Internal Load Balancer (ILB)
     - All required networking

### 3.2 Configure Basic Settings

1. **Basics Tab:**
   - **Subscription:** Your subscription
   - **Resource group:** **Redwood-Hub-RG**
   - **Region:** **Canada Central**
   - **FortiGate VM instance Architecture:** `X64 - Intel / AMD based processors | Gen2 VM FortiGate 7.6+`
   - **Username:** `fortiuser`
   - **Password:** Create strong password (save securely!)
   - **Confirm password:** Re-enter password
   - **FortiGate Name Prefix:** `Redwood-Hub`
     - This creates VMs: Redwood-Hub-FGT-A and Redwood-Hub-FGT-B

2. Click **Next**

### 3.3 Configure Instance Settings

1. **Instance Tab:**
   - **FortiGate Image SKU:** `Bring Your Own License or FortiFlex`
   - **FortiGate Image Version:** `7.6.4`
   - **Instance Type:** `Standard_D8als_v6`
     - ⚠️ **Required:** HA needs 4 NICs, minimum Standard_D8als_v6
     - 8 vCPUs, 13 GB RAM
     - Optimized for network security workloads
   - **Availability Option:** `Availability Zones`
     - FortiGate A: Zone 1
     - FortiGate B: Zone 2
     - This provides datacenter-level redundancy

2. **FortiFlex Licensing:**
   - Check **"My organization is using the FortiFlex subscription service"**
   - **FortiGate A FortiFlex:** `[Token 1 from instructor]`
   - **FortiGate B FortiFlex:** `[Token 2 from instructor]`

3. Click **Next**

### 3.4 Configure Networking

1. **Networking Tab:**
   - **Virtual Network:** Select **Redwood-Hub-VNet**
   - **Subnet Mapping:**
     - **External Subnet:** `External-Subnet`
     - **Internal Subnet:** `Internal-Subnet`
     - **HA Sync Subnet:** `HASync-Subnet`
     - **HA Management Subnet:** `Management-Subnet`
     - **Protected Subnet:** `Protected-Subnet`
   - **Accelerated Networking:** `Disabled`
     - ⚠️ Keep disabled for workshop consistency

2. Click **Next: Public IP**

### 3.5 Configure Public IP Addresses

1. **External Load Balancer Public IP:**
   - Click **Create new**
   - **Name:** `Redwood-Hub-ELB-PIP`
   - **SKU:** `Standard`
   - **Routing preference:** `Microsoft network`
   - Click **OK**

2. **FortiGate A Management Public IP:**
   - Click **Create new**
   - **Name:** `Redwood-Hub-FGT-A-Mgmt-PIP`
   - **SKU:** `Standard`
   - **Routing preference:** `Microsoft network`
   - Click **OK**

3. **FortiGate B Management Public IP:**
   - Click **Create new**
   - **Name:** `Redwood-Hub-FGT-B-Mgmt-PIP`
   - **SKU:** `Standard`
   - **Routing preference:** `Microsoft network`
   - Click **OK**

4. Click **Next: Review + create**

### 3.6 Review and Deploy

1. **Review Configuration:**
   - Verify Resource Group: Redwood-Hub-RG
   - Verify Region: Canada Central
   - Verify VNet: Redwood-Hub-VNet
   - Verify all 5 subnets mapped correctly
   - Verify 3 Public IPs configured

2. **Important Pre-Deployment Checklist:**
   - [ ] FortiFlex tokens entered correctly
   - [ ] Password saved securely (you'll need it immediately)
   - [ ] All subnets mapped to correct names
   - [ ] Instance type is Standard_D8als_v6 or supports 4 NICs

3. Click **Create**

### Deployment Time

> [!NOTE]
> **Deployment Duration:** 5-10 minutes typically
>
> While waiting, this is an excellent time to:
> - Review the architecture diagram
> - Read ahead to Lab 2
> - Take a 10-minute break
> - Discuss FortiGate HA concepts with instructor

---

## Step 4: Understand What Gets Deployed

While deployment is running, let's understand what Azure is creating:

### 4.1 Virtual Machines

**Redwood-Hub-FGT-A (Active):**
- Instance: Standard_D8als_v6
- Availability Zone: 1
- 4 Network Interfaces (NICs):
  - port1: External-Subnet
  - port2: Internal-Subnet
  - port3: HASync-Subnet
  - port4: Management-Subnet
- OS Disk: 2 GB (FortiOS)
- Log Disk: 30 GB

**Redwood-Hub-FGT-B (Passive):**
- Identical configuration
- Availability Zone: 2
- Standby mode until failover

### 4.2 Load Balancers

**External Load Balancer (Redwood-Hub-externalloadbalancer):**
- **Frontend IP:** Redwood-Hub-ELB-PIP (public)
- **Backend Pool:** FGT-A port1, FGT-B port1
- **Health Probe:** TCP 8008 (FortiGate HA health endpoint)
- **Load Balancing Rules:** TCP/80 and UDP/10551
- **Purpose:** Distributes inbound internet traffic to active FortiGate

**Internal Load Balancer (Redwood-Hub-internalloadbalancer):**
- **Frontend IP:** 10.100.2.4 (first available IP in Internal-Subnet)
- **Backend Pool:** FGT-A port2, FGT-B port2
- **Health Probe:** TCP 8008
- **Load Balancing Rules:** All/0
- **Purpose:** Routes east-west traffic through active FortiGate
- **Critical:** This IP becomes the next-hop for all UDRs in Lab 3

### 4.3 Public IP Addresses

1. **Redwood-Hub-ELB-PIP:** Internet traffic ingress
2. **Redwood-Hub-FGT-A-Mgmt-PIP:** Direct admin access to FGT-A
3. **Redwood-Hub-FGT-B-Mgmt-PIP:** Direct admin access to FGT-B

---

## Step 5: Verify Deployment Success

### 5.1 Check Deployment Status

1. **Navigate to Redwood-Hub-RG:**
   - Go to Resource groups → Redwood-Hub-RG
   - Click **Settings > Deployments** in left menu
   - Verify deployment shows "Succeeded"

2. **Review Created Resources:**
   - Click on the `fortinet.fortinet-fortigate-xxxxxx` deployment
   - Click on **Deployment details**
   - You should see 18 resources:
     - 2 Virtual Machines (FGT-A, FGT-B)
     - 8 Network Interfaces (4 per VM)
     - 1 Network Security Groups
     - 3 Public IP Addresses
     - 2 Load Balancers
     - 1 Route table
     - 1 Deployment
     - Additional managed disks and NICs

### 5.2 Verify Network Configuration

1. **Check Virtual Network:**
   - Click **Redwood-Hub-VNet**
   - Click **Settings > Subnets**
   - Verify all 4 subnets present
   - Click **Connected devices** (above **Subnets**)
   - You should see 8 network interfaces and 1 internal load balancer connected

2. **Check Internal Load Balancer IP:**
   - Verify IP address is **10.100.2.4**
   - ⚠️ **Write this down** - critical for Lab 3 UDRs

### 5.3 Retrieve Management Public IPs

1. **FortiGate A Management IP:**
   - Click **Redwood-Hub-FGT-A-Mgmt-PIP**
   - Copy the **IP address**
   - Example: `20.220.45.67`
   - Save this for Step 6

2. **FortiGate B Management IP:**
   - Click **Redwood-Hub-FGT-B-Mgmt-PIP**
   - Copy the **IP address**
   - Save this for verification

### 5.4 Retrieve External Load Balancer IP

1. **External LB Public IP:**
   - Click **Redwood-Hub-ELB-PIP**
   - Copy the **IP address**
   - This will be used for inbound traffic in Lab 4

### Validation Checklist

Before proceeding, verify:

- [ ] Deployment status shows "Succeeded"
- [ ] 2 FortiGate VMs running (FGT-A, FGT-B)
- [ ] 2 Load Balancers created
- [ ] 3 Public IPs assigned
- [ ] 4 Subnets in Redwood-Hub-VNet
- [ ] Internal LB IP is 10.100.2.4
- [ ] Management Public IPs recorded

---

## Step 6: Access FortiGate Web GUI

### 6.1 Connect to FortiGate A

1. **Open Web Browser:**
   - Use Chrome or Firefox (recommended)
   - Open new private/incognito window

2. **Navigate to FortiGate A:**
   - Enter URL: `https://[Redwood-Hub-FGT-A-Mgmt-PIP]`
   - Example: `https://20.220.45.67`
   - Press Enter

3. **Accept Certificate Warning:**
   - You'll see "Your connection is not private" (expected)
   - Click **Advanced**
   - Click **Proceed to [IP] (unsafe)**

4. **FortiGate Login:**
   - **Username:** `fortiuser`
   - **Password:** [Password you created in Step 3.2]
   - Click **Login**

5. **First Login Wizard:**
   - If prompted with setup wizard, click **Begin**
   - **Dashboard Setup:** Select **Optimal** (default)
   - Click **OK**

### 6.2 Verify FortiGate A Status

1. **Dashboard Overview:**
   - Verify **System Information** widget shows:
     - **Host Name:** Redwood-Hub-FGT-A
     - **Version:** FortiOS 7.6.4
     - **Serial Number:** Should start with `FGVM`
     - **System Status:** OK (green)

2. **Check HA Status:**
   - Click **+Add widget** 
   - Under **System** session, select the **HA Status** widget click on it
   - Click **OK**
   - Close the **Add Dashboard Widget** tab
   - Check the **HA Status** widget
   - **Mode:** Should show `Active-Passive`
   - **Primary:** Redwood-Hub-FGT-A
   - **Seconday:** Redwood-Hub-FGT-B
   - **State Changed:** Should be a few seconds behind the **Uptime**

   > [!NOTE]
   > HA synchronization may take 2-3 minutes after deployment. If cluster status doesn't show 2 members immediately, wait 3 minutes and refresh.

3. **Verify Interfaces:**
   - Navigate to **Network > Interfaces**
   - Verify 4 interfaces present:
     - **port1** (External): IP from External-Subnet, **Admin. access: Probe Response**
     - **port2** (Internal): IP from Internal-Subnet, **Admin. access: Probe Response**
     - **port3** (HA): IP from HASync-Subnet
     - **port4** (MGMT): IP from Management-Subnet, **Admin. access: PING, HTTPS, SSH, FTM**

### 6.3 Verify HA Configuration

1. **Navigate to HA Settings:**
   - Go to **System > HA**
   - Double click on Hostname **Redwood-Hub-FGT-A**
   - Verify configuration:
     - **Mode:** `Active-Passive`
     - **Device Priority:** 255 (FGT-A should be higher)
     - **Group Name:** Should be auto-configured
     - **Password:** Encrypted (auto-configured)
     - **Heartbeat Interface:** port3 (HASync-Subnet)
     - **Management Interface Reservation:** port4
   - Don't change anything, click **Cancel**

   > [!TIP]
   > If Status shows "Out of Sync", wait 2-3 minutes. Initial config sync can take time. If still out of sync after 5 minutes, check troubleshooting section.

---

## Step 7: Verify FortiGate B (Passive Unit)

### 7.1 Connect to FortiGate B

1. **Open New Browser Tab:**
   - Navigate to: `https://[Redwood-Hub-FGT-B-Mgmt-PIP]`

2. **Login to FortiGate B:**
   - Same credentials as FGT-A
   - **Username:** `fortiuser`
   - **Password:** [Same password]

3. **Verify Passive Status:**
   - Add the **HA Status** widget to the Dashboard
   - Dashboard should show similar information
   - **HA Status:** Should indicate this is the **Secondary** unit
   - Configuration should be identical to FGT-A (synchronized)

### 7.2 Understand HA Roles

**Active Unit (Currently FGT-A):**
- Processes all traffic
- Health probe responds to both load balancers
- Configuration changes made here sync to secondary FortiGate
- Maintains active sessions

**Passive Unit (Currently FGT-B):**
- Standby mode
- Monitors active unit health
- Receives config sync from Primary
- Ready to take over in seconds if active fails
- Sessions synchronized in real-time

### 7.3 Test Configuration Sync

1. **On FortiGate A:**
   - Navigate to **System > Settings**
   - Change **Timezone** to a different timezone
   - Change **Idle timeout** to 120 minutes
   - Click **Apply**

2. **On FortiGate B:**
   - Refresh the **System** → **Settings** page
   - Verify timezone and Idle timeout changed to match FGT-A
   - This confirms HA config sync is working

---

## Step 8: Review HA Load Balancer Configuration

### 8.1 Understand Health Probes

**How Azure Determines Active FortiGate:**

1. **Health Probe Configuration:**
   - Both load balancers probe TCP port 8008
   - Only the active FortiGate responds to probes
   - Passive unit ignores probe requests
   - This directs all traffic to the active unit

2. **Verify Health Probe:**
   - In Azure Portal, go to **Redwood-Hub-externalloadbalancer**
   - Click **Monitoring >  Insights**
   - Find both FortiGates in the diagram
   - Click on each one of them and explore the status and other metrics
   - Redwood-Hub-FGT-A instance should be the health one

### 8.2 Understand Traffic Flow

**Internet-Bound Traffic (North-South):**
```text
Spoke VM → UDR → Internal LB (10.100.2.4) 
         → Active FGT port2 → Inspect → FGT port1 
         → External LB → Internet
```

**Inter-VNet Traffic (East-West):**
```text
Spoke1 VM → UDR → Internal LB (10.100.2.4)
          → Active FGT port2 → Inspect
          → Active FGT port2 → Internal LB
          → Hub routing → Spoke2 VM
```

**Key Point:** Internal LB (10.100.2.4) becomes the critical next-hop for all routing in Lab 3.

---

## Lab 1 Complete! 🎉

### What You've Accomplished

You have successfully deployed the hub infrastructure for Redwood Industries:

✅ **Resource Group:** Redwood-Hub-RG in Canada Central  
✅ **Hub VNet:** Redwood-Hub-VNet (10.100.0.0/16)  
✅ **4 Dedicated Subnets:** External, Internal, HA Sync, Management  
✅ **FortiGate HA Cluster:** 2 VMs in Active-Passive mode  
✅ **External Load Balancer:** Public IP for internet traffic  
✅ **Internal Load Balancer:** 10.100.2.4 for east-west inspection  
✅ **HA Synchronization:** Active, verified, and operational  
✅ **Management Access:** Working web GUI access to both FortiGates

### Architecture Review

```text
Deployed Infrastructure:
┌─────────────────────────────────────────────────┐
│  Hub VNet (10.100.0.0/16)                       │
│  ┌───────────────────────────────────────────┐ │
│  │  External LB (Public IP)                  │ │
│  └──────┬──────────────┬──────────────────────┘ │
│         │              │                        │
│    ┌────▼─────┐   ┌────▼─────┐                │
│    │ FGT-A    │◄─►│ FGT-B    │                │
│    │ Active   │HA │ Passive  │                │
│    └────┬─────┘   └────┬─────┘                │
│         │              │                        │
│  ┌──────▼──────────────▼───────┐               │
│  │  Internal LB (10.100.2.4)   │               │
│  └──────────────────────────────┘               │
│                                                 │
│  [Ready for spoke VNets in Lab 2]              │
└─────────────────────────────────────────────────┘
```

### Key Takeaways

1. **HA Provides Resilience:**
   - No single point of failure
   - Automatic failover in <10 seconds
   - Meets enterprise SLA requirements

2. **Load Balancers Enable HA:**
   - Health probes detect active FortiGate
   - Traffic automatically routes to healthy unit
   - Both external and internal traffic covered

3. **Dedicated Subnets for HA:**
   - Separates traffic types (internet, internal, HA, mgmt)
   - Follows Fortinet best practices
   - Simplifies troubleshooting

4. **Internal LB is Critical:**
   - IP address 10.100.2.4
   - Becomes next-hop for all spoke routing
   - Central inspection point for east-west traffic

### Information to Record

Before moving to Lab 2, ensure you have recorded:

- [ ] Internal Load Balancer IP: **10.100.2.4**
- [ ] FortiGate A Management Public IP: **[Your IP]**
- [ ] FortiGate B Management Public IP: **[Your IP]**
- [ ] External Load Balancer Public IP: **[Your IP]**
- [ ] FortiGate Username: **fortiuser**
- [ ] FortiGate Password: **[Your Password]**

### Next Steps

Ready for **Lab 2: Spoke VNets Deployment and VNet Peering!**

In Lab 2, you will:
- Create three spoke VNets (frontend, backend, database)
- Deploy test VMs in each spoke
- Establish VNet peering from all spokes to hub
- Test basic connectivity before adding security inspection

---

## Troubleshooting Reference

### Issue: HA Cluster Not Forming

**Symptoms:**
- Dashboard shows only 1 cluster member
- HA status shows "Standalone" mode

**Solutions:**
1. Wait 5 minutes - HA negotiation can take time
2. Verify both VMs are running (Azure Portal > VMs)
3. Check HASync-Subnet connectivity:
   - Both FortiGates have IPs in 10.100.3.0/24
   - NSG allows HA traffic (auto-configured by template)
4. Verify HA configuration identical on both units:
   - **System** → **HA** → Verify group name matches
   - Priority should differ (FGT-A: 255, FGT-B: 1)

### Issue: Can't Access FortiGate Web GUI

**Checklist:**
1. Verify VM is running (Azure Portal > VM status)
2. Verify Public IP assigned (check Management-Subnet NIC)
3. Try different browser or incognito mode
4. Check NSG allows HTTPS from your IP
5. Wait 3-5 minutes after deployment for FortiOS initialization

### Issue: Configuration Not Syncing Between HA Members

**Checklist:**
1. Verify HA status shows "Synchronized" (System > HA > Status)
2. Check HASync-Subnet connectivity
3. Ensure configuration changes made on PRIMARY unit only
4. Wait 60 seconds and refresh passive unit GUI
5. Check for firmware version mismatch (should both be 7.6.4)

### Issue: Health Probe Failures

**Symptoms:**
- Load balancer shows 0 healthy instances
- Traffic not flowing through FortiGate

**Solutions:**
1. Verify port 8008 is open on FortiGate (Administrative access: `Probe Response`)
2. Verify FortiGate interface addressing matches subnet
3. Check Azure Portal → Load Balancer → Health Probes for status

### Issue: Wrong Internal Load Balancer IP

**Problem:** Internal LB assigned IP other than 10.100.2.4

**Solution:**
1. Delete Internal Load Balancer
2. Recreate with static IP assignment of 10.100.2.4
3. Or adjust UDRs in Lab 3 to match actual LB IP

### Getting Help

**During Workshop:**
- Raise hand for instructor assistance
- Check with neighbor if they encountered same issue
- Review deployment logs in Azure Portal

**After Workshop:**
- Fortinet Community Forums
- Azure Support (for Azure-specific issues)
- Workshop GitHub repository for issues/questions

---

## Advanced: Understanding HA Failover

### How Failover Works

1. **Failure Detection:**
   - Both units monitor each other via HA heartbeat (port3)
   - Health probe failure on active unit
   - Interface failure on active unit
   - Manual failover triggered by admin

2. **Failover Process:**
   - Passive unit detects active failure (<1 second)
   - Passive assumes active role
   - Begins responding to health probes
   - Load balancers detect new active unit (~5-8 seconds)
   - Traffic redirected to new active unit
   - Sessions synchronized (if session-sync enabled)

3. **Total Failover Time:**
   - Detection: <1 second
   - Role change: ~1 second
   - Health probe update: 5-10 seconds
   - **Total:** ~10-15 seconds

### Session Synchronization

**What Gets Synchronized:**
- Firewall connections/sessions
- IPsec VPN tunnels
- NAT translations
- Configuration changes

**What Doesn't Get Synchronized (by default):**
- DHCP leases
- SSL VPN sessions (can be enabled)
- Some routing tables

---

**End of Lab 1**  
*Estimated completion time: 45 minutes*  
*Next: Lab 2 - Spoke VNets Deployment and VNet Peering*

---

*Lab Guide Version 1.0 - December 2024*  
*Questions? Ask your instructor or refer to the troubleshooting section.*