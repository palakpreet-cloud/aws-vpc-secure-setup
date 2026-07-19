# VPC Networking — Core Concepts

This document explains the core networking concepts behind the architecture built in this repo, in plain language with reference to the actual resources created.

## 1. VPC (Virtual Private Cloud)

A VPC is your own isolated network inside AWS — a private slice of the cloud where you define the IP address range and control everything inside it.

- **This repo's VPC**: `my-secure-vpc`, CIDR block `10.0.0.0/16`
- `/16` means 65,536 IP addresses are available in total, split across subnets.

## 2. Subnets

A subnet is a smaller range of IPs carved out of the VPC's CIDR block, tied to a single Availability Zone (AZ).

- This setup has **4 subnets across 2 AZs**:
  - Public Subnet A
  - Public Subnet B
  - Private Subnet A
  - Private Subnet B

**Why 2 AZs?** High availability. If one AZ goes down, resources in the other AZ keep running. This is a standard production pattern, not just a learning exercise.

**Public vs Private — what actually makes a subnet "public"?**
A subnet is public *only* because its route table sends `0.0.0.0/0` traffic to an Internet Gateway. There's no special "public" flag on the subnet itself — it's entirely about routing.

## 3. Internet Gateway (IGW)

- **Resource**: `my-igw`, attached to `my-secure-vpc`
- An IGW is the door between your VPC and the public internet.
- It allows two-way traffic: resources in public subnets can reach the internet, and (if they have public IPs) be reached from it.
- Without an IGW attached, *nothing* in the VPC can talk to the internet — no matter how route tables are configured.

## 4. Route Tables

A route table is a set of rules that decides where network traffic is directed, based on destination IP.

- **`public-rt`**: routes `0.0.0.0/0` (all internet-bound traffic) → IGW
  - Associated with Public Subnet A and Public Subnet B
- **`private-rt`**: routes `0.0.0.0/0` → NAT Gateway
  - Associated with Private Subnet A and Private Subnet B

**Key insight**: Every subnet has a route table. If you don't explicitly associate one, it uses the VPC's default route table. The route table is what determines a subnet's "personality" (public or private).

## 5. NAT Gateway

- **Resource**: `my-nat-gw`, zonal, placed in **Public Subnet A**
- NAT = Network Address Translation.
- Purpose: lets resources in **private subnets** initiate outbound connections to the internet (e.g., to download OS updates or hit an API) **without** being directly reachable from the internet.
- **Why it must live in a public subnet**: the NAT gateway itself needs internet access via the IGW. Private subnet traffic flows: Private Subnet → NAT Gateway (in public subnet) → IGW → Internet.
- **One-way door**: Internet → NAT Gateway is blocked. NAT only allows traffic that was *initiated* from inside.
- **Zonal vs cross-AZ**: A zonal NAT gateway only serves subnets in its own AZ efficiently. For true production HA, you'd typically want one NAT gateway per AZ — noted here as a future improvement, not implemented in this version (cost trade-off).

## 6. Security Groups (SGs) — and SG Chaining

A security group is a virtual firewall attached to resources (like EC2 instances), controlling inbound/outbound traffic at the **instance level** (route tables operate at the **subnet level** — different layer).

- **`web-sg`**: allows inbound traffic on ports 80 (HTTP) and 443 (HTTPS) from anywhere (`0.0.0.0/0`)
  - This is appropriate for a public-facing web server.
- **`db-sg`**: allows inbound traffic on port 3306 (MySQL) **only from `web-sg`** — not from any IP range.

**This is SG chaining**: instead of writing `db-sg`'s rule as "allow from IP range X," it's written as "allow from anything that belongs to `web-sg`." This means:
- Only instances tagged with `web-sg` can reach the database.
- If the web tier scales up/down or gets new IPs, the rule still works — no manual IP updates needed.
- This is a core defense-in-depth pattern: even if someone bypasses the network layer, the database tier is locked down to a specific application tier, not an IP range.

## 7. How It All Fits Together

See the architecture diagram in README.md for the visual layout. In summary:

- The Internet connects to the VPC through the Internet Gateway (my-igw), which allows two-way traffic.
- The IGW connects to Public Subnet A and Public Subnet B, both using public-rt, which routes 0.0.0.0/0 to the IGW.
- The NAT Gateway (my-nat-gw) sits in Public Subnet A and provides outbound-only internet access for private subnets.
- Private Subnet A and Private Subnet B both use private-rt, which routes 0.0.0.0/0 to the NAT Gateway.
- db-sg allows inbound traffic on port 3306 only from web-sg — the database tier is never directly reachable from the internet, only from the web tier.

## 8. Why These Choices (Security Rationale)

| Decision | Reason |
|---|---|
| 2 AZs | High availability — survive an AZ failure |
| Separate public/private subnets | Defense in depth — DB never directly exposed |
| NAT in public subnet only | Private resources get outbound internet without inbound exposure |
| SG chaining (db-sg → web-sg) | Database access tied to application identity, not IP — more resilient and auditable |
| IAM user `preet-admin` instead of root | Root account reserved for account-level tasks only; follows least-privilege principle |
| Root MFA enabled | Protects the most powerful account credential from compromise |

## 9. Terms Glossary (Quick Reference)

- **CIDR block**: A way of specifying an IP range and its size (e.g., `10.0.0.0/16`)
- **AZ (Availability Zone)**: A physically separate data center within an AWS region
- **Ingress / Egress**: Inbound / outbound traffic
- **0.0.0.0/0**: Shorthand for "all IPv4 addresses" — used to mean "anywhere"
- **Least privilege**: Giving an identity only the permissions it needs, nothing more

## 10. Where Theory Meets Practice

Understanding these concepts on paper is different from knowing what breaks when 
one is missing. See [TROUBLESHOOTING.md](./TROUBLESHOOTING.md) for four deliberate 
misconfigurations (SG rule removal, NAT route removal, IGW detachment, an open 
database exposure) and what actually happened in each case.
