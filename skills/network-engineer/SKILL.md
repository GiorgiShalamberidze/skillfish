---
name: network-engineer
description: Network engineering: VPC/VNet design, routing, DNS, load balancers, firewalls, segmentation, troubleshooting, and traffic change planning.
---

# Network Engineer

Design and troubleshoot networks by following the packet path, not by guessing. This skill focuses on clear segmentation, deterministic routing, DNS and TLS correctness, load-balancer behavior, firewall intent, and safe rollout plans for production traffic changes.

## When to Use

- Designing VPC, VNet, subnet, peering, or hybrid-connectivity layouts
- Planning DNS, TLS, load balancer, CDN, firewall, or NAT changes
- Troubleshooting latency, intermittent reachability, packet loss, or routing issues
- Segmenting environments, teams, or service tiers with least-privilege access
- Reviewing east-west and north-south traffic controls
- Writing migration or rollback plans for network changes

## Workflow

1. Map the full data path.
   Identify client, DNS, CDN, edge, load balancer, firewall, route table, NAT, service mesh, and destination dependencies before debugging.
2. Verify naming and addressing.
   Confirm records, TTL, IP ranges, subnet boundaries, overlapping CIDRs, and source/destination expectations.
3. Check control planes before data planes.
   Look at DNS resolution, route tables, ACLs, security groups, firewall policies, and load-balancer listeners before blaming the application.
4. Isolate one layer at a time.
   Reproduce with direct IP, then host, then service VIP, then public endpoint so the failing hop becomes obvious.
5. Stage traffic changes.
   Use canaries, weighted routing, low TTLs, and explicit rollback windows for any customer-facing cutover.
6. Leave behind diagrams and guardrails.
   Update topology docs, IP ownership, certificate expiry tracking, and change runbooks after each significant change.

## Troubleshooting Ladder

| Layer | Questions to Answer |
|---|---|
| DNS | Does the name resolve to the expected record, and is TTL acceptable for the change window? |
| Routing | Does traffic have a valid path both ways without asymmetric routing or overlapping CIDRs? |
| Security | Are firewall, ACL, and security-group rules aligned with the real source and destination? |
| Load balancing | Are health checks, listener rules, stickiness, and target registration correct? |
| TLS | Does SNI, certificate chain, protocol version, and trust path match the client behavior? |
| Application | Is the service actually listening and returning the expected response on the expected port? |

## Design Rules

- Use explicit segmentation by environment, service tier, and trust boundary.
- Minimize shared flat networks; they make blast radius and troubleshooting worse.
- Prefer DNS names over hardcoded addresses in application configs.
- Keep low TTLs for migrations only as long as needed; permanent low TTLs add resolver load.
- Document ownership for every zone, load balancer, VIP, and external dependency.
- Treat firewall rules as code and review them like application changes.

## Safe Change Checklist

- Current path diagram and dependency list
- Pre-change health snapshot
- Rollback criteria and owner
- DNS TTL plan if cutover is name-based
- Validation from multiple network vantage points
- Post-change monitoring for latency, error rate, and health-check behavior

## Output Format

When using this skill, produce:

1. **Traffic path map** - source, hops, controls, and destination
2. **Problem or design summary** - what must work, what is failing, and where the risk sits
3. **Layer-by-layer plan** - DNS, routing, security, load balancing, TLS, and service checks
4. **Change plan** - canary, cutover, monitoring, and rollback steps
5. **Documentation updates** - diagrams, ownership, and guardrails to retain
