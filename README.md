# AWS VPC — Deep Dive Reference

> A complete reference covering every major VPC concept, with architecture diagrams, configuration details, and operational guidance. Split into 18 sections for easy navigation.

---

## Table of Contents

1. [Overview](#1-overview)
2. [Subnets](#2-subnets)
3. [Route Tables](#3-route-tables)
4. [Internet Gateway (IGW)](#4-internet-gateway-igw)
5. [NAT Gateway & NAT Instance](#5-nat-gateway--nat-instance)
6. [Security Groups](#6-security-groups)
7. [Network ACLs (NACLs)](#7-network-acls-nacls)
8. [VPC Endpoints](#8-vpc-endpoints)
9. [VPC Peering](#9-vpc-peering)
10. [AWS Transit Gateway (TGW)](#10-aws-transit-gateway-tgw)
11. [VPN Connections](#11-vpn-connections)
12. [AWS Direct Connect (DX)](#12-aws-direct-connect-dx)
13. [Elastic Network Interfaces (ENIs)](#13-elastic-network-interfaces-enis)
14. [Elastic IPs (EIPs)](#14-elastic-ips-eips)
15. [VPC Flow Logs](#15-vpc-flow-logs)
16. [DNS in VPC](#16-dns-in-vpc)
17. [High Availability & Multi-AZ Architecture](#17-high-availability--multi-az-architecture)
18. [CIDR Planning Cheat Sheet](#18-cidr-planning-cheat-sheet)

---

## 1. Overview

### What is a VPC?

An **Amazon Virtual Private Cloud (VPC)** is a logically isolated section of the AWS cloud where you launch AWS resources in a virtual network that you define. It gives you complete control over:

- IP address ranges (CIDR blocks)
- Subnet layout (public / private / isolated)
- Route tables and network gateways
- Security at both instance and subnet levels
- Connectivity to the internet, other VPCs, and on-premises networks

Think of a VPC as your own private data center network — but running on AWS infrastructure, without hardware to manage.

```
┌─────────────────────────────────────────────────────────────────────┐
│                         AWS Region (us-east-1)                      │
│                                                                     │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │                    Your VPC (10.0.0.0/16)                    │   │
│  │                                                              │   │
│  │   ┌──────────────────────┐  ┌──────────────────────────┐    │   │
│  │   │  AZ-A (us-east-1a)   │  │   AZ-B (us-east-1b)      │    │   │
│  │   │  ┌────────────────┐  │  │  ┌────────────────────┐   │    │   │
│  │   │  │ Public Subnet  │  │  │  │  Public Subnet     │   │    │   │
│  │   │  │  10.0.1.0/24   │  │  │  │   10.0.2.0/24      │   │    │   │
│  │   │  └────────────────┘  │  │  └────────────────────┘   │    │   │
│  │   │  ┌────────────────┐  │  │  ┌────────────────────┐   │    │   │
│  │   │  │ Private Subnet │  │  │  │  Private Subnet    │   │    │   │
│  │   │  │  10.0.11.0/24  │  │  │  │   10.0.12.0/24     │   │    │   │
│  │   │  └────────────────┘  │  │  └────────────────────┘   │    │   │
│  │   └──────────────────────┘  └──────────────────────────┘    │   │
│  └──────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────┘
```

### Default VPC vs Custom VPC

| Feature                        | Default VPC              | Custom VPC                          |
|-------------------------------|--------------------------|-------------------------------------|
| Created by                    | AWS automatically        | You                                 |
| CIDR block                    | 172.31.0.0/16 (fixed)   | Any RFC 1918 range you choose       |
| Subnets                       | One public per AZ        | You design the layout               |
| Internet Gateway              | Attached automatically   | You attach it                       |
| Route to internet             | Already in main route table | You add it                       |
| DNS hostnames                 | Enabled                  | Disabled by default                 |
| Best for                      | Quick experiments, demos | All production workloads            |

> **Recommendation:** Never use the Default VPC for production. Always create a custom VPC with a deliberate IP plan.

### IPv4 and IPv6 CIDR Rules

**IPv4:**
- VPC CIDR: `/16` to `/28` (AWS minimum is /28, maximum is /16)
- Cannot be changed after creation (you can *add* secondary CIDRs, not replace the primary)
- Allowed ranges: RFC 1918 private space (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
- You can also use public CIDRs you own (BYOIP)

**IPv6:**
- AWS assigns a `/56` block from Amazon's pool (or you can use BYOIP IPv6)
- Each subnet gets a `/64`
- All IPv6 addresses are globally unique (no private IPv6 in VPC)

---

## 2. Subnets

A **subnet** is a range of IP addresses within your VPC, confined to a single Availability Zone. Every resource you launch (EC2, RDS, Lambda in VPC, etc.) lives in a subnet.

### Types of Subnets

| Type      | Has route to IGW? | Has route to NAT GW? | Use case                            |
|-----------|-------------------|----------------------|-------------------------------------|
| Public    | Yes               | Not needed           | Load balancers, bastion hosts, NAT GW |
| Private   | No                | Yes (for egress)     | App servers, EKS nodes, caches       |
| Isolated  | No                | No                   | Databases, internal services with no egress |

### Subnet Layout Diagram

```
VPC: 10.0.0.0/16
│
├── AZ: us-east-1a
│   ├── Public Subnet:   10.0.1.0/24   (256 IPs → 251 usable)
│   ├── Private Subnet:  10.0.11.0/24
│   └── Isolated Subnet: 10.0.21.0/24
│
├── AZ: us-east-1b
│   ├── Public Subnet:   10.0.2.0/24
│   ├── Private Subnet:  10.0.12.0/24
│   └── Isolated Subnet: 10.0.22.0/24
│
└── AZ: us-east-1c
    ├── Public Subnet:   10.0.3.0/24
    ├── Private Subnet:  10.0.13.0/24
    └── Isolated Subnet: 10.0.23.0/24
```

### Reserved IP Addresses (5 per subnet)

For every subnet, AWS reserves the first 4 IPs and the last 1 IP. In a `/24` (256 total):

| Address       | Reserved for                         |
|---------------|--------------------------------------|
| 10.0.1.0      | Network address                      |
| 10.0.1.1      | VPC router                           |
| 10.0.1.2      | Amazon DNS                           |
| 10.0.1.3      | Reserved for future use              |
| 10.0.1.255    | Broadcast (not supported, reserved)  |

So a `/24` gives you **251 usable IPs**, a `/28` gives you only **11 usable IPs**.

### Key Subnet Settings

- **Auto-assign public IPv4**: When enabled, every instance launched in the subnet automatically gets a public IP (ephemeral — released when instance stops). Enable on public subnets.
- **Auto-assign IPv6**: Assigns an IPv6 address from the subnet's `/64` block.
- **Default subnet**: Behavior similar to subnets in the Default VPC.

---

## 3. Route Tables

A **route table** contains a set of rules (routes) that determine where network traffic is directed. Every subnet must be associated with exactly one route table.

### Main Route Table vs Custom Route Tables

- The **main route table** is created automatically with the VPC. All subnets not explicitly associated with a custom route table use it.
- **Custom route tables** let you control routing per subnet tier (public, private, isolated).
- Best practice: never add routes to the main route table — create separate tables per tier.

### The Local Route

Every route table always contains a **local route** that cannot be removed:

```
Destination     Target
10.0.0.0/16     local
```

This route enables all resources within the VPC to communicate with each other regardless of subnet.

### Common Route Table Configurations

**Public Subnet Route Table:**
```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         igw-xxxxxxxx     ← Internet Gateway
```

**Private Subnet Route Table (with NAT):**
```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         nat-xxxxxxxx     ← NAT Gateway (in public subnet)
```

**Isolated Subnet Route Table:**
```
Destination       Target
10.0.0.0/16       local
                  (nothing else)
```

**With VPC Peering:**
```
Destination       Target
10.0.0.0/16       local
10.1.0.0/16       pcx-xxxxxxxx     ← Peering connection to VPC-B
0.0.0.0/0         igw-xxxxxxxx
```

### Route Priority (Longest Prefix Match)

AWS always chooses the most specific route (longest prefix). A route to `10.0.1.0/24` takes priority over `10.0.0.0/16` for traffic destined to `10.0.1.5`.

### Subnet-to-Route-Table Association

```
                    ┌──────────────────────────────────────┐
                    │  Public RT                           │
 Public Subnet A ───┤  10.0.0.0/16 → local                │
 Public Subnet B ───┤  0.0.0.0/0   → igw-xxx              │
                    └──────────────────────────────────────┘

                    ┌──────────────────────────────────────┐
                    │  Private RT (AZ-A)                   │
 Private Subnet A ──┤  10.0.0.0/16 → local                │
                    │  0.0.0.0/0   → nat-gw-az-a          │
                    └──────────────────────────────────────┘

                    ┌──────────────────────────────────────┐
                    │  Private RT (AZ-B)                   │
 Private Subnet B ──┤  10.0.0.0/16 → local                │
                    │  0.0.0.0/0   → nat-gw-az-b          │
                    └──────────────────────────────────────┘
```

---

## 4. Internet Gateway (IGW)

An **Internet Gateway** is a horizontally scaled, redundant, highly available VPC component that allows communication between your VPC and the internet.

### Key Facts

- One IGW per VPC (1:1 relationship)
- Fully managed by AWS — no bandwidth bottleneck, no single point of failure
- Performs **NAT** for instances with public IPv4 addresses (translates between private and public IPs)
- For IPv6, IGW routes packets directly (no NAT needed — IPv6 is publicly routable)

### Requirements for an Instance to Reach the Internet

All three must be true:
1. Subnet has a route `0.0.0.0/0 → igw-xxx`
2. The instance has a public IPv4 (or EIP) assigned to its ENI
3. Security group allows the outbound traffic

### Traffic Flow — Public EC2 to Internet

```
   ┌─────────────────────────────────────────────────────────────┐
   │  VPC (10.0.0.0/16)                                          │
   │                                                             │
   │  Public Subnet (10.0.1.0/24)                               │
   │  ┌──────────────────────┐                                  │
   │  │  EC2 Instance        │                                  │
   │  │  Private: 10.0.1.10  │                                  │
   │  │  Public:  54.x.x.x   │                                  │
   │  └──────────┬───────────┘                                  │
   │             │ (packet: src=10.0.1.10, dst=8.8.8.8)         │
   │             ▼                                              │
   │  ┌──────────────────────┐                                  │
   │  │  Internet Gateway    │  ← translates src to 54.x.x.x   │
   │  └──────────┬───────────┘                                  │
   └─────────────┼───────────────────────────────────────────── ┘
                 │
                 ▼
            [ Internet ]
                 │
       (return traffic: dst=54.x.x.x)
                 │
                 ▼
   IGW translates dst back to 10.0.1.10
```

### Egress-Only Internet Gateway (for IPv6)

- Allows outbound IPv6 traffic from the VPC to the internet
- **Stateful**: allows return traffic automatically
- Prevents inbound IPv6 connections from the internet
- Use this on private subnets with IPv6 (analogous to NAT GW for IPv4)

---

## 5. NAT Gateway & NAT Instance

Private subnet instances need outbound internet access (for OS patches, API calls, etc.) without being directly reachable from the internet. **NAT** (Network Address Translation) solves this.

### NAT Gateway (Managed)

- Deployed into a **public subnet** with an **Elastic IP**
- AWS-managed: no patching, auto-scales up to 100 Gbps
- **Billing**: hourly charge (~$0.045/hr) + per-GB data processed (~$0.045/GB)
- Not free tier eligible
- **AZ-scoped**: one NAT GW per AZ for high availability

### NAT Instance (Self-Managed)

- An EC2 instance with the `amzn-ami-vpc-nat` AMI
- You must disable **source/destination check** on the instance
- Single point of failure unless you script failover
- Cheaper for low-traffic workloads; t3.nano can handle ~1 Gbps
- Full control: can run as a bastion, proxy, or VPN endpoint too

### NAT Gateway Traffic Flow

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  VPC (10.0.0.0/16)                                               │
   │                                                                  │
   │  Private Subnet (10.0.11.0/24)       Public Subnet (10.0.1.0/24)│
   │  ┌──────────────────┐               ┌──────────────────────────┐ │
   │  │  EC2 (App)       │               │  NAT Gateway             │ │
   │  │  10.0.11.25      │──────────────►│  Private: 10.0.1.5       │ │
   │  │                  │◄──────────────│  EIP:     52.x.x.x       │ │
   │  └──────────────────┘               └──────────┬───────────────┘ │
   │                                                │                 │
   └────────────────────────────────────────────────┼─────────────────┘
                                                    │
                                           ┌────────▼────────┐
                                           │ Internet Gateway │
                                           └────────┬────────┘
                                                    │
                                               [ Internet ]
```

### High Availability NAT Pattern

For true HA, deploy one NAT GW per AZ and give each AZ's private subnet its own route table pointing to its local NAT GW:

```
AZ-A Private Subnet → Private RT-A → NAT GW-A (in AZ-A public subnet)
AZ-B Private Subnet → Private RT-B → NAT GW-B (in AZ-B public subnet)
```

If AZ-A fails, AZ-B traffic is unaffected. If you use a single NAT GW and its AZ fails, all private subnets lose internet access.

### Comparison

| Feature               | NAT Gateway          | NAT Instance              |
|-----------------------|----------------------|---------------------------|
| Availability          | Managed by AWS       | You manage HA             |
| Bandwidth             | Up to 100 Gbps       | Limited by instance size  |
| Maintenance           | None                 | OS patching, monitoring   |
| Cost (low traffic)    | Higher (~$33/mo)     | Lower (t3.nano ~$4/mo)    |
| Security groups       | Not applicable       | Fully configurable        |
| Port forwarding       | No                   | Yes                       |
| Bastion host          | No                   | Yes (can combine roles)   |

---

## 6. Security Groups

A **Security Group (SG)** acts as a virtual stateful firewall for your EC2 instances and other resources (RDS, Lambda, ELB, etc.). It operates at the **ENI (instance) level**.

### Core Characteristics

- **Stateful**: if you allow inbound traffic on port 443, the return traffic is automatically allowed — no need for an explicit outbound rule.
- **Allow-only**: you can only write allow rules. There is no explicit deny. All traffic not matched by an allow rule is **implicitly denied**.
- **Multiple SGs**: an instance can have up to 5 security groups (soft limit, adjustable).
- **Applied to ENIs**: technically SGs attach to network interfaces, not instances.

### Default Security Group

Every VPC has a default SG. Its initial rules:
- Inbound: allow all traffic from other resources also using the default SG
- Outbound: allow all traffic to anywhere

> **Best practice**: never use the default SG. Create purpose-specific SGs.

### Security Group Chaining (Reference by SG ID)

Instead of specifying IP CIDR ranges, you can reference another SG as the source/destination. This is far more maintainable — as instances scale out or IP addresses change, the rule stays correct.

```
┌──────────────────────────────────────────────────────────────────┐
│  SG: sg-alb (Application Load Balancer)                          │
│  Inbound:  0.0.0.0/0 on port 443                                 │
│  Outbound: sg-app on port 8080                                   │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  SG: sg-app (Application Servers)                                │
│  Inbound:  sg-alb on port 8080                                   │
│  Outbound: sg-db on port 5432                                    │
└────────────────────────────┬─────────────────────────────────────┘
                             │
                             ▼
┌──────────────────────────────────────────────────────────────────┐
│  SG: sg-db (Database Servers)                                    │
│  Inbound:  sg-app on port 5432                                   │
│  Outbound: (none needed — stateful return handled automatically) │
└──────────────────────────────────────────────────────────────────┘
```

### Example Security Group Rules

**ALB Security Group:**
| Direction | Protocol | Port | Source        |
|-----------|----------|------|---------------|
| Inbound   | TCP      | 80   | 0.0.0.0/0     |
| Inbound   | TCP      | 443  | 0.0.0.0/0     |
| Outbound  | All      | All  | 0.0.0.0/0     |

**App Server Security Group:**
| Direction | Protocol | Port | Source/Dest   |
|-----------|----------|------|---------------|
| Inbound   | TCP      | 8080 | sg-alb        |
| Inbound   | TCP      | 22   | sg-bastion    |
| Outbound  | All      | All  | 0.0.0.0/0     |

**Database Security Group:**
| Direction | Protocol | Port | Source        |
|-----------|----------|------|---------------|
| Inbound   | TCP      | 5432 | sg-app        |
| Outbound  | All      | All  | 0.0.0.0/0     |

---

## 7. Network ACLs (NACLs)

A **Network Access Control List (NACL)** is a stateless firewall applied at the **subnet boundary**. It controls traffic entering and leaving the subnet.

### Core Characteristics

- **Stateless**: return traffic must be explicitly allowed. If you allow inbound TCP/443, you must also allow outbound TCP on the ephemeral port range (1024–65535).
- **Numbered rules**: rules are evaluated lowest number first. First match wins.
- **Allow and Deny**: unlike SGs, NACLs support explicit deny rules.
- **One NACL per subnet**: each subnet is associated with exactly one NACL.
- **Default NACL**: allows all inbound and outbound traffic.

### Ephemeral Ports

When a client initiates a TCP connection, the OS picks a random **ephemeral (source) port** in the range:
- Linux: `32768–60999`
- Windows: `49152–65535`
- AWS recommends covering `1024–65535` to be safe

Your NACL outbound rules must allow this range to permit return traffic for inbound connections.

### Example NACL (Public Subnet)

**Inbound Rules:**
| Rule # | Type       | Protocol | Port Range  | Source       | Allow/Deny |
|--------|------------|----------|-------------|--------------|------------|
| 100    | HTTPS      | TCP      | 443         | 0.0.0.0/0    | ALLOW      |
| 110    | HTTP       | TCP      | 80          | 0.0.0.0/0    | ALLOW      |
| 120    | Custom TCP | TCP      | 1024-65535  | 0.0.0.0/0    | ALLOW      |
| 130    | SSH        | TCP      | 22          | 10.0.0.0/8   | ALLOW      |
| *      | All        | All      | All         | 0.0.0.0/0    | DENY       |

**Outbound Rules:**
| Rule # | Type       | Protocol | Port Range  | Destination  | Allow/Deny |
|--------|------------|----------|-------------|--------------|------------|
| 100    | HTTPS      | TCP      | 443         | 0.0.0.0/0    | ALLOW      |
| 110    | HTTP       | TCP      | 80          | 0.0.0.0/0    | ALLOW      |
| 120    | Custom TCP | TCP      | 1024-65535  | 0.0.0.0/0    | ALLOW      |
| *      | All        | All      | All         | 0.0.0.0/0    | DENY       |

### Security Groups vs NACLs — Side-by-Side

| Feature               | Security Group               | NACL                            |
|-----------------------|------------------------------|---------------------------------|
| Level                 | Instance (ENI)               | Subnet                          |
| State                 | Stateful                     | Stateless                       |
| Rules                 | Allow only                   | Allow + Deny                    |
| Rule evaluation       | All rules evaluated          | Lowest rule number first        |
| Return traffic        | Automatically allowed        | Must be explicitly allowed      |
| Scope                 | Per resource                 | Per subnet                      |
| Default behavior      | Deny all inbound             | Allow all (default NACL)        |
| Use case              | Micro-segmentation           | Subnet-level broad blocking     |

> **Best practice**: Use Security Groups as your primary control layer. Use NACLs only for broad subnet-level blocks (e.g., blocking a specific IP range across the entire subnet tier).

---

## 8. VPC Endpoints

A **VPC Endpoint** lets resources inside your VPC privately connect to supported AWS services **without** leaving the Amazon network — no Internet Gateway, NAT device, VPN, or Direct Connect required.

### Gateway Endpoints

- **Services**: Amazon S3 and Amazon DynamoDB only
- **Free** — no hourly charge
- Works via a route table entry pointing to the endpoint
- Traffic never leaves the AWS network
- Endpoint policies can restrict which S3 buckets or DynamoDB tables are accessible

```
Route Table Entry Added Automatically:
Destination         Target
pl-xxxxxxxx (S3)    vpce-xxxxxxxx
```

```
Private Subnet EC2 ──── [VPC local route] ──── S3 Gateway Endpoint ──── S3
                             (no NAT, no IGW)
```

### Interface Endpoints (AWS PrivateLink)

- **Services**: 100+ AWS services (EC2 API, SSM, Secrets Manager, KMS, SNS, SQS, Kinesis, CloudWatch, and many more)
- Creates an **ENI with a private IP** in your subnet
- **Billed**: ~$0.01/hr per AZ + data processing per GB
- Supports endpoint policies
- Supports **Private DNS**: when enabled, the service's standard DNS name (e.g., `ec2.us-east-1.amazonaws.com`) resolves to the private IP — no code changes needed

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  VPC (10.0.0.0/16)                                              │
   │                                                                 │
   │  Private Subnet                                                 │
   │  ┌──────────────────────┐                                      │
   │  │  EC2 / Lambda / ECS  │                                      │
   │  └──────────┬───────────┘                                      │
   │             │                                                   │
   │             ▼                                                   │
   │  ┌──────────────────────────────────────────────────────────┐  │
   │  │  Interface Endpoint (ENI: 10.0.11.50)                    │  │
   │  │  DNS: ssm.us-east-1.amazonaws.com → 10.0.11.50           │  │
   │  └──────────────────────────────────────────────────────────┘  │
   │             │                                                   │
   └─────────────┼───────────────────────────────────────────────── ┘
                 │ (stays on AWS private backbone)
                 ▼
          [ AWS SSM Service ]
          (never touches internet)
```

### When to Use Which

| Situation                              | Use                          |
|----------------------------------------|------------------------------|
| Private access to S3 or DynamoDB       | Gateway Endpoint (free)      |
| Private access to any other AWS service| Interface Endpoint           |
| Share your own service with other VPCs | PrivateLink (Endpoint Service) |
| Reduce NAT GW data costs for S3        | Gateway Endpoint             |

---

## 9. VPC Peering

**VPC Peering** creates a private, one-to-one networking connection between two VPCs. Traffic stays on the AWS backbone and never traverses the public internet.

### Key Characteristics

- **Non-transitive**: if VPC-A is peered with VPC-B, and VPC-B is peered with VPC-C, VPC-A cannot talk to VPC-C through VPC-B. You'd need a direct A↔C peering.
- **No overlapping CIDRs**: the CIDR blocks of the two VPCs cannot overlap.
- Works within a region, across regions, and across AWS accounts.
- After creating the peering connection, you must add routes on **both sides**.

### Transitive Routing Limitation

```
VPC-A (10.0.0.0/16)  ←──pcx-AB──→  VPC-B (10.1.0.0/16)
                                           │
                                        pcx-BC
                                           │
                                    VPC-C (10.2.0.0/16)

VPC-A CANNOT reach VPC-C via VPC-B.
Traffic from A destined for 10.2.x.x arrives at B
but B will NOT forward it to C — peering is not transitive.
```

### Route Table Changes Required

**VPC-A Route Table (to reach VPC-B):**
```
10.1.0.0/16    pcx-AB
```

**VPC-B Route Table (to reach VPC-A):**
```
10.0.0.0/16    pcx-AB
```

### Full Mesh vs Transit Gateway

For N VPCs, full-mesh peering requires `N*(N-1)/2` connections:

| VPCs | Peering connections needed |
|------|---------------------------|
| 2    | 1                         |
| 5    | 10                        |
| 10   | 45                        |
| 20   | 190                       |

At scale, use **Transit Gateway** instead (Section 10).

### DNS Resolution Across Peering

By default, DNS hostnames of peered VPC instances resolve to their public IP (if any). To resolve to private IPs across the peering:
- Enable **DNS resolution from accepter VPC** on the peering connection
- Both sides must have `enableDnsSupport = true`

---

## 10. AWS Transit Gateway (TGW)

**AWS Transit Gateway** is a regional hub that interconnects VPCs, VPNs, and Direct Connect in a hub-and-spoke model. It eliminates the full-mesh complexity of VPC peering at scale.

### Architecture

```
                          ┌──────────────────────┐
             ┌────────────│   Transit Gateway     │────────────┐
             │            │   (TGW-hub)           │            │
             │            └──────────┬────────────┘            │
             │                       │                         │
        ┌────┴────┐             ┌────┴────┐              ┌─────┴────┐
        │  VPC-A  │             │  VPC-B  │              │  VPC-C   │
        │(10.0/16)│             │(10.1/16)│              │(10.2/16) │
        └─────────┘             └─────────┘              └──────────┘
             │                                                 │
        ┌────┴────┐                                    ┌───────┴──────┐
        │  VPN    │                                    │ Direct       │
        │ (On-prem│                                    │ Connect      │
        │  DC)    │                                    │ (On-prem DC) │
        └─────────┘                                    └──────────────┘
```

### TGW Concepts

**Attachment**: a connection from a VPC, VPN, DX, or another TGW to the Transit Gateway.

**TGW Route Table**: the routing brain of TGW. A TGW can have multiple route tables, allowing you to segment traffic (e.g., prod VPCs cannot route to dev VPCs).

**Association**: each attachment is associated with one TGW route table (determines which routes the attachment can use).

**Propagation**: an attachment can automatically propagate its CIDR into a TGW route table.

### Example: Isolated VPC Routing

```
TGW Route Table: "Prod"
  10.0.0.0/16 → vpc-A attachment
  10.1.0.0/16 → vpc-B attachment

TGW Route Table: "Dev"
  10.2.0.0/16 → vpc-C attachment

VPC-A and VPC-B are associated with "Prod" → they can reach each other.
VPC-C is associated with "Dev" → isolated from Prod VPCs.
```

### Multi-Account with Resource Access Manager (RAM)

Share the TGW to other AWS accounts via AWS RAM. Each account attaches their VPCs to the shared TGW without needing peering connections.

### Inter-Region Peering

TGWs in different regions can be peered together. Traffic between regions flows over the AWS global backbone (not the public internet). Static routes must be added — BGP propagation is not supported across inter-region TGW peering.

### Bandwidth and Limits

| Metric                        | Default Limit         |
|-------------------------------|-----------------------|
| Max attachments per TGW       | 5,000                 |
| Max route tables per TGW      | 20                    |
| Max routes per route table    | 10,000                |
| Bandwidth per VPC attachment  | Up to 50 Gbps         |
| VPN attachment bandwidth      | 1.25 Gbps per tunnel  |

---

## 11. VPN Connections

AWS provides two types of VPN for extending your network into a VPC.

### Site-to-Site VPN

Connects your **on-premises network or datacenter** to your VPC over IPsec tunnels through the internet.

**Components:**
- **Virtual Private Gateway (VGW)**: the AWS side of the VPN, attached to your VPC
- **Customer Gateway (CGW)**: represents your on-premises VPN device in AWS (stores its public IP and BGP ASN)
- **VPN Connection**: the logical connection between VGW and CGW, consisting of **2 IPsec tunnels** (for redundancy — each terminates in a different AWS AZ)

```
On-Premises Datacenter
┌─────────────────────────────────┐
│  Your Router / Firewall         │
│  (Customer Gateway: 203.x.x.x) │
└──────────────┬──────────────────┘
               │
      ┌────────┴────────┐
      │  Tunnel 1        │  ← IPsec/IKE over internet
      │  Tunnel 2        │  ← IPsec/IKE over internet (different AWS AZ)
      └────────┬────────┘
               │
┌──────────────┴──────────────────────────────────────────────┐
│  VPC                                                        │
│  ┌────────────────────────────┐                             │
│  │  Virtual Private Gateway   │                             │
│  │  (VGW — attached to VPC)  │                             │
│  └────────────────────────────┘                             │
└─────────────────────────────────────────────────────────────┘
```

**Routing options:**
- **Static routing**: you manually enter your on-premises CIDRs in the VPN connection
- **BGP (dynamic routing)**: VGW and CGW exchange routes automatically (recommended)

**Bandwidth:** ~1.25 Gbps max per tunnel (2.5 Gbps total with ECMP).

### AWS Client VPN

Allows **individual users** to securely access AWS resources and on-premises networks using an OpenVPN-based client.

- Users authenticate via Active Directory, SAML 2.0 IdP, or certificate-based auth
- Assigns each user a private IP from the client CIDR pool
- Subnet associations determine which VPC subnets clients can access
- Split-tunnel mode: only VPC-destined traffic goes through the VPN; internet traffic goes direct

```
User Laptop
(OpenVPN client)
      │
      │  TLS/OpenVPN over internet
      ▼
Client VPN Endpoint (ENIs in your VPC subnets)
      │
      ▼
VPC Resources (EC2, RDS, EKS…)
```

---

## 12. AWS Direct Connect (DX)

**AWS Direct Connect** provides a dedicated, private network connection from your datacenter to AWS — bypassing the public internet entirely for lower latency, consistent bandwidth, and reduced data transfer costs.

### Connection Types

| Type       | Speed Options               | Notes                              |
|------------|-----------------------------|------------------------------------|
| Dedicated  | 1 Gbps, 10 Gbps, 100 Gbps  | Physical port in a DX location     |
| Hosted     | 50 Mbps – 10 Gbps          | Provisioned by an AWS DX partner   |

### Physical Path

```
Your Datacenter
      │
      │  Your network (fiber)
      ▼
DX Location (colocation facility)
      │  Cross-connect to AWS cage
      ▼
AWS Direct Connect Router
      │  AWS private backbone
      ▼
AWS Region
```

### Virtual Interfaces (VIFs)

A VIF is a virtual layer-3 connection layered on top of the physical DX port.

| VIF Type     | Connects to        | Use case                                    |
|--------------|--------------------|---------------------------------------------|
| Private VIF  | Virtual Private GW | Access resources in a single VPC            |
| Public VIF   | AWS public network | Access AWS public services (S3, DynamoDB)   |
| Transit VIF  | Transit Gateway    | Access multiple VPCs via a single DX + TGW  |

### Direct Connect Gateway (DXGW)

A **DX Gateway** lets a single DX connection reach VPCs in multiple AWS Regions and multiple accounts — without needing a DX connection per region.

```
On-Prem Datacenter
      │
      │ DX Connection (10 Gbps)
      ▼
DX Location
      │
      ▼
Direct Connect Gateway
      ├──── Private VIF ──── VPC (us-east-1)
      ├──── Private VIF ──── VPC (eu-west-1)
      └──── Transit VIF ──── TGW (ap-southeast-1)
                              ├── VPC-A
                              └── VPC-B
```

### Encryption

DX is **not encrypted** by default. Options to add encryption:
1. **DX + Site-to-Site VPN**: run IPsec VPN over the DX connection (hardware VPN via VGW)
2. **MACsec**: Layer-2 encryption available on Dedicated 10G/100G connections

### Resilience Best Practices

| Level     | Setup                                                   |
|-----------|---------------------------------------------------------|
| High      | Two DX connections to two different DX locations        |
| Maximum   | Above + a Site-to-Site VPN as backup failover path      |
| Dev/Test  | Single DX + VPN backup                                  |

---

## 13. Elastic Network Interfaces (ENIs)

An **Elastic Network Interface (ENI)** is a logical networking component in a VPC that represents a virtual network card. Every EC2 instance has at least one ENI (the primary, `eth0`).

### ENI Attributes

Each ENI can have:
- One primary private IPv4 address
- One or more secondary private IPv4 addresses
- One Elastic IP per private IPv4
- One public IPv4 (ephemeral)
- One or more IPv6 addresses
- Up to 5 security groups
- A MAC address
- A source/destination check flag

### Primary vs Secondary ENIs

| Property               | Primary ENI (eth0)         | Secondary ENI (eth1…)        |
|------------------------|----------------------------|------------------------------|
| Deletable              | No (deleted with instance) | Yes                          |
| Detachable             | No                         | Yes (can move to another instance) |
| Created by             | AWS at launch              | You (manually or at launch)  |

### Dual-Homed Instance

Attach a secondary ENI from a different subnet to create an instance with interfaces in two subnets (e.g., a network appliance, firewall, or proxy):

```
┌───────────────────────────────────────────────────┐
│  EC2 Instance (e.g., firewall appliance)          │
│                                                   │
│  eth0 (10.0.1.10) ─── Public Subnet              │
│  eth1 (10.0.11.10) ── Private Subnet             │
└───────────────────────────────────────────────────┘
```

### ENI Failover Pattern

For fast failover without re-IP: pre-attach a secondary ENI to a standby instance. On failure, detach from primary and re-attach to standby. The Elastic IP follows the ENI — clients see no IP change.

---

## 14. Elastic IPs (EIPs)

An **Elastic IP (EIP)** is a static public IPv4 address allocated to your AWS account that you can associate with any EC2 instance or NAT Gateway in your VPC.

### Key Facts

- Allocated from AWS's public IPv4 pool (or from your BYOIP range)
- Persists independently of instances — survives stop/start cycles
- Moves between instances instantly (important for failover)
- **Charges**: free while associated with a running instance; **charged** (~$0.005/hr) when allocated but not associated (to discourage hoarding)
- Soft limit: 5 EIPs per region per account (requestable increase)

### Typical Use Cases

| Use Case                            | Why EIP                                     |
|-------------------------------------|---------------------------------------------|
| NAT Gateway                         | Required — must have a static public IP     |
| Bastion host                        | Stable IP to allowlist in firewalls         |
| Application failover                | Move EIP to standby instance in seconds     |
| DNS A record pointing to EC2        | Stable IP so DNS doesn't need updating      |

### EIP vs Auto-Assigned Public IP

| Feature                | Auto-assigned Public IP | Elastic IP           |
|------------------------|-------------------------|----------------------|
| Static                 | No (changes on stop)    | Yes                  |
| Survives stop/start    | No                      | Yes                  |
| Transferable           | No                      | Yes                  |
| Cost when associated   | Free                    | Free                 |
| Cost when unassociated | N/A                     | ~$0.005/hr           |

---

## 15. VPC Flow Logs

**VPC Flow Logs** capture metadata about IP traffic flowing through network interfaces in your VPC. Useful for security analysis, compliance, and troubleshooting connectivity issues.

### What Gets Captured

Flow logs record **accepted and rejected** traffic at the network level. Each log record is a flow (not a packet — it's an aggregation window of ~1–10 minutes).

**Default flow log record fields:**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action log-status
```

**Example record (ACCEPT):**
```
2 123456789012 eni-abc12345 10.0.1.25 10.0.11.50 45892 5432 6 20 4000 1620000000 1620000060 ACCEPT OK
```

**Example record (REJECT — NACL or SG blocked):**
```
2 123456789012 eni-abc12345 203.0.113.5 10.0.1.25 54321 22 6 3 180 1620000000 1620000060 REJECT OK
```

### Capture Levels

| Level     | Captures traffic for                          |
|-----------|-----------------------------------------------|
| VPC       | All ENIs in the VPC                           |
| Subnet    | All ENIs in a specific subnet                 |
| ENI       | A single network interface                    |

### Destinations

| Destination              | Best for                                       |
|--------------------------|------------------------------------------------|
| CloudWatch Logs          | Real-time monitoring, metric filters, alarms   |
| Amazon S3                | Long-term storage, Athena queries, cost-efficient |
| Kinesis Data Firehose    | Real-time streaming to S3/OpenSearch/Splunk    |

### Custom Flow Log Format

You can choose which fields to include. Adding newer fields like `vpc-id`, `subnet-id`, `instance-id`, `tcp-flags`, `pkt-srcaddr`, `pkt-dstaddr` helps with advanced analysis.

### Cost Considerations

- Ingestion cost to CloudWatch Logs: ~$0.50/GB
- S3 storage: ~$0.023/GB/month
- For high-traffic VPCs, use S3 + Athena for cost-effective querying
- Consider filtering: capture only `REJECT` flows for security alerting

---

## 16. DNS in VPC

Every VPC gets a built-in DNS resolver provided by AWS — the **Amazon Route 53 Resolver** (accessible at the VPC base CIDR +2 address, e.g., `10.0.0.2`).

### VPC DNS Settings

Two critical settings on every VPC:

| Setting                | Description                                              | Default in custom VPC |
|------------------------|----------------------------------------------------------|-----------------------|
| `enableDnsSupport`     | Enables the Route 53 Resolver for the VPC                | true                  |
| `enableDnsHostnames`   | Assigns DNS hostnames to EC2 instances with public IPs   | false                 |

> Both must be `true` for EC2 instances to get public DNS hostnames like `ec2-54-x-x-x.compute-1.amazonaws.com`.

### Private Hosted Zones

You can attach a **Route 53 Private Hosted Zone** (PHZ) to your VPC. Resources in the VPC resolve names from the PHZ:

```
EC2 queries: db.internal.mycompany.com
  └── Route 53 Resolver (10.0.0.2) checks PHZ
  └── Returns: 10.0.21.15 (RDS instance private IP)
```

PHZs can be associated with multiple VPCs (including cross-account).

### Hybrid DNS with Route 53 Resolver Endpoints

When you have a hybrid environment, you need DNS to resolve:
- AWS resources from on-premises
- On-premises hostnames from within VPC

#### Inbound Endpoint
An ENI in your VPC that accepts DNS queries **from on-premises**:

```
On-prem DNS server
      │ query: app.internal.aws (forward rule)
      ▼
Inbound Resolver Endpoint (ENI: 10.0.11.53)
      │
      ▼
Route 53 Resolver → returns private IP of app.internal.aws
```

#### Outbound Endpoint
An ENI in your VPC that forwards DNS queries **to on-premises DNS**:

```
EC2 Instance
      │ query: server.corp.internal
      ▼
Route 53 Resolver → checks forwarding rules
      │ matches: *.corp.internal → forward to 192.168.1.53
      ▼
Outbound Resolver Endpoint (ENI: 10.0.11.54)
      │
      ▼
On-premises DNS (192.168.1.53)
      │
      ▼
Returns IP of server.corp.internal
```

### Full Hybrid DNS Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│  AWS VPC                                                                 │
│                                                                          │
│  EC2: "resolve server.corp.internal"                                     │
│    │                                                                     │
│    ▼                                                                     │
│  Route 53 Resolver (10.0.0.2)                                            │
│    │  checks Resolver Rules                                              │
│    │  *.corp.internal → forward to 192.168.1.53 via Outbound Endpoint   │
│    ▼                                                                     │
│  Outbound Endpoint ENI ──────────────────────────────────────────────┐  │
└──────────────────────────────────────────────────────────────────────┼──┘
                         VPN / Direct Connect                          │
                                                                       ▼
                                                         On-Premises DNS
                                                         (192.168.1.53)
                                                                       │
                                                         Resolves and returns IP
```

---

## 17. High Availability & Multi-AZ Architecture

This section brings everything together into a production-ready, highly available VPC architecture — the canonical 3-tier pattern.

### Architecture Goals

- No single point of failure at the network layer
- Each tier isolated in its own subnet type
- AZ-independent: failure of one AZ doesn't take down the application
- Minimal blast radius per security boundary

### Complete Production VPC — ASCII Diagram

```
  ┌─────────────────────────────────────────────────────────────────────────────────┐
  │  AWS Region (e.g. us-east-1)                                                    │
  │  VPC: 10.0.0.0/16                                                               │
  │                                                                                 │
  │  ┌──────────────────────────────────────────────────────────────────────────┐   │
  │  │                       Internet Gateway (IGW)                             │   │
  │  └────────────────────────────────┬─────────────────────────────────────────┘   │
  │                                   │                                             │
  │  ┌──────────────────────────────┐ │ ┌──────────────────────────────────────┐    │
  │  │   AZ: us-east-1a             │ │ │   AZ: us-east-1b                     │    │
  │  │                              │ │ │                                      │    │
  │  │  ┌────────────────────────┐  │ │ │  ┌─────────────────────────────┐    │    │
  │  │  │  PUBLIC (10.0.1.0/24)  │◄─┘ └─►│  PUBLIC (10.0.2.0/24)       │    │    │
  │  │  │  ┌──────────────────┐  │       │  ┌──────────────────────┐    │    │    │
  │  │  │  │ ALB Node (AZ-A)  │  │       │  │ ALB Node (AZ-B)      │    │    │    │
  │  │  │  └──────────────────┘  │       │  └──────────────────────┘    │    │    │
  │  │  │  ┌──────────────────┐  │       │  ┌──────────────────────┐    │    │    │
  │  │  │  │  NAT Gateway A   │  │       │  │  NAT Gateway B       │    │    │    │
  │  │  │  │  EIP: 52.x.x.x   │  │       │  │  EIP: 54.x.x.x       │    │    │    │
  │  │  │  └──────────────────┘  │       │  └──────────────────────┘    │    │    │
  │  │  └────────────────────────┘       └─────────────────────────────┘    │    │
  │  │                                                                        │    │
  │  │  ┌────────────────────────┐       ┌─────────────────────────────┐     │    │
  │  │  │  PRIVATE (10.0.11.0/24)│       │  PRIVATE (10.0.12.0/24)     │     │    │
  │  │  │  ┌──────────────────┐  │       │  ┌──────────────────────┐   │     │    │
  │  │  │  │  App Server A1   │  │       │  │  App Server B1        │   │     │    │
  │  │  │  │  App Server A2   │  │       │  │  App Server B2        │   │     │    │
  │  │  │  └──────────────────┘  │       │  └──────────────────────┘   │     │    │
  │  │  │  Route: 0.0.0.0/0 →   │       │  Route: 0.0.0.0/0 →        │     │    │
  │  │  │    NAT GW-A            │       │    NAT GW-B                 │     │    │
  │  │  └────────────────────────┘       └─────────────────────────────┘     │    │
  │  │                                                                        │    │
  │  │  ┌────────────────────────┐       ┌─────────────────────────────┐     │    │
  │  │  │ ISOLATED (10.0.21.0/24)│       │ ISOLATED (10.0.22.0/24)     │     │    │
  │  │  │  ┌──────────────────┐  │       │  ┌──────────────────────┐   │     │    │
  │  │  │  │  RDS Primary     │  │  ───► │  │  RDS Standby (Multi- │   │     │    │
  │  │  │  │  (PostgreSQL)    │  │       │  │  AZ Replica)         │   │     │    │
  │  │  │  └──────────────────┘  │       │  └──────────────────────┘   │     │    │
  │  │  │  No internet route     │       │  No internet route           │     │    │
  │  │  └────────────────────────┘       └─────────────────────────────┘     │    │
  │  └──────────────────────────────────────────────────────────────────────┘    │
  └─────────────────────────────────────────────────────────────────────────────────┘

Security Groups:
  sg-alb:  inbound 80/443 from 0.0.0.0/0
  sg-app:  inbound 8080 from sg-alb only
  sg-db:   inbound 5432 from sg-app only
```

### Traffic Flow (User Request)

```
User Browser
    │ HTTPS to ALB DNS
    ▼
Route 53 → ALB DNS → ALB (distributes across AZ-A and AZ-B nodes)
    │
    ▼
ALB → forwards to App Server (in Private Subnet, any AZ)
    │
    ▼
App Server → queries RDS endpoint → RDS Primary (Isolated Subnet AZ-A)
    │                               (RDS handles failover to AZ-B if needed)
    ▼
Response flows back through ALB → User
```

### SSM Session Manager vs Bastion Host

Traditional bastion hosts require an open SSH port. Use **AWS Systems Manager Session Manager** instead:

| Feature                     | Bastion Host         | SSM Session Manager    |
|-----------------------------|----------------------|------------------------|
| Open inbound SSH port       | Yes (port 22)        | No                     |
| Requires public IP          | Yes                  | No                     |
| Audit logging               | Manual setup         | Built-in (CloudTrail)  |
| IAM-controlled access       | Partial              | Full                   |
| Cost                        | EC2 instance cost    | Free (interface endpoint optional) |

---

## 18. CIDR Planning Cheat Sheet

Good IP planning prevents painful future re-architectures. Plan for 3-5x your current scale.

### RFC 1918 Private Address Ranges

| Range               | Hosts Available       | Common use            |
|---------------------|-----------------------|-----------------------|
| 10.0.0.0/8          | ~16.7 million         | Enterprise / AWS VPCs |
| 172.16.0.0/12       | ~1 million            | Mid-size networks     |
| 192.168.0.0/16      | ~65,000               | Small / home networks |

### VPC CIDR Sizing

| VPC CIDR   | Total IPs | Usable (approx) | Good for                          |
|------------|-----------|-----------------|-----------------------------------|
| /16        | 65,536    | ~65,000         | Large production VPCs             |
| /17        | 32,768    | ~32,000         | Medium production VPCs            |
| /20        | 4,096     | ~4,000          | Small workload VPC                |
| /24        | 256       | 251             | Tiny VPC (not recommended)        |

> Use `/16` for every production VPC. Future subnetting flexibility is worth the address space.

### Subnet Sizing Reference

| Subnet CIDR | Total IPs | Usable IPs | Typical use                            |
|-------------|-----------|------------|----------------------------------------|
| /24         | 256       | 251        | General subnets                        |
| /25         | 128       | 123        | Smaller tier subnets                   |
| /26         | 64        | 59         | Small service subnets                  |
| /27         | 32        | 27         | TGW attachments, VPN endpoints         |
| /28         | 16        | 11         | Interface endpoint subnets, minimal    |

### Recommended Layout for a Production VPC (10.0.0.0/16)

```
VPC: 10.0.0.0/16

Public Subnets (one per AZ):
  10.0.1.0/24   AZ-A
  10.0.2.0/24   AZ-B
  10.0.3.0/24   AZ-C

Private / App Subnets:
  10.0.11.0/24  AZ-A
  10.0.12.0/24  AZ-B
  10.0.13.0/24  AZ-C

Isolated / DB Subnets:
  10.0.21.0/24  AZ-A
  10.0.22.0/24  AZ-B
  10.0.23.0/24  AZ-C

Reserved for future use:
  10.0.100.0/22  (management, VPN endpoints, endpoints)
  10.0.200.0/22  (future expansion)
```

### Multi-VPC CIDR Strategy (no overlapping!)

When planning multiple VPCs that might ever be peered or connected through TGW:

```
Production VPC:    10.0.0.0/16
Staging VPC:       10.1.0.0/16
Development VPC:   10.2.0.0/16
Shared Services:   10.3.0.0/16
Management VPC:    10.4.0.0/16
On-premises DC:    192.168.0.0/16  (pre-assigned, don't overlap)
```

Using `/16` per VPC in the `10.x.0.0` space gives you 256 distinct VPCs before you run out — ample for most organizations.

---

## Quick Reference — Service Limits

| Resource                          | Default Limit       |
|-----------------------------------|---------------------|
| VPCs per region                   | 5                   |
| Subnets per VPC                   | 200                 |
| Route tables per VPC              | 200                 |
| Routes per route table            | 50 (1000 with propagation) |
| Security groups per VPC           | 2,500               |
| Rules per security group          | 60 inbound + 60 outbound |
| NACLs per VPC                     | 200                 |
| Rules per NACL                    | 20 (each direction) |
| Internet gateways per region      | 5                   |
| NAT Gateways per AZ               | 5                   |
| EIPs per region                   | 5                   |
| VPC peering connections per VPC   | 50                  |

> All limits above are **soft limits** (requestable via AWS Support).

---

## Summary

| Component            | Layer        | Purpose                                              |
|----------------------|--------------|------------------------------------------------------|
| VPC                  | L3 network   | Isolated private network slice in AWS                |
| Subnet               | L3 segment   | AZ-scoped IP range within VPC                        |
| Route Table          | L3 routing   | Controls where traffic is forwarded                  |
| IGW                  | L3 gateway   | Internet access for public subnet resources          |
| NAT Gateway          | L4 NAT       | Outbound internet for private subnet resources       |
| Security Group       | L4 stateful  | Instance-level allow rules                           |
| NACL                 | L3/4 stateless | Subnet-level allow/deny rules                      |
| VPC Endpoint         | L3 private   | Private access to AWS services                       |
| VPC Peering          | L3 routing   | Private 1:1 VPC connectivity                        |
| Transit Gateway      | L3 hub       | Multi-VPC + hybrid hub-and-spoke routing             |
| Site-to-Site VPN     | L3 IPsec     | Encrypted on-prem to VPC over internet               |
| Direct Connect       | L1/L2 dedicated | Private physical link to AWS                      |
| ENI                  | L2 vNIC      | Virtual network card for EC2 and services            |
| Elastic IP           | L3 addressing | Static public IPv4 address                          |
| Flow Logs            | Monitoring   | Traffic metadata capture for analysis                |
| Route 53 Resolver    | DNS          | VPC-internal DNS with hybrid forwarding              |
