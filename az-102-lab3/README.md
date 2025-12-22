# Lab 3: User-Defined Routes Configuration

## Lab Overview

**Duration:** 30 minutes  
**Difficulty:** Intermediate  
**Prerequisites:** Completed Labs 1 and 2

### Objective

Configure User-Defined Routes (UDRs) to force all spoke VNet traffic through the FortiGate cluster's Internal Load Balancer. This establishes the critical routing infrastructure that enables centralized security inspection for both north-south (internet) and east-west (inter-VNet) traffic.

### What You'll Build

By the end of this lab, you will have:

- ✅ Three route tables (one per spoke VNet)
- ✅ Default routes pointing to Internal LB (10.100.2.4)
- ✅ Route table associations with spoke subnets
- ✅ All spoke traffic forced through FortiGate HA cluster
- ✅ Understanding of asymmetric routing patterns

### Architecture

After Lab 3:

```text
         Hub VNet (10.100.0.0/16)
      ┌──────────────────────────────┐
      │  FortiGate HA Cluster        │
      │  ┌────────────────────────┐  │
      │  │ Internal LB: 10.100.2.4│◄─┼─── All traffic
      │  │    (Next-Hop)          │  │    forced here
      │  └────────────────────────┘  │    by UDRs
      └──────────────────────────────┘
           ▲         ▲         ▲
           │         │         │
      UDRs force traffic through ILB
           │         │         │
    ┌──────┴───┐ ┌──┴──────┐ ┌┴────────┐
    │ Frontend │ │ Backend │ │Database │
    │  VNet    │ │  VNet   │ │  VNet   │
    │          │ │         │ │         │
    │ RT:      │ │ RT:     │ │ RT:     │
    │ 0/0→ILB  │ │ 0/0→ILB │ │ 0/0→ILB │
    └──────────┘ └─────────┘ └─────────┘

Result: All traffic inspected by FortiGate
(Connectivity broken until Lab 4 policies)
```

### Business Context

**Current Problem (After Lab 2):**

- Spoke VNets are completely isolated from each other
- VNet peering is non-transitive - spokes have no routes to communicate
- While this is secure, applications can't function
- Need controlled connectivity with security inspection

**UDR Solution:**

- Create routes forcing traffic through FortiGate
- Point all spoke traffic to Internal Load Balancer (10.100.2.4)
- ILB distributes to active FortiGate
- Enables spoke-to-spoke communication (once policies added in Lab 4)
- Complete traffic inspection and logging

**Compliance Impact:**

- PCI-DSS requires network segmentation with inspection
- Zero-trust model: inspect all traffic, even internal
- Centralized logging for audit requirements
- UDRs enable inspection while maintaining isolation

---

## Understanding User-Defined Routes

### What are UDRs?

**Azure's Default Routing:**

- VMs in the SAME VNet communicate directly
- VMs can reach peered VNets (but peering is non-transitive)
- Hub-spoke topology: spokes can reach hub, but not each other
- No security inspection possible
- Isolation by default, but limits application connectivity

**User-Defined Routes:**

- Custom routing tables
- Override Azure's default behavior
- Specify next-hop for traffic
- Enable security appliances in traffic path

### Key UDR Concepts

**Route Table:**

- Container for routes
- Associated with subnet(s)
- Affects all VMs in associated subnets

**Next-Hop Types:**

- **Virtual Appliance:** IP address of security device
- **VNet Gateway:** VPN or ExpressRoute gateway
- **Virtual Network:** Azure's default routing
- **Internet:** Direct internet egress
- **None:** Drop traffic (black hole)

**Route Specificity:**

- Most specific route wins (longest prefix match)
- UDRs override Azure system routes
- 0.0.0.0/0 = default route (catch-all)

### Traffic Flow with UDRs

**Before UDRs:**

```text
Frontend VM (10.101.1.4)
   ↓ (wants to reach Backend 10.102.1.4)
Azure checks routing table
   ↓ (finds peering to hub, but NO route to Backend)
   ↓ (peering is non-transitive)
❌ No route exists - packet dropped
   ↓
[Spokes are isolated - cannot communicate]
```

**After UDRs:**

```text
Frontend VM (10.101.1.4)
   ↓ (wants to reach Backend)
Azure checks routing table
   ↓ (finds UDR: 0.0.0.0/0 → 10.100.2.4)
Send to Internal LB (10.100.2.4)
   ↓
Internal LB sends to Active FortiGate
   ↓ (port2 receives packet)
FortiGate inspects, applies policy
   ↓ (if allowed, forwards)
FortiGate sends back to port2
   ↓
Internal LB routes to destination
   ↓
Backend VM (10.102.1.4)
   ↓
[Route now exists - full inspection and logging]
[But blocked until Lab 4 policies added]
```

---

## Part A: Create Frontend Route Table

### Step 1: Create Frontend Route Table

#### 1.1 Navigate to Route Table Creation

1. **From Frontend Resource Group:**
   - Navigate to **Redwood-Frontend-RG**
   - Click **+ Create**
   - Search: `route table`
   - Click **Create > Route table**

#### 1.2 Configure Route Table

1. **Basics Tab:**
   - **Subscription:** Your subscription
   - **Resource group:** `Redwood-Frontend-RG`
   - **Region:** `Canada Central`
   - **Name:** `Redwood-Frontend-RT`
   - **Propagate gateway routes:** `No`

   > [!NOTE]
   > **Propagate gateway routes = No** prevents VPN/ExpressRoute routes from overriding our UDRs

2. Click **Review + create** > **Create**

### Step 2: Add Default Route to Frontend RT

#### 2.1 Navigate to Routes

1. **After creation completes:**
   - Click **Go to resource**
   - Or navigate to **Redwood-Frontend-RG** → **Redwood-Frontend-RT**

2. **Open Routes Configuration:**
   - Click **Setting > Routes** in left menu
   - Click **+ Add**

#### 2.2 Configure Default Route

1. **Add Route:**
   - **Route name:** `Default-to-FortiGate`
   - **Destination type:** `IP Addresses`
   - **Destination IP addresses/CIDR ranges:** `0.0.0.0/0`
   - **Next hop type:** `Virtual appliance`
   - **Next hop address:** `10.100.2.4`

   > [!WARNING]
   > **Critical:** Next-hop must be exactly `10.100.2.4` (Internal LB IP from Lab 1)
   > Any typo here breaks all traffic flow

2. Click **Add**

### Step 3: Associate with Frontend Subnet

#### 3.1 Open Subnet Associations

1. **In Redwood-Frontend-RT:**
   - Click **Settings > Subnets** in left menu
   - Click **+ Associate**

#### 3.2 Configure Association

1. **Associate Subnet:**
   - **Virtual network:** `Redwood-Frontend-VNet`
   - **Subnet:** `Protected-Subnet`
   - Click **OK**

   > [!NOTE]
   > Association takes effect immediately. All VMs in Frontend-VNet/Protected-Subnet now use this route table.

### Validation

- ✅ Route table created: Redwood-Frontend-RT
- ✅ Default route added: 0.0.0.0/0 → 10.100.2.4
- ✅ Associated with Redwood-Frontend-VNet/Protected-Subnet

---

## Part B: Create Backend Route Table

### Step 4: Create Backend Route Table

#### 4.1 Navigate and Create

1. **From Backend Resource Group:**
   - Navigate to **Redwood-Backend-RG**
   - Click **+ Create**
   - Search: `route table`
   - Click **Create > Route table**

#### 4.2 Configure Route Table

1. **Basics:**
   - **Resource group:** `Redwood-Backend-RG`
   - **Region:** `Canada Central`
   - **Name:** `Redwood-Backend-RT`
   - **Propagate gateway routes:** `No`

2. Click **Review + create > Create**

### Step 5: Add Default Route to Backend RT

1. **Navigate to Routes:**
   - Go to resource (or Redwood-Backend-RG > Redwood-Backend-RT)
   - Click **Settings > Routes**
   - Click **+ Add**

2. **Configure Route:**
   - **Route name:** `Default-to-FortiGate`
   - **Destination type:** `IP Addresses`
   - **Destination IP addresses/CIDR ranges:** `0.0.0.0/0`
   - **Next hop type:** `Virtual appliance`
   - **Next hop address:** `10.100.2.4`
   - Click **Add**

### Step 6: Associate with Backend Subnet

1. **Open Subnets:**
   - Click **Settings > Subnets**
   - Click **+ Associate**

2. **Configure:**
   - **Virtual network:** `Redwood-Backend-VNet`
   - **Subnet:** `Protected-Subnet`
   - Click **OK**

### Validation

- ✅ Route table created: Redwood-Backend-RT
- ✅ Default route added: 0.0.0.0/0 → 10.100.2.4
- ✅ Associated with Redwood-Backend-VNet/Protected-Subnet

---

## Part C: Create Database Route Table

### Step 7: Create Database Route Table

#### 7.1 Navigate and Create

1. **From Database Resource Group:**
   - Navigate to **Redwood-Database-RG**
   - Click **+ Create**
   - Search: `route table`
   - Click **Create > Route table**

#### 7.2 Configure Route Table

1. **Basics:**
   - **Resource group:** `Redwood-Database-RG`
   - **Region:** `Canada Central`
   - **Name:** `Redwood-Database-RT`
   - **Propagate gateway routes:** `No`

2. Click **Review + create > Create**

### Step 8: Add Default Route to Database RT

1. **Navigate to Routes:**
   - Go to resource
   - Click **Settings > Routes**
   - Click **+ Add**

2. **Configure Route:**
   - **Route name:** `Default-to-FortiGate`
   - **Destination type:** `IP Addresses`
   - **Destination IP addresses/CIDR ranges:** `0.0.0.0/0`
   - **Next hop type:** `Virtual appliance`
   - **Next hop address:** `10.100.2.4`
   - Click **Add**

### Step 9: Associate with Database Subnet

1. **Open Subnets:**
   - Click **Settings > Subnets**
   - Click **+ Associate**

2. **Configure:**
   - **Virtual network:** `Redwood-Database-VNet`
   - **Subnet:** `Protected-Subnet`
   - Click **OK**

### Validation

- ✅ Route table created: Redwood-Database-RT
- ✅ Default route added: 0.0.0.0/0 → 10.100.2.4
- ✅ Associated with Redwood-Database-VNet/Protected-Subnet

---

## Part D: Verify UDR Configuration

### Step 10: Verify All Route Tables

#### 10.1 Check Frontend Route Table

1. **Navigate to Redwood-Frontend-RT:**
   - Go to **Redwood-Frontend-RG > Redwood-Frontend-RT**
   - Click **Help > Effective routes**

2. **Verify Effective Routes:**
   - You should see routes including:
     - **0.0.0.0/0** → Next hop: 10.100.2.4 (Virtual appliance)
     - Local VNet routes (10.101.0.0/16)
     - Peering routes to hub

#### 10.2 Check Backend Route Table

1. **Navigate to Redwood-Backend-RT:**
   - Verify same configuration pattern
   - **0.0.0.0/0** → 10.100.2.4

#### 10.3 Check Database Route Table

1. **Navigate to Redwood-Database-RT:**
   - Verify same configuration pattern
   - **0.0.0.0/0** → 10.100.2.4

### Step 11: Verify Route Propagation to VMs

#### 11.1 Check Frontend VM Routes

1. **Navigate to Frontend VM 1:**
   - Go to **Redwood-Frontend-RG > Redwood-Frontend-VM**
   - Click **Networking > Network Settings > Network interface**
   - Click **Help > Effective routes**

2. **Review Routes:**
   - **Source: User** - This is your UDR
   - **Address prefix: 0.0.0.0/0**
   - **Next hop type: Virtual appliance**
   - **Next hop IP: 10.100.2.4**

   > [!NOTE]
   > This confirms the VM is using your UDR

#### 11.2 Repeat for Other VMs (optional)

Verify Backend VM and Database VM also show the UDR in their effective routes.

---

## Part E: Understand Traffic Impact

### Step 12: Test Current Connectivity (Expected Failures)

#### 12.1 Test Internet Connectivity

**From Frontend VM (via Serial Console or SSH from FortiGate):**

```bash
ping -c 3 8.8.8.8
```

**Expected Result:**

- ❌ **Fails** (timeout)
- **Why:** Traffic routed to FortiGate, but no policies allow it yet

#### 12.2 Test Inter-Spoke Connectivity

**From Frontend VM:**

```bash
ping -c 3 10.102.1.4  # Backend VM
ping -c 3 10.103.1.4  # Database VM
```

**Expected Result:**

- ❌ **Both fail**
- **Why:** Traffic forced through FortiGate, no policies configured

#### 12.3 Test Hub Connectivity

**From Frontend VM:**

```bash
# Try to reach FortiGate's internal interface
ping -c 3 10.100.2.4
```

**Expected Result:**

- ❌ **Likely fails**
- **Why:** ICMP may be blocked by FortiGate default policy

> [!WARNING]
> **Connectivity Broken By Design**
>
> After configuring UDRs, all spoke connectivity is broken. This is expected and correct behavior.
>
> **What's Happening:**
>
> 1. VMs send traffic to Internal LB (10.100.2.4)
> 2. ILB forwards to active FortiGate
> 3. FortiGate has no policies allowing traffic
> 4. Implicit deny blocks everything
>
> **Lab 4 Will Fix:**
>
> - Create firewall policies allowing traffic
> - Configure NAT for internet access
> - Enable logging and inspection
> - Restore full connectivity with security

---

## Understanding Routing Behavior

### Symmetric Routing in FortiGate HA

**Traffic Flow is Symmetric:**

With Active-Passive HA, all traffic flows through the same active FortiGate unit in both directions:

**Frontend to Backend (Request):**

```text
Frontend VM (10.101.1.4)
   ↓ UDR: 0.0.0.0/0 → 10.100.2.4
Internal LB (10.100.2.4)
   ↓ (always routes to Active FortiGate)
Active FortiGate port2 (receives)
   ↓ (inspect, apply policy)
Active FortiGate port2 (forwards)
   ↓ via Azure routing
Backend VM (10.102.1.4)
```

**Backend to Frontend (Response):**

```text
Backend VM (10.102.1.4)
   ↓ UDR: 0.0.0.0/0 → 10.100.2.4
Internal LB (10.100.2.4)
   ↓ (always routes to SAME Active FortiGate)
Active FortiGate port2 (receives)
   ↓ (inspect, apply policy)
Active FortiGate port2 (forwards)
   ↓ via Azure routing
Frontend VM (10.101.1.4)
```

**Key Point:** Traffic is symmetric - same FortiGate unit in both directions. The Internal Load Balancer always sends traffic to the active unit based on health probes. Only during HA failover (when active unit fails) does the traffic path change to the newly active unit.

### Route Table Hierarchy

**Azure evaluates routes in this order:**

1. **User Defined Routes (UDRs)** - Highest priority
2. **BGP Routes** - From VPN/ExpressRoute
3. **System Routes** - Azure's default routing

**In our case:**

- UDR (0.0.0.0/0 → 10.100.2.4) overrides everything
- Even more specific routes use our next-hop
- Only exception: local subnet routes (can't override)

### Why Default Route (0.0.0.0/0)?

**Alternative Specific Routes:**

You could create specific routes for each destination:

```text
10.102.0.0/16 → 10.100.2.4 (to Backend)
10.103.0.0/16 → 10.100.2.4 (to Database)
0.0.0.0/0     → 10.100.2.4 (to Internet)
```

**Why Default Route Better:**

- Single route catches everything
- Simpler management
- Scales as you add spoke VNets
- No risk of missing a destination

**Trade-off:**

- Less granular control
- All traffic forced through FortiGate (including local subnet)
- For local subnet, use /32 routes if needed

---

## Challenge! ⚔️

Frontend-VM-1 and Frontend-VM-2 are still able to communicate.
Why is that? Can you identify what configuration is missing?

---

## Lab 3 Complete! 🎉

### What You've Accomplished

You have successfully configured routing infrastructure:

✅ **Three Route Tables Created:**

- Redwood-Frontend-RT
- Redwood-Backend-RT
- Redwood-Database-RT

✅ **Default Routes Configured:**

- All three route tables point to 10.100.2.4
- Next-hop type: Virtual appliance

✅ **Subnet Associations:**

- Each route table associated with its spoke subnet
- Routes applied to all VMs in subnets

✅ **Traffic Forced Through FortiGate:**

- All spoke traffic now goes to Internal LB
- ILB distributes to active FortiGate
- Complete security inspection path established

✅ **Connectivity Broken (Expected):**

- Internet access: ❌
- Inter-spoke: ❌
- Intra-spoke ✅ - Solve the Challenge!
- Proves UDRs working correctly
- Lab 4 will restore with security policies

### Architecture Review

```text
Routing Configuration After Lab 3:

         Hub VNet (10.100.0.0/16)
      ┌────────────────────────────────┐
      │  Internal LB: 10.100.2.4       │◄─┐
      │  ↓                             │  │
      │  Active FortiGate              │  │
      │  (No policies yet)             │  │
      │  ↓                             │  │
      │  Implicit DENY all             │  │
      └────────────────────────────────┘  │
                                          │
      All traffic from spokes forced here │
                                          │
    ┌──────────────┬──────────────┬──────┴──┐
    │              │              │         │
    │ Frontend     │ Backend      │Database │
    │ RT:          │ RT:          │ RT:     │
    │ 0/0→10.100   │ 0/0→10.100   │0/0→10.  │
    │   .2.4       │   .2.4       │  100.2.4│
    └──────────────┴──────────────┴─────────┘

Result: All traffic blocked (no FortiGate policies)
Next: Lab 4 creates policies to allow traffic
```

### Key Takeaways

1. **UDRs Enable Security Inspection:**
   - Override Azure's direct routing
   - Force traffic through security appliances
   - Foundation for zero-trust architecture

2. **Default Route (0.0.0.0/0) Simplicity:**
   - Single route for all destinations
   - Easier to manage than specific routes
   - Scales as environment grows

3. **Internal Load Balancer Critical:**
   - Single next-hop (10.100.2.4) for all spokes
   - Forwards traffic to active FortiGate
   - Enables HA failover without route changes

4. **Temporary Connectivity Loss Expected:**
   - Proves routing working correctly
   - FortiGate seeing traffic but blocking
   - Lab 4 firewall policies will allow traffic

### Validation Checklist

Before proceeding to Lab 4, verify:

**Route Tables:**

- [ ] Redwood-Frontend-RT created and associated
- [ ] Redwood-Backend-RT created and associated
- [ ] Redwood-Database-RT created and associated

**Default Routes:**

- [ ] All three RTs have 0.0.0.0/0 → 10.100.2.4
- [ ] Next-hop type is "Virtual appliance"
- [ ] No typos in next-hop IP address

**Associations:**

- [ ] Frontend RT → Frontend VNet/Protected-Subnet
- [ ] Backend RT → Backend VNet/Protected-Subnet
- [ ] Database RT → Database VNet/Protected-Subnet

**Effective Routes:**

- [ ] VM effective routes show UDR (source: User)
- [ ] Next-hop IP is 10.100.2.4

**Expected Behavior:**

- [ ] Internet connectivity fails (expected)
- [ ] Inter-spoke connectivity fails (expected)
- [ ] Connectivity will be restored in Lab 4

### Next Steps

Ready for **Lab 4: Security Policies and Testing!**

In Lab 4, you will:

- Configure FortiGate firewall policies for north-south traffic
- Implement east-west traffic inspection between spokes
- Enable NAT for internet access
- Test end-to-end connectivity
- Use FortiView for traffic visualization
- Test HA failover scenario

**This is where everything comes together!**

---

## Troubleshooting Reference

### Issue: Route Not Showing in Effective Routes

**Checklist:**

1. Verify route table associated with subnet
2. Verify VM's NIC is in associated subnet
3. Refresh effective routes page (can take 30-60 seconds)
4. Check route table is in same region as VNet

### Issue: Wrong Next-Hop IP

**Problem:** Typo in next-hop address

**Solution:**

1. Edit route in route table
2. Change next-hop to exactly `10.100.2.4`
3. Save changes
4. Verify in VM effective routes

### Issue: Traffic Still Working (UDR Not Applied)

**Possible Causes:**

1. Route table not associated with subnet
2. More specific route overriding default

**Solutions:**

1. Verify association in route table → Subnets
2. Check for other routes in route table

### Issue: Can't Create Route Table

**Common Errors:**

- **Wrong region:** Must be Canada Central
- **Resource group locked:** Check for locks
- **Quota exceeded:** Check subscription limits
- **Name conflict:** Choose unique name

### Issue: VM Still Has Internet After UDR

**This would be unusual** - verify:

1. No Public IP assigned to VM
2. Route table actually associated
3. Default route configured correctly

### Issue: Unsure If UDRs Working

**Test Method:**

1. Check VM effective routes (should show UDR)
2. Try to ping internet - should fail
3. Check FortiGate logs - should see blocked traffic
4. Confirms traffic reaching FortiGate

---

## Advanced: Alternative Routing Designs

### Design 1: Specific Routes (Not Recommended Here)

Instead of 0.0.0.0/0, use specific routes:

```text
Redwood-Frontend-RT:
  10.102.0.0/16 → 10.100.2.4 (Backend)
  10.103.0.0/16 → 10.100.2.4 (Database)
  0.0.0.0/0     → 10.100.2.4 (Internet)
```

**Pros:**

- More granular control
- Can route some traffic differently

**Cons:**

- More routes to manage
- Must update when adding spoke VNets
- Higher chance of errors

### Design 2: /24 Host Routes

For intra-subnet inspection:

```text
10.101.1.0/24 → 10.100.2.4
```

**Pros:**

- Can inspect intra-subnet traffic

**Cons:**

- Complex management
- Route table limits (400 routes)

### Our Design: Default Route

```text
0.0.0.0/0 → 10.100.2.4
```

**Why This Works Best:**

- Simple (one route)
- Scalable (works for any number of spokes)
- Catches all traffic including internet
- Easy to troubleshoot

---

**End of Lab 3**  
*Estimated completion time: 30 minutes*  
*Next: Lab 4 - Security Policies and Testing*

---

*Lab Guide Version 1.0 - December 2024*  
*Questions? Ask your instructor or refer to the troubleshooting section.*
