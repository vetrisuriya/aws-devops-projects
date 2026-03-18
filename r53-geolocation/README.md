# AWS Route 53 — Geolocation-Based DNS Routing

> **Project:** DNS-layer geo-restriction using Route 53 Geolocation Routing Policy  
> **Domain:** vetrisuriya.in  
> **Region:** ap-northeast-1 (Tokyo) · ap-south-1 (Mumbai for testing)  
> **Stack:** Route 53 · EC2 (Tokyo) · EC2 (Mumbai) · Apache HTTP

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Architecture Summary](#architecture-summary)
3. [Step 1 — Route 53 Hosted Zone Setup](#step-1--route-53-hosted-zone-setup)
4. [Step 2 — Geolocation Routing Policy](#step-2--geolocation-routing-policy)
5. [Step 3 — EC2 Instances (Tokyo + Mumbai)](#step-3--ec2-instances-tokyo--mumbai)
6. [Step 4 — Traffic Flow Testing](#step-4--traffic-flow-testing)
7. [How It All Works Together](#how-it-all-works-together)
8. [Key Technical Insights](#key-technical-insights)
9. [Route 53 Routing Policies — Comparison](#route-53-routing-policies--comparison)
10. [Real-World Use Cases](#real-world-use-cases)
11. [What I Learned](#what-i-learned)

---

## Project Overview

This project implements **DNS-layer geographic access control** using AWS Route 53 Geolocation Routing. The goal was to restrict access to an EC2-hosted web application so that **only requests originating from Japan** are resolved to the Tokyo EC2 instance — all other regions receive an `NXDOMAIN` (domain does not exist) response.

This is a lightweight, infrastructure-level restriction — no application code changes required. The DNS layer itself enforces the policy.

---

## Architecture Summary

```
┌────────────────────────────────────────────────────────────┐
│                   DNS Resolution Flow                      │
└────────────────────────────────────────────────────────────┘

Japan User
    │
    ▼
Route 53 Resolver
    │  Geolocation check: Is source IP from Japan?
    │
    ├──► YES → Resolve to EC2 Tokyo (ap-northeast-1)  ✅
    │           Apache serves response
    │
    └──► NO  → NXDOMAIN (no default record)           ❌
                Domain does not exist for this region


Mumbai EC2 (Testing)
    │
    ├──► DNS query via vetrisuriya.in → NXDOMAIN       ❌ (not Japan)
    └──► Direct public IP access     → Apache responds ✅ (bypasses DNS)
```

---

## Step 1 — Route 53 Hosted Zone Setup

A **Route 53 Public Hosted Zone** was created for the domain `vetrisuriya.in`. The hosted zone manages all DNS records for the domain and is the control plane for the geolocation routing policy.

| Property | Value |
|---|---|
| Domain | `vetrisuriya.in` |
| Hosted Zone Type | Public |
| Service | Amazon Route 53 |
| Record TTL | 300 seconds |

**What is a Hosted Zone?**

A hosted zone is a container for DNS records for a domain. When a DNS query reaches Route 53 for `vetrisuriya.in`, Route 53 evaluates all records in this hosted zone and returns the appropriate response based on the routing policy configured.

---

## Step 2 — Geolocation Routing Policy

The core of this project is the **Geolocation Routing Policy** applied to the A record for `vetrisuriya.in`.

| Property | Value |
|---|---|
| Record Type | A Record |
| Routing Policy | Geolocation |
| Location | Japan |
| Target | EC2 Tokyo public IP (redacted) |
| TTL | 300 seconds |
| Default Record | ❌ Not configured (intentional) |

**Why no default record?**

This is a deliberate design decision. Without a default record:

- Requests from Japan → resolved to Tokyo EC2 ✅
- Requests from any other region → `NXDOMAIN` ❌

If a default record were added, non-Japan requests would still resolve (to the default target). Omitting the default record enforces a strict allowlist — **only Japan resolves**.

**How Route 53 Geolocation Works**

Route 53 determines the geographic location of a DNS query based on the **resolver's IP address** (the recursive DNS resolver the client uses — typically the ISP's resolver), not the client's IP. This is an important distinction:

- A user in Japan using a Japanese ISP → Japanese resolver IP → matches Japan geolocation ✅
- A user using a VPN with a Tokyo exit node → may or may not match depending on the resolver

---

## Step 3 — EC2 Instances (Tokyo + Mumbai)

### Production Instance — Tokyo (ap-northeast-1)

| Property | Value |
|---|---|
| Region | ap-northeast-1 (Tokyo) |
| Purpose | Production web server |
| Web Server | Apache HTTP |
| Access | Via DNS (vetrisuriya.in) from Japan only |

The Tokyo instance runs Apache HTTP and serves the web application. It is the **only registered target** in the Route 53 A record.

### Test Instance — Mumbai (ap-south-1)

| Property | Value |
|---|---|
| Region | ap-south-1 (Mumbai) |
| Purpose | Testing geo-restriction behavior |
| Web Server | Apache HTTP |
| Access | Direct IP only (not in DNS) |

The Mumbai instance was used to **validate the geolocation restriction**. Since Mumbai is not Japan, DNS queries from this instance returned `NXDOMAIN` — confirming the policy works correctly.

---

## Step 4 — Traffic Flow Testing

Three test scenarios were validated:

### Scenario 1 — Japan User → DNS Query ✅

```
Japan client → DNS query for vetrisuriya.in
           → Route 53 checks geolocation
           → Source matches Japan
           → Returns: Tokyo EC2 public IP
           → Client connects to Tokyo instance
           → Apache responds with web page
Result: SUCCESS ✅
```

### Scenario 2 — Mumbai EC2 → DNS Query ❌

```
Mumbai EC2 → DNS query for vetrisuriya.in
           → Route 53 checks geolocation
           → Source is ap-south-1 (India, not Japan)
           → No matching record, no default record
           → Returns: NXDOMAIN
           → Connection fails
Result: NXDOMAIN as expected ✅ (geo-restriction working)
```

### Scenario 3 — Direct IP Access (Bypass DNS) ✅

```
Any client → Direct HTTP request to Tokyo EC2 public IP
           → Bypasses DNS resolution entirely
           → Apache on EC2 responds directly
Result: SUCCESS ✅ (DNS policy bypassed via direct IP)
```

> **Security Note:** Scenario 3 reveals that DNS-layer restriction alone does not prevent all access. For complete enforcement, combine Route 53 geolocation routing with **Security Group rules** or **WAF Geo-Match conditions** that block traffic at the network or application layer.

---

## How It All Works Together

```
┌─────────────────────────────────────────────────────────────┐
│                    FULL DNS FLOW                            │
└─────────────────────────────────────────────────────────────┘

Step 1 │ User types vetrisuriya.in in browser
       │
Step 2 │ OS sends DNS query to local recursive resolver
       │
Step 3 │ Resolver forwards query to Route 53 authoritative NS
       │
Step 4 │ Route 53 checks geolocation of resolver IP
       │
Step 5 │ If resolver IP = Japan → return Tokyo EC2 IP (A record)
       │ If resolver IP ≠ Japan → return NXDOMAIN (no default)
       │
Step 6 │ Client receives IP (or NXDOMAIN)
       │
Step 7 │ If IP received → TCP connection → Apache HTTP → response
       │ If NXDOMAIN   → browser shows "Site cannot be reached"
       │
Step 8 │ TTL = 300s → DNS result cached by resolver for 5 minutes
```

---

## Key Technical Insights

### 1. Resolver IP vs Client IP
Route 53 geolocation uses the **recursive resolver's IP**, not the actual client IP. In most cases these match (same country), but VPN users or clients using public resolvers like `8.8.8.8` may get unexpected results. AWS Route 53 uses a continually updated IP-to-location database to map resolver IPs.

### 2. No Default Record = Strict Allowlist
The absence of a default record transforms geolocation routing from a **preferred routing** mechanism into a **hard access control** mechanism. This is a powerful pattern for compliance — only whitelisted geographies can resolve the domain.

### 3. TTL and Propagation
With TTL set to 300 seconds, DNS responses are cached for 5 minutes. Changes to the A record take up to 300 seconds to propagate globally. During testing, lower TTLs (60s) are useful to see changes faster.

### 4. DNS Restriction ≠ Complete Security
DNS restriction is the **first layer** only. Direct IP access bypasses it entirely. For production geo-restriction enforce:
- Route 53 Geolocation → DNS layer
- AWS WAF Geo Match → HTTP layer
- Security Groups / NACLs → Network layer

### 5. Geolocation vs Latency-Based Routing

| Policy | What It Does | Use Case |
|---|---|---|
| Geolocation | Routes based on **where** the user is | Compliance, content licensing, access control |
| Latency-Based | Routes to the **lowest latency** region | Performance optimization |
| Geoproximity | Routes based on distance with bias control | Fine-grained regional distribution |

---

## Route 53 Routing Policies — Comparison

| Policy | Decision Basis | Use Case |
|---|---|---|
| **Simple** | Single record, no logic | Basic DNS — one target |
| **Weighted** | Percentage split across records | A/B testing, canary deployments |
| **Latency-Based** | Lowest network latency to region | Global apps needing low latency |
| **Failover** | Primary/secondary with health checks | Active-passive disaster recovery |
| **Geolocation** | User's geographic location | Compliance, regional content |
| **Geoproximity** | Distance with configurable bias | Traffic shifting between regions |
| **Multi-Value** | Returns multiple healthy IPs | Basic load distribution |
| **IP-Based** | Specific CIDR ranges | Custom routing for known IP blocks |

---

## Real-World Use Cases

| Use Case | How Geolocation Routing Helps |
|---|---|
| **Regulatory compliance** | Ensure EU users only hit EU-region servers (GDPR) |
| **Content licensing** | Stream media only in licensed geographic territories |
| **Latency optimization** | Route users to the nearest regional endpoint |
| **A/B testing by region** | Test new features in one country before global rollout |
| **Disaster recovery** | Route away from a failed region during an outage |
| **Beta rollouts** | Limit access to users in a specific country during testing |

---

## What I Learned

- **Route 53 Geolocation Routing** works at the DNS layer — no application changes required to enforce geographic restrictions
- **Not configuring a default record** is a valid and intentional pattern — it turns routing into an explicit allowlist
- **Resolver IP ≠ Client IP** — understanding this distinction is critical when debugging unexpected geolocation results
- **DNS TTL matters** — a high TTL means changes take longer to propagate; always lower TTL before making changes
- **DNS restriction alone is not sufficient** for complete geo-blocking — it must be layered with WAF and Security Groups for production enforcement
- **Direct IP access always bypasses DNS** — any security model relying solely on DNS is incomplete

---