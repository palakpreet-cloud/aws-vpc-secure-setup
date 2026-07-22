## Day 1 — VPC, Subnets, Routing, SG Chaining

### What I built
- VPC `my-secure-vpc` (10.0.0.0/16), 4 subnets across 2 AZs (ap-south-1a, ap-south-1b)
- Internet Gateway `my-igw`, attached to the VPC
- `public-rt` (routes to IGW) and `private-rt` (local only, NAT route added Day 2)
- `web-sg` (80/443 from 0.0.0.0/0) and `db-sg` (3306, source = web-sg via SG chaining)

### Experiment: breaking db-sg's inbound rule

**What I did:**
Deleted the only inbound rule on `db-sg` (MySQL/3306, source `web-sg`),
leaving the security group with 0 inbound permission entries.

![db-sg broken](screenshots/troubleshooting/day1-dbsg-broken.png)

**What I expected to happen:**
If an EC2 instance in `web-sg` tried to reach a database instance in
`db-sg` on port 3306, the connection would **hang and eventually time out**
— not get an instant "connection refused."

**Why this happens:**
Security groups are stateful *allow-lists*, not firewalls with explicit
deny rules. There's no "reject" rule to trigger an immediate response —
if no inbound rule matches the incoming traffic, the packet is silently
dropped at the network interface level. The client's TCP handshake never
gets a SYN-ACK back, so the client just waits until its own connection
timeout expires (typically 30s+, depending on the client). This is a key
difference from something like a Network ACL, which can have explicit
deny rules that reject traffic immediately.

**How I'd confirm this in a real scenario:**
Since there's no EC2 instance in this VPC yet, this is a structural/logic
test rather than a live traffic test. In a real deployment, I'd:
1. Check the SG inbound rules first — fastest way to catch this
2. Enable VPC Flow Logs (filtered to REJECT) to confirm the packet was
   actually dropped rather than some other issue (e.g. wrong port, SG
   not attached to the DB instance, NACL blocking it)

**Fix applied:**
Re-added the inbound rule: Type = MySQL/Aurora, Port = 3306,
Source = `web-sg` (security group reference, not a CIDR — this is what
makes it SG chaining rather than a plain IP allowlist).

![db-sg fixed](screenshots/troubleshooting/day1-dbsg-fixed.png)

**Takeaway:**
SG-to-SG references are safer than CIDR-based rules for internal traffic
like this, because access is tied to *group membership* rather than a
specific IP range. Even if the DB's private IP were somehow known/guessed,
traffic can't reach it unless it originates from something inside `web-sg`.

## Day 2 — NAT Gateway + Private Route Break

### What I built
- Elastic IP allocated for NAT Gateway use
- NAT Gateway `my-nat-gw` created in `public-subnet-a` (Zonal, Public connectivity)
- Added route to `private-rt`: `0.0.0.0/0 → my-nat-gw`

This gives private subnets outbound-only internet access — instances there
can reach out (e.g. package updates, calling external APIs) but nothing
from the internet can initiate a connection in, unlike the public subnets
which route directly through the IGW.

### Experiment: breaking the NAT route

**What I did:**
Deleted the `0.0.0.0/0 → my-nat-gw` route from `private-rt`, leaving only
the local route (`10.0.0.0/16 → local`).

![NAT route broken](screenshots/troubleshooting/day2-natroute-broken.png)

**What I expected to happen:**
Any resource in a private subnet would lose all outbound internet access.
Intra-VPC traffic (talking to resources in other subnets within
`10.0.0.0/16`) would keep working fine, since the local route is untouched.

**Why this happens:**
Without a `0.0.0.0/0` route, there's simply no path out of the VPC for
private subnet traffic. It's not a case of AWS rejecting or blocking the
traffic at the edge — the traffic never leaves the VPC's local routing
scope at all, since the route table has no matching destination for it.
This is different from an SG block (Day 1), where traffic reaches the ENI
and gets dropped there — this failure happens earlier, at the routing layer.

**How I'd confirm this in a real scenario:**
An EC2 instance in a private subnet would show connection timeouts on any
outbound request (e.g. `curl`, `apt update`). Checking the route table
would be the fastest diagnostic step — no explicit error message points
to routing as the cause, so this is a "check the boring stuff first" case.

**Fix applied:**
Re-added the route: Destination = `0.0.0.0/0`, Target = `my-nat-gw`.

![NAT route fixed](screenshots/troubleshooting/day2-natroute-fixed.png)

**Takeaway:**
Routing failures and security group failures look identical from the
outside (both present as timeouts), but the root cause and fix location
are completely different — one's a route table problem, the other's a
security group problem. Diagnosing correctly means checking both layers,
not assuming it's always the SG.

### Cleanup
Deleted `my-nat-gw` and released the associated Elastic IP at the end of
the session to avoid ongoing NAT Gateway hourly charges, since Day 3
doesn't require it. The `private-rt` NAT route was also removed since it
pointed to a now-deleted resource. Will recreate NAT Gateway when needed
again in Phase 3.
