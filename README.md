# AZ-102: FortiGate HA with Multi-VNet Hub-Spoke Architecture

## Fortinet Security Hands On Workshop | Azure Series

### Welcome

Building on the foundational skills from AZ-101, this advanced workshop addresses the challenges organizations face when scaling their Azure security infrastructure. As cloud deployments grow beyond a single VNet, maintaining consistent security policies, ensuring high availability, and managing complex traffic flows between multiple network segments becomes critical. Many organizations struggle with designing resilient security architectures, implementing proper network segmentation, and achieving both north-south (internet) and east-west (inter-VNet) traffic inspection without creating bottlenecks or single points of failure.

In this workshop, participants will deploy FortiGate in Active-Passive High Availability mode to protect a multi-VNet environment. Using Redwood Industries' expansion scenario, you'll build a hub-spoke topology with three spoke VNets (frontend, backend, database), configure vNet Peering and User Defined Routes for comprehensive traffic inspection, and establish complete visibility across the entire network fabric. This architecture represents real-world enterprise deployments and prepares you for production-scale implementations.

At the core of this solution is FortiGate HA with Azure Load Balancers (ELB/ILB), providing automatic failover and centralized security inspection. You'll learn how to leverage Azure's infrastructure for resilience while maintaining the security capabilities and operational consistency that FortiGate provides across hybrid environments.

### Time Requirements

The estimated time to complete this workshop is **3 hours**.

### Target Audience

- Cloud security architects designing multi-VNet environments
- Network security engineers implementing HA solutions
- Fortinet presales engineers and consultants
- IT professionals responsible for enterprise Azure deployments
- Partners expanding their FortiGate Azure practice
- Security operations teams managing hybrid infrastructure

**Experience Level:** Intermediate to Advanced professionals with AZ-101 completion or equivalent hands-on experience.

### What You'll Learn

- Design centralized security topology with FortiGate HA in the hub and multiple spoke VNets, including IP addressing strategy
- Configure Active-Passive high availability, failover mechanisms, and health probes
- Configure both External Load Balancer (for internet traffic) and Internal Load Balancer (for east-west inspection) with FortiGate
- Create route tables across VNets and implement user-defined routes to force traffic through the security hub
- Understand north-south (internet) and east-west (inter-VNet) traffic flows, including asymmetric routing
- Build security policies for different traffic types and configure outbound NAT
- Use FortiView, logs, and packet captures for traffic analysis and connectivity validation
- Test failure scenarios and demonstrate resilience for business value/ROI discussions

### Reference Architecture

After completing this workshop, you will have deployed the following architecture:

```text
┌─────────────────────────────────────────────────────────────────┐
│                      Azure Subscription                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │           Hub VNet (10.100.0.0/16) - Canada Central        │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │         FortiGate HA Cluster                         │  │ │
│  │  │  ┌──────────────┐      ┌──────────────┐             │  │ │
│  │  │  │  FGT-A       │◄────►│  FGT-B       │             │  │ │
│  │  │  │  (Active)    │ HA   │  (Passive)   │             │  │ │
│  │  │  └──────┬───────┘      └──────┬───────┘             │  │ │
│  │  │         │                     │                      │  │ │
│  │  │    ┌────▼─────────────────────▼────┐                │  │ │
│  │  │    │   External Load Balancer      │◄───Internet    │  │ │
│  │  │    │   (Public IP)                 │                │  │ │
│  │  │    └───────────────────────────────┘                │  │ │
│  │  │    ┌───────────────────────────────┐                │  │ │
│  │  │    │   Internal Load Balancer      │                │  │ │
│  │  │    │   (10.100.2.4)               │                │  │ │
│  │  │    └───────────────────────────────┘                │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│           │              │              │                        │
│      (Peering)      (Peering)      (Peering)                    │
│           │              │              │                        │
│  ┌────────▼─────┐ ┌─────▼──────┐ ┌─────▼──────┐                │
│  │  Frontend    │ │  Backend   │ │  Database  │                │
│  │  VNet        │ │  VNet      │ │  VNet      │                │
│  │ 10.101.0.0/16│ │10.102.0.0/16│ │10.103.0.0/16│              │
│  │              │ │            │ │            │                │
│  │ ┌──────────┐ │ │┌──────────┐│ │┌──────────┐│                │
│  │ │ Web VM   │ │ ││ App VM  ││ ││ DB VM   ││                │
│  │ └──────────┘ │ │└──────────┘│ │└──────────┘│                │
│  └──────────────┘ └────────────┘ └────────────┘                │
│                                                                  │
│  All traffic flows through FortiGate HA for inspection         │
│  UDRs force traffic to Internal Load Balancer (10.100.2.4)     │
└─────────────────────────────────────────────────────────────────┘
```

### Key Differences from AZ-101

| Aspect | AZ-101 | AZ-102 |
|--------|--------|--------|
| **Availability** | Single FortiGate VM | Active-Passive HA Cluster |
| **Load Balancing** | None | External + Internal LB |
| **Network Design** | Single VNet + Subnets | Hub-Spoke Multi-VNet |
| **Spoke Count** | 1 Protected Subnet | 3 Spoke VNets |
| **Traffic Patterns** | North-South only | North-South + East-West |
| **Complexity** | Beginner/Intermediate | Intermediate/Advanced |

### Business Value & ROI

**For Redwood Industries (Sample Customer):**

**Current Challenge:**

- Migrating critical applications to Azure
- Require 99.9% uptime SLA for FortiGate security
- Need network segmentation for compliance (PCI-DSS)
- Must inspect all traffic between application tiers

**AZ-102 Solution Benefits:**

1. **High Availability:**
   - Active-Passive HA eliminates single point of failure
   - Automatic failover in <10 seconds
   - Meets 99.9% uptime SLA requirement

2. **Scalability:**
   - Hub-spoke design supports multiple spoke VNets
   - Centralized security management reduces operational overhead
   - Easy to add new application tiers

3. **Security & Compliance:**
   - Complete visibility across all VNets
   - East-west traffic inspection for zero-trust architecture
   - Centralized logging for compliance auditing

4. **Cost Optimization:**
   - Single HA cluster protects all VNets (vs. firewall per VNet)
   - 60-70% cost savings vs. Azure Firewall at scale
   - Operational consistency = reduced training costs

### Workshop Structure

#### Lab 1: Hub VNet and FortiGate HA Deployment (45 min)
- Create hub VNet with proper subnet architecture
- Deploy FortiGate HA from Azure Marketplace
- Configure External and Internal Load Balancers
- Verify HA cluster formation and health probes
- **Break: 10 minutes**

#### Lab 2: Spoke VNets and VNet Peering (30 min)
- Create three spoke VNets (frontend, backend, database)
- Deploy test VMs in each spoke
- Establish VNet peering between hub and all spokes
- Test basic connectivity

#### Lab 3: User-Defined Routes and Traffic Steering (30 min)
- Create route tables for all spoke VNets
- Configure UDRs pointing to Internal LB
- Associate route tables with spoke subnets
- Understand traffic flow patterns
- **Break: 10 minutes**

#### Lab 4: Security Policies and End-to-End Testing (45 min)
- Configure FortiGate policies for north-south traffic
- Implement east-west traffic inspection
- Test internet access from all spokes
- Test inter-spoke communication
- Validate traffic flows with FortiView
- Test HA failover scenario
- **Break: 10 minutes**

### Laboratories

This workshop is organized in sequential laboratories, building on concepts from AZ-101.

**Lab 1** - [Hub VNet Infrastructure and FortiGate HA Deployment](labs/lab1-hub-fortigate-ha/README.md)  
**Lab 2** - [Spoke VNets Deployment and VNet Peering](labs/lab2-spokes-peering/README.md)  
**Lab 3** - [User-Defined Routes Configuration](labs/lab3-udrs-routing/README.md)  
**Lab 4** - [Security Policies and Testing](labs/lab4-policies-testing/README.md)

### Additional Resources

**Fortinet Documentation:**
- FortiGate Azure HA Guide: https://docs.fortinet.com/document/fortigate-public-cloud/7.6.0/azure-administration-guide/161167/deploying-fortigate-vm-ha-in-azure
- Hub-Spoke Architecture: https://docs.fortinet.com/document/fortigate-public-cloud/7.6.0/azure-administration-guide/849193/hub-and-spoke-topology

**Azure Documentation:**
- Hub-Spoke Network Topology: https://learn.microsoft.com/azure/architecture/reference-architectures/hybrid-networking/hub-spoke
- Azure Load Balancer: https://learn.microsoft.com/azure/load-balancer/
- User-Defined Routes: https://learn.microsoft.com/azure/virtual-network/virtual-networks-udr-overview

**Related Workshops:**
- AZ-101: FortiGate as Azure Cloud Firewall (prerequisite)

---

> [!NOTE]
> The workshop provides examples and sample code as instructional content. These examples help you understand FortiGate HA deployment and multi-VNet architecture. **Please note that these examples are not suitable for use in production environments without proper testing, security hardening, and customization for your specific requirements**.

---

> [!CAUTION]
> This lab uses several Azure resources including multiple VMs and load balancers. **Expected costs: $20-30 for the 3-hour workshop if resources are deleted immediately after**. At the end of the workshop, delete all resource groups to avoid ongoing charges. Stopping VMs is not sufficient - you must delete resources.

---

> [!WARNING]
> This lab deploys resources across multiple VNets with complex routing. If using a production subscription, use extreme caution with route table configuration to avoid impacting existing workloads. **We strongly recommend using an isolated test subscription for this workshop**.

---

## Getting Started

### Before You Begin

1. **Verify Prerequisites:**
   - Azure subscription with Contributor access
   - Completed AZ-101 or equivalent experience
   - 2 FortiFlex tokens available (request from instructor if needed)

2. **Review AZ-101 Concepts:**
   - Azure VNet and subnet fundamentals
   - FortiGate interface configuration
   - Basic firewall policy creation
   - NAT configuration for internet access

3. **Prepare Your Environment:**
   - Modern web browser (Chrome or Firefox recommended)
   - Stable internet connection
   - Notepad for recording IP addresses and passwords
   - Azure Portal access: https://portal.azure.com

### Workshop Flow

1. Start with **Lab 1** - do not skip ahead
2. Complete each lab's validation checklist before proceeding
3. Take scheduled breaks to avoid fatigue
4. Ask questions during the workshop
5. Delete all resources at the end to avoid charges

### Need Help?

- **During Workshop:** Raise your hand for instructor assistance
- **Post-Workshop:** Use Fortinet Community forums
- **Documentation Issues:** Submit GitHub issue to workshop repository

---

**Ready to begin?** Proceed to [Lab 1: Hub VNet Infrastructure and FortiGate HA Deployment](labs/lab1-hub-fortigate-ha/README.md)

---

*Workshop Version 1.0 - December 2024*  
*Estimated Total Time: 3 hours*  
*Next Workshop: AZ-103 - FortiGate Azure Virtual WAN Integration*