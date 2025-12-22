# Lab 4: Security Policies and End-to-End Testing

## Lab Overview

**Duration:** 45 minutes  
**Difficulty:** Intermediate-Advanced  
**Prerequisites:** Completed Labs 1, 2, and 3

### Objective

Configure FortiGate firewall policies to enable secure connectivity for all traffic flows - north-south (internet access) and east-west (inter-VNet communication and intra-VNet communication). Validate complete end-to-end functionality, test HA failover scenarios, and use FortiView for traffic visualization and troubleshooting.

### What You'll Build

By the end of this lab, you will have:

- ✅ FortiGate address objects for all spoke VNets
- ✅ Static routes for spoke network reachability
- ✅ Firewall policy for internet access (north-south)
- ✅ Firewall policies for inter-VNet and intra-VNet traffic (east-west)
- ✅ NAT configuration for internet-bound traffic
- ✅ Working internet connectivity from all spokes
- ✅ Working inter-spoke communication with inspection
- ✅ Traffic logs and FortiView visualization
- ✅ Verified HA failover functionality

### Architecture

Final Architecture After Lab 4:

```text
                    Internet
                       ▲
                       │
                  External LB
               (Public IP egress)
                       ▲
                       │
         ┌─────────────┴─────────────┐
         │   FortiGate HA Cluster    │
         │   ┌───────────────────┐   │
         │   │ FGT-A ◄──► FGT-B  │   │
         │   │ Active    Passive │   │
         │   └───────────────────┘   │
         │                           │
         │  Policies:                │
         │  • Internet Access (NAT)  │
         │  • East-West Inspection   │
         │                           │
         │   Internal LB: 10.100.2.4 │
         └─────────────┬─────────────┘
                       │
              All traffic forced here
                  (via UDRs)
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Frontend      Backend        Database
   10.101.x       10.102.x      10.103.x
        │              │              │
   VM-1: .1.4    VM: .1.4       VM: .1.4
   VM-2: .1.5
        │              │              │
    ✓ Internet    ✓ Internet    ✓ Internet
    ✓ VM-1↔VM-2*  ✓ Frontend    ✓ Backend
    ✓ Backend     ✓ Database    ✓ Frontend
    ✓ Database
    
All traffic inspected, logged, and controlled
*Intra-VNet traffic only inspected if micro-segmentation enabled
```

### Business Context

Redwood Industries is ready to activate their multi-tier application:

**Requirements:**

- Web tier (Frontend) needs internet access for customer requests
- App tier (Backend) needs to communicate with Frontend and Database
- Database tier needs controlled access from Backend only
- All internet-bound traffic must use NAT (private IPs hidden)
- Complete traffic logging for compliance and troubleshooting

**Success Criteria:**

- Application tiers can communicate securely
- Internet access working from all tiers
- No direct spoke-to-spoke routing (all through FortiGate)
- HA failover completes in <15 seconds
- FortiView shows all traffic flows

---

## Part A: Configure Address Objects and Routes

### Step 1: Access FortiGate Web GUI

#### 1.1 Login to Active FortiGate

1. **Navigate to FortiGate A:**
   - Open browser: `https://[FGT-A-Management-Public-IP]`
   - Login with `fortiuser` credentials from Lab 1

2. **Verify Active Status:**
   - Dashboard should show **HA Status: Primary (Active)**
   - If showing **Secondary**, use FortiGate B instead

> [!NOTE]
> All configuration should be done on the **Primary** unit only. HA will automatically sync to passive unit.

---

### Step 2: Create Address Objects

Address objects provide reusable definitions for networks in firewall policies.

#### 2.1 Create Frontend VNet Address Object

1. **Navigate to Address Objects:**
   - **Policy & Object > Addresses**
   - Click **Address tab > Create New**

2. **Configure Frontend Address:**
   - **Name:** `Redwood-Frontend-VNet`
   - **Interface:** `port2` (Internal interface)
   - **Type:** `Subnet`
   - **IP/Netmask:** `10.101.0.0/16`
   - **Routing Configuration:** ✓ Enable
   - Click **OK**

#### 2.2 Create Backend VNet Address Object

1. **Create New Address:**
   - Click **Create New**

2. **Configure:**
   - **Name:** `Redwood-Backend-VNet`
   - **Interface:** `port2`
   - **Type:** `Subnet`
   - **Subnet / IP Range:** `10.102.0.0/16`
   - **Routing Configuration:** ✓ Enable
   - Click **OK**

#### 2.3 Create Database VNet Address Object

1. **Create New Address:**
   - Click **Create New**

2. **Configure:**
   - **Name:** `Redwood-Database-VNet`
   - **Interface:** `port2`
   - **Type:** `Subnet`
   - **Subnet / IP Range:** `10.103.0.0/16`
   - **Routing Configuration:** ✓ Enable
   - Click **OK**

#### 2.4 Create Address Group for All Spokes

1. **Navigate to Address Groups:**
   - **Policy & Objects > Addresses**
   - Click **Address Group > Create New**

2. **Configure Group:**
   - **Group Name:** `Redwood-All-Spokes`
   - **Type:** `Folder`
   - **Members:** Add all three spoke addresses:
     - `Redwood-Frontend-VNet`
     - `Redwood-Backend-VNet`
     - `Redwood-Database-VNet`
   - **Routing Configuration:** ✓ Enable
   - Click **OK**

> [!NOTE]
> Address groups simplify policy creation when multiple networks need the same treatment.

### Validation - Address Objects

- ✅ Three spoke VNet addresses created
- ✅ All configured with interface port2
- ✅ Address group created with all spokes
- ✅ Visible in **Policy & Objects > Addresses**

---

### Step 3: Create Static Routes to Spoke VNets

FortiGate needs routes to reach the spoke VNets through Azure's native routing (VNet peering).

#### 3.1 Understanding the Routing

**Traffic Flow from FortiGate TO Spokes:**

```text
FortiGate port2 (10.100.2.x)
   ↓
Sends to Azure default gateway (10.100.2.1)
   ↓
Azure routing via VNet peering
   ↓
Spoke VNet destination
```

**Key Point:** 

- Next-hop is Azure's default gateway for the Internal subnet: **10.100.2.1**
- This is NOT the Internal Load Balancer (10.100.2.4)
- The ILB (10.100.2.4) is only used for traffic FROM spokes TO FortiGate
- FortiGate to spokes uses normal Azure routing through the default gateway

**Why 10.100.2.1?**
- Azure reserves .1 in every subnet as the default gateway
- This is Azure's virtual router
- Handles VNet peering routing automatically
- FortiGate sends traffic here, Azure routes it to the correct spoke via peering

**Complete Bidirectional Traffic Flow:**

```text
Traffic FROM Spoke TO FortiGate:
═══════════════════════════════════
Frontend VM (10.101.1.4)
   ↓ UDR: 0.0.0.0/0 → 10.100.2.4
Internal LB (10.100.2.4) ← UDR sends here
   ↓ Health probe determines active unit
Active FortiGate port2 receives packet


Traffic FROM FortiGate TO Spoke:
═══════════════════════════════════
Active FortiGate port2 sends packet
   ↓ Static route: 10.101.0.0/16 → 10.100.2.1
Azure Default Gateway (10.100.2.1) ← FortiGate sends here
   ↓ Azure native routing via VNet peering
Frontend VM (10.101.1.4) receives packet
```

**Key Asymmetry:**
- **Inbound (spoke→FortiGate):** Via ILB (10.100.2.4) - for HA support
- **Outbound (FortiGate→spoke):** Via Azure gateway (10.100.2.1) - native routing

This is normal and expected in Azure NVA architectures!

#### 3.2 Create Route to Frontend VNet

1. **Navigate to Static Routes:**
   - **Network > Static Routes**
   - Click **Create New**

2. **Configure Route:**
   - **Destination:** Select **Named Address**. from dropdown
   - **Named Address:** Select `Redwood-Frontend-VNet`
   - **Gateway Address:** **Specify:**`10.100.2.1`
     - This is Azure's default gateway for Internal subnet
   - **Interface:** `port2`
   - **Administrative Distance:** `10` (default)
   - **Comment:** `Route to Frontend VNet via Azure routing`
   - Click **OK**

#### 3.3 Create Route to Backend VNet

1. **Create New Route:**
   - Click **Create New**

2. **Configure:**
   - **Destination:** Named Address > `Redwood-Backend-VNet`
   - **Gateway Address:** `10.100.2.1`
   - **Interface:** `port2`
   - **Comment:** `Route to Backend VNet via Azure routing`
   - Click **OK**

#### 3.4 Create Route to Database VNet

1. **Create New Route:**
   - Click **Create New**

2. **Configure:**
   - **Destination:** Named Address > `Redwood-Database-VNet`
   - **Gateway Address:** `10.100.2.1`
   - **Interface:** `port2`
   - **Comment:** `Route to Database VNet via Azure routing`
   - Click **OK**

### Validation - Static Routes

1. **Verify Routes Created:**
   - **Network** → **Static Routes**
   - Should see 3 new routes for spoke VNets
   - All with gateway 10.100.2.1 via port2

2. **Check Routing Table:**
   - Navigate to **Dashboard** → **Network** widget
   - Click **Static & Dynamic Routing**
   - Or use CLI: `get router info routing-table all`
   - Verify routes appear as active

---

## Part B: Configure North-South Traffic (Internet Access)

### Step 4: Create Internet Access Policy

This policy allows all spokes to access the internet with NAT.

#### 4.1 Navigate to Firewall Policy

1. **Open Policies:**
   - **Policy & Objects > Firewall Policy**
   - Review existing policies (likely only implicit deny)

#### 4.2 Create Internet Access Policy

1. **Create New Policy:**
   - Click **+ Create new**

2. **Configure Policy:**
   - **Name:** `Spokes-to-Internet`
   - **Incoming Interface:** `port2` (Internal)
   - **Outgoing Interface:** `port1` (External)
   - **Source:**
     - Click **+** button
     - Select `Redwood-All-Spokes` (the group we created)
     - Click **Close**
   - **Destination:** `all`
   - **Schedule:** `always`
   - **Service:** `ALL`
   - **Action:** `ACCEPT`

3. **Configure NAT:**
   - Scroll down to **NAT** section
   - **NAT:** Toggle to **ON** (enabled)
   - **IP Pool Configuration:** Use Outgoing Interface Address

4. **Configure Logging:**
   - Scroll to **Logging Options**
   - **Log Allowed Traffic:** `All Sessions`
   - This logs every connection for visibility

5. **Click OK**

### Understanding This Policy

**What it does:**

- Allows any traffic from spoke VNets (source: Redwood-All-Spokes)
- To any internet destination (destination: all)
- Via port1 (External interface)
- With NAT enabled (translates private IPs to FortiGate port1's IP)

**NAT Operation:**

```text
Frontend VM (10.101.1.4) → Wants to reach 8.8.8.8
   ↓
FortiGate receives on port2 (via Internal LB)
   ↓
Policy matches: Spokes-to-Internet
   ↓
NAT translates source IP:
   From: 10.101.1.4
   To: FortiGate port1 IP (10.100.1.x)
   ↓
Sends to internet directly to Azure gateway via port1 (bypasses External LB)
   ↓
Azure reaches out the destinatio 8.8.8.8
   ↓
Response comes back to Azure gateway
   ↓
Response comes back to port1 IP
   ↓
FortiGate de-NATs back to 10.101.1.4
   ↓
Returns to Frontend VM via port2
```

**External Load Balancer Role:**

- **NOT used for outbound traffic** (spokes → internet)
- **Only used for inbound traffic** (internet → spokes)
- External LB is for incoming connections like VIPs or published services
- Outbound traffic exits directly through port1's IP address

### Validation - Internet Policy

- ✅ Policy created and enabled
- ✅ NAT is ON (blue toggle)
- ✅ Logging enabled for all sessions
- ✅ Policy order shows at top (processed first)

---

### Step 5: Test Internet Connectivity

#### 5.1 Access Frontend VM 1

**Method: Azure Serial Console**

1. **Navigate to Frontend VM:**
   - Azure Portal > **Redwood-Frontend-RG**
   - Click **Redwood-Frontend-VM-1**

2. **Open Serial Console:**
   - Click **Help > Serial console** (left menu)
   - Wait for console to initialize (30 seconds)
   - Press **Enter** to get login prompt

3. **Login:**
   - Username: `azureuser`
   - Password: [your password from Lab 2]

#### 5.2 Test Internet Access

**From Frontend VM:**

**Test DNS Resolution:**

```bash
# Test name resolution
dig www.google.com

# Expected: Resolves to IP succeed
```

**Test HTTPS:**
```bash
# Test web access
curl -I https://www.fortinet.com

# Expected: HTTP 200 OK response
```

#### 5.3 Test from Other Spokes

**Method: SSH session from FortiGate CLI**

**Backend VM:**

1. From FortiGate CLI, SSH to Backend:

   ```bash
   execute ssh azureuser@10.102.1.4
   ```

2. Test internet:

   ```bash
   curl -I https://www.microsoft.com
   ```

**Database VM:**

1. Exit Backend VM: `exit`
2. SSH to Database:

   ```bash
   execute ssh azureuser@10.103.1.4
   ```

3. Test internet:

   ```bash
   curl -I https://www.fortinet.com
   ```

### Validation - Internet Connectivity

- ✅ Frontend VM can reach internet
- ✅ Backend VM can reach internet
- ✅ Database VM can reach internet
- ✅ DNS resolution working
- ✅ HTTPS connections successful

---

### Step 6: Verify Traffic Logs

#### 6.1 Check Forward Traffic Logs

1. **Navigate to Logs:**
   - In FortiGate GUI: **Log & Report** → **Forward Traffic**

2. **Observe Logged Sessions:**
   - You should see entries for your curl tests
   - **Source IP:** 10.101.1.4, 10.102.1.4, 10.103.1.4
   - **Destination IP:** Fortinet/Microsoft IPs
   - **Policy:** Spokes-to-Internet
   - **Action:** accept
   - **NAT:** Yes (source NAT applied)

3. **Filter Traffic:**
   - Click **Advanced Filters**
   - **Source IP:** Enter `10.101.1.4`
   - Click **Apply**
   - Shows only Frontend VM traffic

4. **Examine Log Details:**
   - Click on any log entry
   - Review fields:
     - **Sent/Received bytes:** Shows data transferred
     - **Source NAT IP:** FortiGate's External LB Public IP
     - **Duration:** Session length
     - **Application:** Detected application (PING, HTTPS, etc.)

### Understanding Log Fields

**Key Fields:**
- **srcip:** Original source IP (spoke VM)
- **srcport:** Source port (random high port)
- **dstip:** Destination IP (internet)
- **dstport:** Destination port (80, 443, etc.)
- **trandisp:** Action (accept/deny)
- **policyid:** Which policy matched
- **nat:** NAT applied (yes/no)
- **sentbyte/rcvdbyte:** Traffic volume

> [!IMPORTANT] Challenge ⚔️
> Where you able to find the DNS request in the logs? Why?

---

## Part C: Configure East-West Traffic (Inter-VNet)

### Step 7: Create Inter-VNet Communication Policies

These policies allow spoke VNets to communicate with each other through FortiGate inspection.

#### 7.1 Create Frontend-to-Backend Policy

1. **Navigate to Firewall Policy:**
   - **Policy & Objects > Firewall Policy**
   - Click **+ Create New**

2. **Configure Policy:**
   - **Name:** `Frontend-to-Backend`
   - **Incoming Interface:** `port2`
   - **Outgoing Interface:** `port2`
     - ⚠️ **Same interface** - this is east-west traffic
   - **Source:** `Redwood-Frontend-VNet`
   - **Destination:** `Redwood-Backend-VNet`
   - **Schedule:** `always`
   - **Service:** `ALL`
   - **Action:** `ACCEPT`

3. **NAT Configuration:**
   - **NAT:** **OFF** (disabled)
   - ⚠️ **Critical:** No NAT for east-west traffic
   - Private IPs should remain visible between VNets

4. **Logging:**
   - **Log Allowed Traffic:** `All Sessions`

5. Click **OK**

#### 7.2 Create Backend-to-Frontend Policy

1. **Create New Policy:**
   - Click **Create New**

2. **Configure (Reverse Direction):**
   - **Name:** `Backend-to-Frontend`
   - **Incoming Interface:** `port2`
   - **Outgoing Interface:** `port2`
   - **Source:** `Redwood-Backend-VNet`
   - **Destination:** `Redwood-Frontend-VNet`
   - **Service:** `ALL`
   - **Action:** `ACCEPT`
   - **NAT:** **OFF**
   - **Log Allowed Traffic:** `All Sessions`

3. Click **OK**

#### 7.3 Create Additional Inter-VNet Policies

**Create these policies following the same pattern:**

**Backend-to-Database:**
- Source: `Redwood-Backend-VNet`
- Destination: `Redwood-Database-VNet`
- NAT: OFF

**Database-to-Backend:**
- Source: `Redwood-Database-VNet`
- Destination: `Redwood-Backend-VNet`
- NAT: OFF

> [!TIP]
> **Quick Method:** Create first policy, then right-click > **Copy** and then right-click > **Paste > Below**. Edit the new policy and modify source/destination for faster creation.

### Understanding East-West Policies

**Why port2 → port2?**

- Traffic enters FortiGate on port2 from source VNet
- FortiGate inspects and applies policy
- Traffic exits FortiGate on same port2 toward destination VNet
- This is called "hairpin routing" or "U-turn routing"

**Why No NAT?**

- VNets are private networks
- Source IPs should remain visible for troubleshooting
- Application logs show real source IP
- NAT only needed for internet-bound traffic

### Validation - East-West Policies

- ✅ 6 inter-VNet policies created (bidirectional for 3 VNets)
- ✅ All policies use port2 → port2
- ✅ NAT disabled on all east-west policies
- ✅ Logging enabled on all policies

---

### Step 7.4: Configure Intra-VNet Communication (Optional Micro-Segmentation)

If you have multiple VMs in the same VNet (like we have: Redwood-Frontend-VM-1 and Redwood-Frontend-VM-2), by default they can communicate directly without going through FortiGate because they're in the same subnet.

**To enable FortiGate inspection for intra-VNet traffic (micro-segmentation):**

#### 7.4.1 Add Intra-VNet Route to UDR (Azure Portal)

1. **Navigate to Frontend Route Table:**
   - Azure Portal → **Redwood-Frontend-RG**
   - Click **Redwood-Frontend-RT**

2. **Add Intra-Subnet Route:**
   - Click **Settings** → **Routes**
   - Click **+ Add**
   - **Route name:** `Frontend-to-Frontend`
   - **Destination type:** `IP Addresses`
   - **Destination IP addresses/CIDR ranges:** `10.101.1.0/24`
   - **Next hop type:** `Virtual appliance`
   - **Next hop address:** `10.100.2.4`
   - Click **Add**

> [!NOTE]
> This forces traffic between VMs in the same subnet through FortiGate. Without this route, VMs communicate directly via Azure's local routing.

#### 7.4.2 Create Intra-VNet Firewall Policy

1. **Navigate to Firewall Policy:**
   - **Policy & Object > Firewall Policy**
   - Click **Create New**

2. **Configure Policy:**
   - **Name:** `Frontend-Intra-VNet`
   - **Incoming Interface:** `port2`
   - **Outgoing Interface:** `port2`
   - **Source:** `Redwood-Frontend-VNet`
   - **Destination:** `Redwood-Frontend-VNet`
     - ⚠️ **Same VNet in both source and destination**
   - **Service:** `ALL`
   - **Action:** `ACCEPT`
   - **NAT:** **OFF**
   - **Log Allowed Traffic:** `All Sessions`

3. Click **OK**

#### 7.4.3 Test Intra-VNet Communication

**From Redwood-Frontend-VM-1:**

```bash
# Test connectivity to VM in same VNet
ping -c 3 10.101.1.5

# Expected: Successful pings through FortiGate
```

**Verify in FortiGate Logs:**
- **Log & Report > Forward Traffic**
- Filter: Source = 10.101.1.4, Destination = 10.101.1.5
- **Policy:** Should show "Frontend-Intra-VNet"
- **Interfaces:** port2 → port2 (hairpin)

### Understanding Micro-Segmentation

**Without Intra-VNet Route:**

```text
VM-1 (10.101.1.4) → VM-2 (10.101.1.5)
Direct communication via Azure local routing
No FortiGate inspection
```

**With Intra-VNet Route:**

```text
VM-1 (10.101.1.4)
   ↓ UDR: 10.101.1.0/24 → 10.100.2.4
Internal LB (10.100.2.4)
   ↓
FortiGate inspection
   ↓
VM-2 (10.101.1.5)
```

**Benefits:**

- Zero-trust: Even VMs in same subnet are inspected
- Granular control: Can allow/deny specific VM-to-VM traffic
- Compliance: Some regulations require all traffic inspection

**Trade-offs:**

- Increased latency (extra hop through FortiGate)
- Higher FortiGate resource usage
- More complex troubleshooting

> [!TIP] Recommendation:
>
> Use micro-segmentation when:
>
> - Compliance requires all traffic inspection
> - Need to isolate VMs within same tier
> - Zero-trust architecture mandated
>
> Skip micro-segmentation when:
>
> - VMs in same tier fully trust each other
> - Performance is critical
> - Simpler troubleshooting preferred

---

### Step 8: Test Inter-VNet Connectivity

#### 8.1 Test Frontend to Backend

**From Frontend VM 1:**

```bash
# Test connectivity to Backend VM
ping -c 3 10.102.1.4

# Expected: Successful pings
```

**Expected Result:**
```text
PING 10.102.1.4 (10.102.1.4) 56(84) bytes of data.
64 bytes from 10.102.1.4: icmp_seq=1 ttl=63 time=3.2 ms
64 bytes from 10.102.1.4: icmp_seq=2 ttl=63 time=2.8 ms
64 bytes from 10.102.1.4: icmp_seq=3 ttl=63 time=3.0 ms

--- 10.102.1.4 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss
```

> [!NOTE]
> TTL is 63 (not 64) because packet went through FortiGate (one hop)

#### 8.2 Test Frontend to Database

**From Frontend VM:**

```bash
# Test connectivity to Database VM
ping -c 3 10.103.1.4

# Expected: Fail pings
```

> [!NOTE] No communication from the Frontend to the Database

#### 8.3 Test Backend to Database

**From Backend VM:**

1. **SSH to Backend:**

   ```bash
   # From FortiGate CLI or Frontend VM
   ssh azureuser@10.102.1.4
   ```

2. **Test Database:**

   ```bash
   ping -c 3 10.103.1.4
   
   # Expected: Success pings
   ```

#### 8.4 Test Complete Connectivity Matrix

Create a connectivity matrix to verify all paths:

| From | To | Expected | Test Command |
|------|-----|----------|--------------|
| Frontend-VM-1 | Frontend-VM-2 | ✅ Works * | `ping 10.101.1.5` |
| Frontend-VM-1 | Backend | ✅ Works | `ping 10.102.1.4` |
| Frontend-VM-1 | Database | ❌ Fail by design | `ping 10.103.1.4` |
| Frontend-VM-2 | Frontend-VM-1 | ✅ Works * | `ping 10.101.1.4` |
| Frontend-VM-2 | Backend | ✅ Works | `ping 10.102.1.4` |
| Frontend-VM-2 | Database | ❌ Fail by design  | `ping 10.103.1.4` |
| Backend | Frontend-VM-1 | ✅ Works | `ping 10.101.1.4` |
| Backend | Frontend-VM-2 | ✅ Works | `ping 10.101.1.5` |
| Backend | Database | ✅ Works | `ping 10.103.1.4` |
| Database | Frontend-VM-1 | ❌ Fail by design  | `ping 10.101.1.4` |
| Database | Frontend-VM-2 | ❌ Fail by design  | `ping 10.101.1.5` |
| Database | Backend | ✅ Works | `ping 10.102.1.4` |

*Intra-VNet communication (VM-1 ↔ VM-2) only goes through FortiGate if you configured Step 7.4 (micro-segmentation). Without it, these VMs communicate directly via Azure local routing.

### Validation - Inter-VNet Connectivity

- ✅ All spoke-to-spoke connections working
- ✅ TTL shows packet went through FortiGate (63 not 64)
- ✅ Bi-directional connectivity confirmed
- ✅ Traffic being inspected and logged

---

### Step 9: Verify East-West Traffic in Logs

#### 9.1 Check Inter-VNet Traffic Logs

1. **Navigate to Forward Traffic:**
   - **Log & Report > Forward Traffic**

2. **Filter for East-West Traffic:**
   - **Advanced Filters**
   - **Source IP:** `10.101.1.4`
   - **Destination IP:** `10.102.1.4`
   - Click **Apply**

3. **Observe Log Entries:**
   - **Policy:** Frontend-to-Backend
   - **Source Interface:** port2
   - **Destination Interface:** port2 (same)
   - **NAT:** No (source IP preserved)
   - **Action:** accept

4. **Verify No NAT Applied:**
   - Check **srcip** field = original IP (10.101.1.4)
   - No NAT translation visible
   - This confirms east-west policy working correctly

---

## Part D: FortiView Traffic Visualization

### Step 10: Use FortiView for Traffic Analysis

FortiView provides real-time dashboards for understanding traffic patterns.

#### 10.1 Access FortiView

1. **Navigate to FortiView:**
   - **Dashboard** → **FortiView Sessions**
   - Or **FortiView** in left menu

#### 10.2 Explore Sources Dashboard

1. **View Traffic Sources:**
   - You should see active sources:
     - 10.101.1.4 (Frontend VM-1)
     - 10.101.1.5 (Frontend VM-2)
     - 10.102.1.4 (Backend VM)
     - 10.103.1.4 (Database VM)

2. **Click on a Source:**
   - Click **10.101.1.4**
   - See all destinations this source communicated with:
     - 10.101.1.5 (Frontend VM-2, if micro-segmentation enabled)
     - 10.102.1.4 (Backend)
     - 10.103.1.4 (Database)

3. **Drill Down:**
   - Click on any destination
   - Select **View session logs**
   - See detailed traffic flows

#### 10.3 Explore Destinations Dashboard

1. **Switch to Destinations View:**
   - **FortiView** → **Destinations**

2. **Find Internet Destinations:**
   - You should see:
     - 1.1.1.1 (Cloudflare DNS)
     - Other IPs from curl tests
   - Click on any destination to see sources

#### 10.4 Explore Policy View

1. **Switch to Policies:**
   - **FortiView** → **Policies**

2. **View Policy Statistics:**
   - **Spokes-to-Internet:**
     - Bytes sent/received
     - Session count
     - Top sources using this policy
   - **Inter-VNet Policies:**
     - Similar statistics for east-west traffic

### Understanding FortiView

**Business Value:**
- **Troubleshooting:** Quickly identify connectivity issues
- **Capacity Planning:** See bandwidth usage by source/destination
- **Security:** Identify unusual traffic patterns
- **Compliance:** Visual proof of traffic inspection

**Key Views:**
- **Sources:** Who is generating traffic
- **Destinations:** Where traffic is going
- **Applications:** What applications detected
- **Policies:** Which policies are being used
- **Threats:** Security events (if UTM enabled)

---

## Part E: High Availability Testing

### Step 11: Test HA Failover

#### 11.1 Verify Current HA Status

1. **Check HA Dashboard:**
   - **System > HA**
   - **Status** tab
   - Note which unit is Primary (Active)
   - Note which unit is Secondary (Passive)

2. **Record Current Master:**
   - Example: `Redwood-Hub-FGT-A` is Primary
   - We'll force failover to FGT-B

#### 11.2 Generate Active Traffic

**In the Serial Console from Frontend VM-1:**

1. Install hping3

   ```bash
   sudo apt update
   sudo apt-get install -y hping3
   ```
   
2. Start a continous traffic

```bash
# Start continuous traffic
sudo hping3 -S -p 80 server.bytesec.ca

# Leave this running during failover test
# Don't stop - we're measuring packet loss
```

#### 11.3 Fail over

1. **From Redwood Hub Group:**
   - Navigate to **Redwood-Hub-RG**
   - Click on **Redwood-Hub-FGT-A** resource (Primary)
   - Click on `Stop` to stop the instance.
  
Once the primary instance is stopped, it will trigger an failover and the other instance will become the primary one.

#### 11.4 Monitor Failover Process

**Watch the Frontend VM ping output:**

```text
azureuser@Redwood-Frontend-VM:~$ sudo hping3 -S -p 80 server.bytesec.ca
HPING server.bytesec.ca (eth0 100.49.15.72): S set, 40 headers + 0 data bytes
len=46 ip=100.49.15.72 ttl=50 DF id=0 sport=80 flags=SA seq=0 win=62727 rtt=22.9 ms
len=46 ip=100.49.15.72 ttl=49 DF id=0 sport=80 flags=SA seq=1 win=62727 rtt=26.1 ms
len=46 ip=100.49.15.72 ttl=49 DF id=0 sport=80 flags=SA seq=2 win=62727 rtt=21.7 ms
len=46 ip=100.49.15.72 ttl=49 DF id=0 sport=80 flags=SA seq=3 win=62727 rtt=22.6 ms   ← Failover starts
len=46 ip=100.49.15.72 ttl=49 DF id=0 sport=80 flags=SA seq=15 win=62727 rtt=22.3 ms  ← New unit taking over
len=46 ip=100.49.15.72 ttl=50 DF id=0 sport=80 flags=SA seq=16 win=62727 rtt=20.3 ms
```

**Count Lost Packets:**
- Typically 10-15 packets lost
- At 1 second intervals = ~10-15 seconds downtime
- Within acceptable SLA (<15 seconds)

#### 11.5 Verify New Active Unit

1. **Check HA Status:**
   - On FGT-B (should now be Primary)
   - **System > HA > Status**
   - Verify role changed to Master

2. **Check Azure Load Balancer:**
   - Azure Portal → **Redwood-Hub-internalloadbalancer**
   - **Insights** (under Monitoring)
   - Should show FGT-B as healthy
   - FGT-A may show unhealthy

> [!NOTE]
> It can take up to 10 minutes to Azure update the status of the loadbalancer backends

3. **Verify Traffic Flowing:**
   - Hping should be working again
   - Check FortiView - traffic on FGT-B
   - Logs showing on new active unit

#### 11.6 Restore Original Configuration

**Start FGT-A:**

1. **From Redwood Hub Group:**
   - Navigate to **Redwood-Hub-RG**
   - Click on **Redwood-Hub-FGT-A** resource (Primary)
   - Click on `Start` to start the instance.

2. **Wait for Synchronization:**
   - FGT-A rejoins as Secondary
   - Health probes restore
   - HA cluster back to normal

3. **Optional - Fail Back to FGT-A:**
   - If desired, trigger failover again
   - Use this command to trigger a failover: `exec ha failover set 1`
   - FGT-A becomes Primary again
   - Or leave FGT-B as Primary (both valid)

### Understanding HA Failover

**Failover Triggers:**
- Device failure (VM crash)
- Health probe failure
- Manual failover

**Failover Process:**
```text
1. Active unit failure detected (<1 second)
2. Passive assumes active role (~1 second)
3. Begins responding to health probes (~2-3 seconds)
4. Load balancer updates backend pool (~3-5 seconds)
5. Traffic redirected to new active unit
Total: ~5-10 seconds typical
```

**Session Handling:**
- New sessions: Routed to new active immediately
- Existing sessions: Rebuilt (some packet loss acceptable)
- Session sync can reduce impact (advanced configuration)

### Validation - HA Failover

- ✅ Failover completed successfully
- ✅ Packet loss: 10-15 packets (~10-15 seconds)
- ✅ New active unit processing traffic
- ✅ FortiView showing traffic on new primary
- ✅ Original unit restored and synchronized

---

## Lab 4 Complete! 🎉🎉🎉

### What You've Accomplished

You have successfully configured and validated a complete enterprise-grade Azure security architecture:

✅ **Address Objects Created:**

- All three spoke VNets defined
- Address group for simplified management

✅ **Static Routes Configured:**

- Routes to all spoke VNets
- Via Azure's default gateway (10.100.2.1)

✅ **North-South Traffic Working:**

- Internet access from all spokes
- NAT configured for private IP translation
- All traffic logged

✅ **East-West Traffic Working:**

- All spokes can communicate
- Bidirectional connectivity verified
- No NAT (private IPs preserved)

✅ **Traffic Inspection Operational:**

- All traffic flows through FortiGate
- Complete logging and visibility
- FortiView providing real-time dashboards

✅ **High Availability Validated:**

- Failover tested successfully
- Downtime < 15 seconds
- Automatic recovery verified

### Complete Architecture Achieved

```text
                    Internet ✅
                       ▲
                  External LB
               (Public IP + NAT)
                       ▲
         ┌─────────────┴─────────────┐
         │   FortiGate HA Cluster    │
         │   ┌───────────────────┐   │
         │   │ FGT-A ◄──► FGT-B  │   │
         │   │  HA Sync Working  │   │
         │   └───────────────────┘   │
         │                           │
         │  ✅ Policies Active       │
         │  ✅ NAT Configured        │
         │  ✅ Logging Enabled       │
         │                           │
         │   Internal LB: 10.100.2.4 │
         └─────────────┬─────────────┘
                       │
              ✅ All traffic inspected
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Frontend ✅    Backend ✅      Database ✅
   10.101.x       10.102.x        10.103.x
        │              │              │
   Internet ✅    Internet ✅     Internet ✅
   Backend ✅     Frontend ✅     Frontend ✅
   Database ✅    Database ✅     Backend ✅

Redwood Industries Multi-Tier Application:
✅ Fully operational
✅ Completely secured
✅ Highly available
✅ Production-ready
```

### Key Takeaways

1. **Centralized Security Architecture:**
   - Single FortiGate HA cluster protects all VNets
   - Scales to 100+ spoke VNets without adding firewalls
   - Operational efficiency through centralization

2. **Traffic Flow Control:**
   - UDRs force all traffic through FortiGate
   - Both north-south and east-west inspected
   - No traffic bypasses security inspection

3. **NAT for Internet, No NAT for Internal:**
   - Internet traffic: NAT required (hide private IPs)
   - Inter-VNet traffic: No NAT (preserve source IP)
   - This pattern essential for proper logging and troubleshooting

4. **High Availability Delivers:**
   - No single point of failure
   - Sub-15-second failover time
   - Meets enterprise SLA requirements
   - Azure Load Balancer integration seamless

5. **FortiView Provides Visibility:**
   - Real-time traffic dashboards
   - Essential for troubleshooting
   - Compliance and audit trail
   - Capacity planning insights

### Business Value Delivered

**For Redwood Industries:**

✅ **Security Requirements Met:**

- All inter-tier and intra-tier traffic inspected ✅
- Complete logging for compliance ✅
- Network segmentation enforced ✅
- Zero-trust architecture achieved ✅

✅ **Operational Goals Achieved:**

- 99.9% uptime SLA met (HA failover <15s) ✅
- Centralized management (single pane of glass) ✅
- Scalable to support growth ✅
- Operational consistency with on-prem ✅

✅ **Cost Optimization:**

- 67% cost savings vs Azure Firewall ✅
- Single HA cluster vs multiple firewalls ✅
- BYOL licensing flexibility ✅

### Workshop Complete!

**You can now:**

- ✅ Design hub-spoke topologies with FortiGate HA
- ✅ Deploy FortiGate Active-Passive HA in Azure
- ✅ Configure User-Defined Routes for traffic steering
- ✅ Create firewall policies for north-south traffic
- ✅ Implement east-west traffic inspection
- ✅ Use FortiView for traffic analysis
- ✅ Troubleshoot connectivity issues
- ✅ Test and validate HA failover
- ✅ Articulate business value to customers

### Next Steps in Your Journey

**Immediate Actions:**

1. **Document Your Environment:**

   - Screenshot your working architecture
   - Save firewall policy configurations
   - Export FortiGate config as backup

2. **Experiment Further (Optional):**

   - Add security profiles (IPS, Web Filtering)
   - Configure application control policies
   - Set up FortiAnalyzer for logging
   - Test different traffic patterns

3. **Clean Up Resources:**

   - ⚠️ **Important:** Delete all resource groups to avoid charges
   - Go to each RG → **Delete resource group**
   - Type resource group name to confirm
   - Delete in this order:
     1. Redwood-Frontend-RG
     2. Redwood-Backend-RG
     3. Redwood-Database-RG
     4. Redwood-Hub-RG (last - contains FortiGate)

**Continue Learning:**

- **AZ-103:** FortiGate HA - Active/Active, FAZ & FMG on Azure
- **NSE 4:** FortiOS Administrator

### Feedback & Support

**Enjoyed the workshop?**
- Share feedback with your instructor
- Complete the workshop survey
- Connect with peers on LinkedIn
- Join Fortinet Community forums

**Need Help Post-Workshop?**
- Fortinet Community: https://community.fortinet.com
- Azure Support: For Azure-specific issues
- Partner SE: Your Fortinet Systems Engineer

---

## Troubleshooting Reference

### Issue: Internet Access Not Working

**Checklist:**

1. ✓ Firewall policy "Spokes-to-Internet" exists and enabled
2. ✓ NAT is ON (blue toggle) in the policy
3. ✓ Policy order: Should be near top (processed first)
4. ✓ Source matches spoke VNet addresses
5. ✓ UDR points to correct Internal LB IP (10.100.2.4)

**Debug Commands:**

```bash
# On FortiGate CLI
diagnose debug flow trace start 10
diagnose debug flow filter addr 10.101.1.4
diagnose debug enable

# Generate traffic from VM
# Watch FortiGate CLI output

diagnose debug disable
```

### Issue: Inter-VNet Connectivity Fails

**Checklist:**

1. ✓ Bidirectional policies exist (A→B AND B→A)
2. ✓ Both policies use port2 → port2
3. ✓ NAT is OFF on east-west policies
4. ✓ Static routes exist for both VNets
5. ✓ UDRs configured on both spoke VNets

**Test from FortiGate:**

```bash
# Can FortiGate reach the VMs?
execute ping 10.101.1.4
execute ping 10.102.1.4
execute ping 10.103.1.4

# If FortiGate can't reach, routing problem
# If FortiGate can reach but VMs can't reach each other, policy problem
```

### Issue: NAT Applied When It Shouldn't Be

**Symptom:** East-west traffic shows NAT in logs

**Solution:**

1. Find the inter-VNet policy
2. Edit policy
3. Verify **NAT** toggle is **OFF** (gray, not blue)
4. Click **OK**
5. Test again

### Issue: Logs Not Showing Traffic

**Checklist:**

1. ✓ "Log Allowed Traffic" enabled on policy
2. ✓ Set to "All Sessions" not just "Security Events"
3. ✓ Wait 30-60 seconds for logs to appear
4. ✓ Refresh Log & Report page

**Verify Logging Enabled:**

```bash
# CLI command to show policy
show firewall policy 1

# Look for:
# set logtraffic all
```

### Issue: HA Failover Not Working

**Checklist:**

1. ✓ Both FortiGates showing in HA cluster
2. ✓ HA synchronization status: "In Sync"
3. ✓ Health probes configured on both load balancers
4. ✓ Port 8008 responding on both FortiGates

**Test Health Probe Manually:**

```bash
# From another system
curl -k https://[Internal-LB-IP]:8008

# Should respond when FortiGate is active
```

### Issue: Packet Loss Higher Than Expected

**Normal:** 3-5 packets during failover  
**Concerning:** >10 packets (>10 seconds downtime)

**Possible Causes:**

1. Slow health probe interval (default should be 5 seconds)
2. Network congestion
3. FortiGate overloaded (check CPU)
4. Azure platform issue

**Optimize:**

- Reduce health probe interval to 5 seconds
- Increase probe failures required to 2 (faster detection)

---

## Advanced: Policy Optimization

### Consolidating Policies

**Instead of 6 separate inter-VNet policies, you could create 2:**

**Option A: Allow All Spokes Bidirectional**

```text
Policy 1:
  Source: Redwood-All-Spokes
  Destination: Redwood-All-Spokes
  Interfaces: port2 → port2
  NAT: OFF
```

**Pros:**

- Simpler (1 policy instead of 6)
- Easier to manage

**Cons:**

- Less granular control
- Can't block specific spoke-to-spoke paths
- Harder to troubleshoot (single policy ID for all traffic)

**Recommendation:**

- **For Workshop:** Use separate policies (better learning)
- **For Production:** Depends on requirements
  - Need granular control? Separate policies
  - Simple environment? Consolidated policy acceptable

### Adding Security Profiles

**Beyond Basic Firewall:**

After completing basic policies, you could add:

1. **IPS (Intrusion Prevention):**
   - Blocks known attacks
   - Apply to internet-bound traffic

2. **Web Filtering:**
   - Category-based URL filtering
   - Apply to HTTP/HTTPS traffic

3. **Application Control:**
   - Identify and control applications
   - Block risky apps (P2P, etc.)

4. **Antivirus:**
   - Scan downloaded files
   - Block malware

**To Add to Policy:**

1. Edit policy
2. Scroll to **Security Profiles**
3. Enable desired profiles
4. Click **OK**

> [!NOTE]
> Security profiles require valid FortiGuard subscription and may impact performance. Test in non-production first.

---

## Configuration Backup

### Export FortiGate Configuration

**GUI Method:**

1. **Navigate to Backup:**
   - **System > Settings**
   - **Configuration** section
   - Click **Backup**

2. **Download Config:**
   - **Scope:** Full configuration
   - **Encryption:** Optional (recommended for production)
   - Click **OK**
   - Save .conf file locally

**CLI Method:**

```bash
# Show full configuration
show

# Or export to USB (if attached)
execute backup config management-station <filename>
```

### Configuration Review Checklist

Before completing the workshop, verify your configuration includes:

**Address Objects:**

- [ ] Redwood-Frontend-VNet (10.101.0.0/16)
- [ ] Redwood-Backend-VNet (10.102.0.0/16)
- [ ] Redwood-Database-VNet (10.103.0.0/16)
- [ ] Redwood-All-Spokes (address group)

**Static Routes:**

- [ ] Route to 10.101.0.0/16 via 10.100.2.1
- [ ] Route to 10.102.0.0/16 via 10.100.2.1
- [ ] Route to 10.103.0.0/16 via 10.100.2.1

**Firewall Policies:**

- [ ] Spokes-to-Internet (NAT ON, port2→port1)
- [ ] 4× Inter-VNet policies (NAT OFF, port2→port2)
- [ ] All policies logging enabled

**HA Configuration:**

- [ ] Mode: Active-Passive
- [ ] 2 members synchronized
- [ ] Heartbeat interface: port3

---

**End of Lab 4 and AZ-102 Workshop**  

*Estimated completion time: 45 minutes*

---

*Lab Guide Version 1.0 - December 2024*  
*Thank you for completing AZ-102!*  
*Questions? Contact your instructor or Fortinet support.*

---

## Congratulations! 🎊

You've successfully completed **AZ-102: FortiGate HA with Multi-VNet Hub-Spoke Architecture**!

You now have the skills to design, deploy, and manage enterprise-grade Azure security infrastructures using FortiGate. These are production-ready skills that you can apply immediately in customer environments.

**Keep learning, stay secure, and build great things!**

*- The Fortinet Azure Workshop Team*
