# Discovery Report: Designing a Secure Network for the Onward Logistics Order System

| | |
|---|---|
| **Lab** | Designing a Secure Network for Onward Logistics Order System (Week 2) |
| **Name** | Effie Njoki |
| **Date** | 26 October 2026 |
| **Client** | Onward Logistics (see the Client Profile) |

---

## 1. What I found

### The client's need

Onward Logistics needs its order system to stay open to merchants and customers even when part of AWS fails, while keeping its internal systems hidden from the internet, letting engineers fix servers without risky shortcuts, and spending only what it can justify.

### Solutions that exist

| Solution | How it works | Trade-off for this client |
|---|---|---|
| **Single server (today's setup)** | One hand-built server holds the website and order services, reachable from the internet. | **Strength:** cheapest and simplest. **Weakness:** one failure takes everything down, and internal services are exposed to the internet. Unacceptable when sales days triple the volume. |
| **Public servers with a bastion host** | Servers sit in public subnets; admins log in through one "jump" server over SSH with shared keys. | **Strength:** familiar to many teams. **Weakness:** port 22 is open to the internet, keys get shared and never expire, and the bastion is one more server to patch and pay for. Fails the Head of Security's requirements. |
| **Multi-AZ VPC with a load balancer, private servers and Session Manager (chosen)** | A load balancer in two Availability Zones sends customers to front-end servers in private subnets; internal services are private; engineers connect with Session Manager; a NAT gateway provides updates. | **Strength:** survives the loss of an AZ, nothing internal is reachable from the internet, and no admin ports or keys exist. **Weakness:** costs more than a single server (mainly the load balancer and NAT gateways) and needs more knowledge to run. |

### What happens with no solution

**Lost sales on peak days.** With one server in one data centre, any failure takes the whole order system offline. Outages do happen even on the largest platforms: in October 2025, a failure in AWS's US-EAST-1 region disrupted thousands of companies for hours, including banks and messaging apps. Delivery volume triples on Onward's sales days, so an outage then would cost merchants and customer trust, which the Client Profile calls the product itself.

**A larger attack surface.** Servers with public addresses and open admin ports are probed by automated attacks around the clock. One leaked or shared key can give an attacker control of the order system and the customer data behind it, in markets where data protection rules apply.

**Keys that outlive the people who held them.** Shared SSH keys are rarely changed when staff leave. In 2018, a former Cisco engineer whose access had not been removed deleted 456 servers, shutting down 16,000 customer accounts for up to two weeks and costing about $2.4 million.

**Unexplained spend.** Network resources bill by the hour whether used or not. Without a costed design, the Finance Lead cannot tell which charges are necessary, and the budget is limited.

### Sources

- The Pragmatic Engineer, *AWS takes down a good part of the internet* (October 2025): https://newsletter.pragmaticengineer.com/p/the-pulse-aws-takes-down-a-good-part
- U.S. Department of Justice, *San Jose Man Sentenced To Two Years Imprisonment For Damaging Cisco's Network*: https://www.justice.gov/usao-ndca/pr/san-jose-man-sentenced-two-years-imprisonment-damaging-cisco-s-network
- AWS, *Amazon VPC pricing* (NAT gateways, public IPv4): https://aws.amazon.com/vpc/pricing/
- AWS, *Elastic Load Balancing pricing*: https://aws.amazon.com/elasticloadbalancing/pricing/

---

## 2. The business value of my solution

### My chosen approach

I chose a **two-Availability-Zone network** built entirely in CloudFormation and deployed through Git sync. Customers reach the order system through an **Application Load Balancer**, which spreads traffic across front-end servers in **both** AZs; an **Auto Scaling group** replaces any server that fails. The front-end servers and the internal order services all sit in **private subnets** with no public addresses, so nothing internal can be reached from the internet. Private servers download updates through **NAT gateways**, and engineers administer them with **Session Manager**, so no admin ports are open and no keys exist.

This meets each leader's need directly. The **CTO** gets a front end that keeps working if one AZ fails. The **Head of Security** gets private servers, tightly scoped security groups, and no SSH or shared keys. The **Head of Operations** gets logins and updates without weakening security. The **Finance Lead** gets a costed design with the rejected alternatives explained below.

I rejected a **single NAT gateway** because, if its AZ failed, private servers in the other AZ would also lose updates, which contradicts the high-availability requirement. I rejected **interface endpoints for Session Manager**, because the NAT gateways already give the servers the outbound path they need, and three endpoints in two AZs would add about $44 a month. I added a **free S3 gateway endpoint**, because Amazon Linux downloads its updates from S3, so update traffic avoids NAT data charges.

### Services I will use

| Service | The job it does | Why this service, used this way |
|---|---|---|
| **Amazon VPC** (2 public + 2 private subnets, route tables, internet gateway) | Onward's private network, spread across two AZs | Two AZs remove the single point of failure; private subnets keep internal systems off the internet. |
| **Application Load Balancer** | The single public entry point that shares traffic across both AZs | Customers keep reaching the site if one AZ or server fails; it also means front-end servers need no public addresses. |
| **EC2 Auto Scaling** | Keeps two front-end servers running and replaces failed ones | Recovery happens automatically, without an engineer at 3 a.m. |
| **Amazon EC2** (Free Tier eligible, Amazon Linux 2023) | Runs the front end and the internal order services | Free Tier eligible size keeps cost low for this stage. |
| **NAT gateways** (one per AZ) | Let private servers start outbound connections for updates | Outbound only: the internet can never start a connection in. One per AZ keeps updates working if an AZ fails. |
| **S3 gateway endpoint** | A private, free path to S3 | Update downloads skip NAT data charges, at no cost. |
| **Security groups** | Firewalls around each server and the load balancer | Front-end servers accept web traffic only from the load balancer; internal servers accept nothing from the internet. |
| **Systems Manager Session Manager** + an IAM role | Lets engineers open a terminal on any server | No port 22, no keys; access is controlled by IAM and every session is logged in CloudTrail. |
| **CloudFormation with Git sync** | Builds everything from a template in Git | Every change has an author and a time, and the whole network can be rebuilt from the repository. |

### Estimated monthly cost (eu-west-1, about 730 hours)

| Item | Estimate | Notes |
|---|---|---|
| Application Load Balancer | about $18 to $25 | Hourly charge plus low traffic usage |
| Load balancer public IPv4 (2 AZs) | about $7 | $0.005 per address per hour |
| NAT gateways (2, one per AZ) | about $70 + data | About $0.048 per hour each in Ireland |
| NAT gateway public IPv4 (2) | about $7 | |
| EC2 servers (4 × Free Tier eligible size) | $0 to about $33 | Depends on Free Tier hours remaining |
| S3 gateway endpoint, Session Manager | $0 | No charge |
| **Total** | **about $100 to $140 a month** | |

**Ways to reduce it:** use a single NAT gateway in dev and test only (about $35 less), switch off non-production environments outside working hours, and review the load balancer's usage after the first peak day.

### Who benefits, and how I will know

| Stakeholder | Benefit | How I would know |
|---|---|---|
| **CTO** | The front end stays online if an AZ fails | Stopping a front-end server in one AZ: the site still loads, and Auto Scaling replaces the server |
| **Head of Security** | Nothing internal is exposed; no admin ports or keys | Internal servers have no public IP; no security group allows port 22; connection attempts from the internet fail |
| **Head of Operations** | Engineers can fix servers and install updates | Session Manager opens on every server; a package installs successfully on an internal server |
| **Finance Lead** | Spending is explained and justified | A cost table with each item, the alternatives rejected, and ways to reduce it |
| **Merchants and customers** | Orders can be placed even on peak days | The website stays available during the AZ failure test |

### Limits and risks of my design

- **One region only.** The design survives the loss of an Availability Zone, not of a whole region like the October 2025 US-EAST-1 outage. A second region would add significant cost and complexity.
- **No HTTPS yet.** The load balancer serves HTTP for this lab; production needs an HTTPS certificate.
- **Fixed size.** Two front-end servers are enough for the lab; real sales-day traffic would need scaling rules on the Auto Scaling group.
- **NAT is the main cost.** Two NAT gateways are the price of resilient updates; regional NAT or a single NAT in non-production environments are options to revisit.
- **Not covered:** a database tier, backups, monitoring alarms and budget alerts.
