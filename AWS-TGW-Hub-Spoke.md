# AWS Transit Gateway Hub-and-Spoke Network Architecture

## 📋 Project Overview

A hands-on demonstration of **multi-VPC enterprise networking** using AWS Transit Gateway (TGW) to create a secure, scalable hub-and-spoke topology. This project simulates a real-world scenario where multiple business units (Dev, Prod, Database) need controlled inter-VPC connectivity with security isolation.

### Project Architecture Diagram
```
                    ┌──────────────────────────┐
                    │   Transit Gateway (Hub)  │
                    │    enterprise-tgw        │
                    └──────────────┬───────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    │              │              │
        ┌───────────▼───────┐ ┌────▼────────┐ ┌──▼─────────────┐
        │   Dev VPC         │ │  Prod VPC   │ │  Database VPC  │
        │  10.2.0.0/16      │ │ 10.3.0.0/16 │ │  10.4.0.0/16   │
        │                   │ │             │ │                │
        │ ┌─────────────┐   │ │ ┌─────────┐ │ │ ┌────────────┐ │
        │ │ dev-instance│   │ │ │prod-inst│ │ │ │db-instance │ │
        │ │ 10.2.1.x    │   │ │ │10.3.1.x │ │ │ │10.4.1.x    │ │
        │ └─────────────┘   │ │ └─────────┘ │ │ └────────────┘ │
        └───────────────────┘ └─────────────┘ └────────────────┘
                │                   │                 │
            TGW-RT-Dev          TGW-RT-Prod       TGW-RT-DB
        (Propagates Prod)    (Propagates Dev+DB) (Propagates Prod)

Connectivity Rules:
  ✅ Dev → Prod (allowed)
  ✅ Prod → Dev (allowed)
  ✅ Prod → Database (allowed)
  ✅ Database → Prod (allowed)
  ❌ Dev → Database (BLOCKED)
  ❌ Database → Dev (BLOCKED)
```

---

## 🎯 Project Goals

| Goal | Outcome |
|------|---------|
| **Multi-VPC Connectivity** | Route traffic between 3 isolated VPCs using Transit Gateway hub |
| **Selective Routing** | Allow specific paths (Dev↔Prod, Prod↔DB) while blocking others (Dev⇸DB) |
| **Enterprise Simulation** | Replicate real organizational structure (Development, Production, Shared Services) |
| **Network Isolation** | Maintain security boundaries between sensitive environments |
| **Troubleshooting Skills** | Validate connectivity and diagnose blocked paths using Reachability Analyzer |

---

## 🏗️ Architecture Components

### 1. **Three VPCs (Spokes)**
| VPC | CIDR Block | Subnet | Purpose |
|-----|-----------|--------|---------|
| **dev-vpc** | 10.2.0.0/16 | 10.2.1.0/24 | Development environment |
| **prod-vpc** | 10.3.0.0/16 | 10.3.1.0/24 | Production applications |
| **db-vpc** | 10.4.0.0/16 | 10.4.1.0/24 | Shared database services |

### 2. **Transit Gateway (Hub)**
- **Name**: `enterprise-tgw`
- **ASN**: 64512 (default)
- **Route Table Strategy**: Separate route tables per attachment for granular control

### 3. **Transit Gateway Route Tables**
| Route Table | Associated With | Propagates From | Traffic Flow |
|-------------|-----------------|-----------------|--------------|
| **dev-tgw-rt** | Dev attachment | Prod attachment | Dev can reach Prod (10.3.0.0/16) |
| **prod-tgw-rt** | Prod attachment | Dev + DB attachments | Prod can reach both (10.2.0.0/16, 10.4.0.0/16) |
| **db-tgw-rt** | DB attachment | Prod attachment ONLY | DB can reach Prod only (10.3.0.0/16) |

### 4. **EC2 Instances (Test Nodes)**
| Instance | VPC | Private IP | Role |
|----------|-----|-----------|------|
| **dev-instance** | dev-vpc | 10.2.1.x | Source for Dev testing |
| **prod-instance** | prod-vpc | 10.3.1.x | Middleman/Hub instance |
| **db-instance** | db-vpc | 10.4.1.x | Destination for data access |

### 5. **Network Security**
- **Security Groups**: Instance-level access control (ICMP for ping, SSH for admin, port 3306 for MySQL simulation)
- **Route Tables**: VPC-level routing to Transit Gateway
- **Network ACLs**: Subnet-level stateless filtering (optional for this lab)

---

## 🔧 Implementation Details

### Phase 1: VPC Creation
- Created 3 independent VPCs with non-overlapping CIDR ranges (10.2.x, 10.3.x, 10.4.x)
- Each VPC has one public subnet for EC2 instance placement
- Each VPC has its own route table for local routing

### Phase 2: Transit Gateway Setup
- Deployed central Transit Gateway to act as the routing hub
- Configured to support DNS and ECMP (Equal Cost Multi-path routing)
- Default route table associations enabled for initial simplicity

### Phase 3: Attachment & Routing Configuration
- Attached all 3 VPCs to the Transit Gateway
- Created separate Transit Gateway route tables for each attachment
- Configured route propagation rules:
  - **Dev route table**: Propagates only Prod (10.3.0.0/16)
  - **Prod route table**: Propagates both Dev (10.2.0.0/16) and DB (10.4.0.0/16)
  - **DB route table**: Propagates only Prod (10.3.0.0/16)

### Phase 4: VPC Route Table Updates
- Added routes in each VPC route table pointing to the Transit Gateway:
  - Dev VPC route table: `10.3.0.0/16 → enterprise-tgw`
  - Prod VPC route table: `10.2.0.0/16 → enterprise-tgw`, `10.4.0.0/16 → enterprise-tgw`
  - DB VPC route table: `10.3.0.0/16 → enterprise-tgw`

### Phase 5: Testing Infrastructure
- EC2 instances in each VPC for connectivity validation
- Security groups configured to allow ICMP (ping) between VPCs
- Database security group restricts access to Prod only

---

## 📊 Expected Connectivity Results

### Test Matrix

| Source | Destination | Protocol | Expected | Result |
|--------|-------------|----------|----------|--------|
| dev-instance | prod-instance | ICMP (ping) | ✅ Reachable | Replies received |
| dev-instance | db-instance | ICMP (ping) | ❌ Unreachable | Timeout/No reply |
| prod-instance | dev-instance | ICMP (ping) | ✅ Reachable | Replies received |
| prod-instance | db-instance | ICMP (ping) | ✅ Reachable | Replies received |
| db-instance | prod-instance | ICMP (ping) | ✅ Reachable | Replies received |
| db-instance | dev-instance | ICMP (ping) | ❌ Unreachable | Timeout/No reply |
| prod-instance | db-instance | TCP 3306 (MySQL) | ✅ Reachable | Connection successful |
| dev-instance | db-instance | TCP 3306 (MySQL) | ❌ Unreachable | Connection timeout |

### Why Dev Cannot Reach Database
- **TGW Route Table Configuration**: `db-tgw-rt` does NOT propagate Dev attachment
- **Route Table Association**: Dev attachment uses `dev-tgw-rt`, which doesn't have 10.4.0.0/16 route
- **Result**: Traffic from Dev destined for DB subnet has no route and is dropped at TGW

---

## 🧪 Testing Procedures

### Test 1: Dev → Prod Connectivity
```bash
# SSH into dev-instance
# Ping prod-instance private IP
ping 10.3.1.x

# Expected: ICMP Echo Reply received
# Packets sent: 4, received: 4, 0% packet loss
```

### Test 2: Prod → Database Connectivity
```bash
# SSH into prod-instance
# Ping db-instance private IP
ping 10.4.1.x

# Expected: ICMP Echo Reply received
# Packets sent: 4, received: 4, 0% packet loss
```

### Test 3: Dev → Database Blockage (Expected to Fail)
```bash
# SSH into dev-instance
# Attempt to ping db-instance private IP
ping 10.4.1.x

# Expected: ICMP Timeout (no replies)
# Packets sent: 4, received: 0, 100% packet loss
```

### Test 4: Reachability Analyzer (Automated Validation)
**Dev → Database Path**
- **Source**: dev-instance
- **Destination**: db-instance
- **Expected Result**: "Not Reachable"
- **Reason**: TGW route table propagation rules block this path

**Prod → Database Path**
- **Source**: prod-instance
- **Destination**: db-instance
- **Expected Result**: "Reachable"
- **Reason**: Prod route table includes DB propagation

---

## 🔍 Troubleshooting Guide

| Issue | Symptoms | Root Cause | Solution |
|-------|----------|-----------|----------|
| **All pings fail** | No connectivity anywhere | TGW attachments not "Available" | Wait 30-60 seconds for attachments to activate |
| **Only some pings fail** | Dev↔Prod works, Dev→DB fails | Route propagation misconfigured | Verify TGW route table propagations match design |
| **Reachability shows "unknown"** | Analysis is incomplete | AWS analyzing paths | Wait 30-60 seconds for analysis to complete |
| **Can SSH but no ping** | TCP works, ICMP doesn't | Security group ICMP rule missing | Add ICMP rule: Type 8 (Echo), -1 (All codes) |
| **Transit Gateway Creation fails** | Error on TGW creation | IAM permissions insufficient | Ensure `ec2:CreateTransitGateway` permissions |
| **VPC attachment fails** | "Invalid VPC ID" error | Wrong subnet selected | Select subnet from correct VPC |

---

## 💡 Learning Outcomes

### Networking Concepts Mastered
- ✅ **Hub-and-Spoke Topology**: Centralized routing vs. full-mesh VPC peering
- ✅ **Transit Gateway Mechanics**: How TGW attachments and route tables work together
- ✅ **Route Propagation**: Dynamic route advertisement vs. static routing
- ✅ **Network Segmentation**: Using routing rules for security isolation
- ✅ **Multi-layer Security**: Route tables + TGW route tables + Security groups

### AWS Skills Developed
- ✅ Creating and configuring Transit Gateways
- ✅ Managing Transit Gateway attachments and route tables
- ✅ Designing network architectures for enterprise multi-VPC scenarios
- ✅ Using Reachability Analyzer for connectivity validation
- ✅ Implementing least-privilege network access

### Real-World Applications
| Scenario | Implementation |
|----------|-----------------|
| **Microservices Mesh** | Connect service VPCs to shared monitoring/logging VPC (DB-like isolation) |
| **Multi-Account Setup** | TGW supports cross-account attachments for centralized network hub |
| **Disaster Recovery** | Route prod-instance to standby prod-instance in different region via TGW |
| **Hybrid Cloud** | Extend pattern to on-premises via VPN attachment to same TGW |
| **Network Firewalling** | Add third-party NVA in dedicated VPC, route traffic through it |

---

## 🚀 Advanced Extensions

### Extension 1: Add NAT Gateway for Outbound Internet
- Deploy NAT Gateway in prod-vpc
- Add default route (0.0.0.0/0) pointing to NAT for dev/db instances
- Enable instances to reach AWS services/internet while maintaining isolation

### Extension 2: Enable Flow Logs
- Enable VPC Flow Logs in each VPC
- Capture rejected/accepted traffic to CloudWatch Logs
- Analyze why Dev↔DB traffic is blocked

### Extension 3: Add VPN Attachment
- Create Site-to-Site VPN attachment to TGW
- Simulate on-premises network connecting to enterprise hub
- Extend hub-and-spoke to hybrid architecture

### Extension 4: Transit Gateway Peering
- Create second TGW in different region
- Peer the two TGWs
- Enable cross-region multi-VPC connectivity

### Extension 5: Network Firewall Integration
- Deploy AWS Network Firewall in dedicated VPC
- Route cross-VPC traffic through firewall
- Implement Layer 7 (application) filtering

---

## 💰 Cost Considerations

| Component | Pricing | Free Tier? |
|-----------|---------|-----------|
| **Transit Gateway** | $0.05/hour (~$36/month) | ❌ No |
| **TGW Data Processing** | $0.02 per GB processed | ❌ No |
| **EC2 Instances (t3.micro)** | Varies by region | ✅ Yes (750 hrs/month) |
| **Data Transfer** | $0.02/GB (inter-region) | Partial |
| **VPC Flow Logs** | $0.50 per million records | ❌ No |

**Lab Cost Estimate**: ~$1-3 for a full day of testing (TGW + data processing)

**Cost Optimization Tips**:
- Delete TGW immediately after testing (most expensive component)
- Keep EC2 running in free tier only
- Use same-region VPCs (free data transfer between AZs)
- Disable Flow Logs unless specifically testing

---

## 🎓 Prerequisites

- ✅ AWS account with appropriate IAM permissions
- ✅ Understanding of VPC, subnets, route tables
- ✅ Familiarity with EC2 instance access (SSH)
- ✅ Basic networking knowledge (CIDR notation, routing)
- ✅ ~30 minutes to deploy, ~15 minutes to test

---

## 📚 Related AWS Services

| Service | Use Case |
|---------|----------|
| **AWS VPC Peering** | Point-to-point VPC connections (simpler, fewer VPCs) |
| **AWS PrivateLink** | Private connectivity to AWS services/third-party APIs |
| **VPN Connections** | Encrypt traffic between TGW and on-premises |
| **AWS Direct Connect** | Dedicated network connection to AWS (enterprise-grade) |
| **AWS Network Firewall** | Layer 7 inspection and filtering |
| **AWS CloudWatch Logs** | Centralized logging for Flow Logs |

---

## 🔐 Security Best Practices Applied

1. **Least Privilege Routing**: Only routes needed are propagated (Dev can't reach DB)
2. **Security Group Filtering**: Instance-level access restrictions (Database SG only allows Prod)
3. **Network Segmentation**: Separate subnets per VPC maintain blast radius
4. **Audit Trail**: Flow Logs (when enabled) show all traffic decisions
5. **No Internet Exposure**: Private IPs used for inter-VPC communication

---

## 📖 Deployment Checklist

- [ ] Create 3 VPCs (dev, prod, db) with correct CIDR blocks
- [ ] Create subnets in each VPC
- [ ] Create VPC route tables
- [ ] Create Internet Gateways (optional, for outbound internet)
- [ ] Attach IGWs to VPCs
- [ ] Create Transit Gateway
- [ ] Create TGW attachments for all 3 VPCs
- [ ] Create TGW route tables (dev-rtb, prod-rtb, db-rtb)
- [ ] Configure route propagation rules
- [ ] Associate attachments to route tables
- [ ] Update VPC route tables to point to TGW
- [ ] Create Security Groups for each VPC
- [ ] Launch EC2 instances in each VPC
- [ ] Test connectivity with ping
- [ ] Test with Reachability Analyzer
- [ ] Verify blockage (Dev → DB fails)
- [ ] Review Flow Logs (optional)
- [ ] Clean up resources

---

## 📝 Key Takeaways

| Concept | Insight |
|---------|---------|
| **Hub-and-Spoke Scale** | N attachments to hub vs. N(N-1)/2 peering connections for full mesh |
| **Centralized Management** | Single TGW as source of truth for multi-VPC routing |
| **Granular Control** | Route tables per attachment enable selective connectivity |
| **Security Through Routing** | Network isolation doesn't require expensive NVAs in every VPC |
| **Operational Simplicity** | TGW auto-scales; manual peering becomes unmanageable at scale |

---

## 🔗 References

- [AWS Transit Gateway Documentation](https://docs.aws.amazon.com/vpc/latest/tgw/)
- [VPC Routing Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Transit Gateway Route Tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Reachability Analyzer Guide](https://docs.aws.amazon.com/vpc/latest/userguide/reachability-analyzer.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

---

## 👨‍💻 Project Status

| Phase | Status | Details |
|-------|--------|---------|
| **Architecture Design** | ✅ Complete | Hub-and-spoke with selective routing |
| **Implementation** | ✅ Complete | 3 VPCs, TGW, route tables configured |
| **Testing** | ✅ Complete | Connectivity validated, blockage confirmed |
| **Documentation** | ✅ Complete | This guide + inline comments |


---

**Last Updated**: December 7, 2025  
**AWS Region**: us-east-1 (recommended) or your preference  
**Estimated Lab Duration**: 45 minutes (setup + testing)  
**Cleanup Required**: Yes - Transit Gateway charges apply
