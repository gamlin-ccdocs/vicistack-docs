# VICIdial Cloud Deployment: AWS, GCP & DigitalOcean

**Last updated: March 2026 | Reading time: ~18 minutes**

Every VICIdial guide assumes you have a rack, a static IP, and a data center contract. That made sense in 2014. In 2026, the majority of new VICIdial deployments we see at ViciStack start with the same question: can I run this in the cloud?

The short answer is yes. The longer answer is yes, but [VICIdial](/glossary/asterisk/) and cloud infrastructure have fundamentally different assumptions about networking, CPU scheduling, and storage I/O that you need to understand before you spin up an EC2 instance and wonder why half your calls have one-way audio.

This guide covers the real-world deployment of VICIdial on AWS, Google Cloud Platform, and DigitalOcean. We are not going to rehash the [basic installation steps](/blog/vicidial-setup-guide/) --- that guide exists and it is comprehensive. What we are covering here is everything that changes when your VICIdial server runs on a hypervisor in someone else's data center instead of on bare metal you control.

**The pragmatic alternative:** ViciStack deploys VICIdial on dedicated bare-metal infrastructure specifically optimized for real-time telephony workloads. No shared vCPUs, no noisy neighbors, no NAT headaches. [Skip the cloud complexity altogether.](/pricing/)

---

## Why Cloud for VICIdial

Before we get into the how, let us establish when cloud deployment actually makes sense. Not every VICIdial operation belongs in AWS, and pretending otherwise is dishonest.

### When Cloud Makes Sense

**Elastic scaling for seasonal operations.** If your agent count swings from 20 in January to 150 during open enrollment season, cloud infrastructure lets you scale telephony servers up and down without buying hardware that sits idle 8 months a year. Insurance, tax preparation, political campaigns --- these verticals benefit enormously from cloud elasticity.

**Geographic redundancy.** Running [VICIdial clusters](/blog/vicidial-cluster-guide/) across multiple regions gives you disaster recovery that is nearly impossible to achieve with colocated hardware without massive capital expenditure. Your database server in us-east-1, telephony in us-west-2, and a warm standby in eu-west-1 --- that is a real DR strategy.

**No hardware lifecycle management.** No drive replacements at 3 AM. No RAID controller failures. No firmware updates that require a maintenance window. The cloud provider handles the physical layer and you focus on the VICIdial layer.

**Remote teams and distributed operations.** If your agents are already connecting via [WebRTC](/glossary/webrtc/) from their homes, the location of your server matters less. A cloud instance with low latency to your [SIP carrier](/glossary/carrier/) may perform identically to a colocated box.

### When Cloud Does Not Make Sense

**Sustained high-density operations.** If you are running 100+ agents on a stable, predictable workload 12 hours a day, 5 days a week, bare metal is cheaper. Period. The [cost analysis](/blog/vicidial-cost-2026/) is clear: cloud compute at scale costs 2-3x what equivalent bare metal costs.

**Ultra-low-latency requirements.** Bare metal gives you dedicated CPU cores and no hypervisor overhead. For operations where [predictive dialer](/glossary/predictive-dialing/) performance is critical and every millisecond of [AMD](/glossary/amd/) detection matters, dedicated hardware still wins.

**Budget-constrained startups.** If you have 10 agents and $500/month for everything, a refurbished Dell PowerEdge on Hetzner or OVH for $50-80/month will outperform a $200/month cloud instance. Save the cloud budget for when you actually need elasticity.

---

## VICIdial-Specific Cloud Challenges

Before we dive into provider-specific configurations, you need to understand the fundamental challenges that make VICIdial cloud deployment different from deploying a web application.

### SIP NAT Traversal

This is the single biggest pain point and the reason most first-time cloud VICIdial deployments fail. [SIP](/glossary/sip/) was designed in an era when every device had a public IP address. It embeds IP addresses inside the protocol payload, not just in the packet headers. When your VICIdial server sits behind a cloud provider's NAT, the SIP INVITE packets contain the server's private IP (e.g., 10.0.1.45) instead of its public IP. Your carrier sees the private IP, tries to send RTP media there, and it goes nowhere. Result: one-way audio or no audio at all.

The fix involves multiple layers:

**Asterisk SIP configuration** --- In `/etc/asterisk/sip.conf` (or `pjsip.conf` if you are using [PJSIP](/glossary/pjsip/)):

```ini
; sip.conf - NAT settings for cloud deployment
externip=<YOUR_ELASTIC_IP>
localnet=10.0.0.0/8
localnet=172.16.0.0/12
localnet=192.168.0.0/16
nat=force_rport,comedia
directmedia=no
```

For PJSIP (the modern approach):

```ini
; pjsip.conf
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0
external_media_address=<YOUR_ELASTIC_IP>
external_signaling_address=<YOUR_ELASTIC_IP>
local_net=10.0.0.0/8
local_net=172.16.0.0/12
local_net=192.168.0.0/16
```

**Carrier-side configuration** --- Most [SIP carriers](/glossary/carrier/) need to know you are behind NAT. Some require you to enable "NAT keep-alive" or "SIP Comedia" on their portal. Others need you to whitelist the specific public IP so they send media to the right place.

**[STUN/TURN for WebRTC agents](/glossary/nat-traversal/)** --- If your agents connect via ViciPhone (WebRTC), you need a STUN server at minimum. Cloud NAT adds another layer of complexity here. We recommend running your own coturn instance on the same VPC:

```bash
# Install coturn on a small instance in the same VPC
apt install coturn
```

### Latency Requirements

VICIdial's [predictive dialer](/glossary/predictive-dialing/) algorithm makes real-time decisions about when to place calls based on agent availability. The entire pipeline --- from hopper to dial to carrier to answer detection to agent connection --- is latency-sensitive. Here are the numbers that matter:

| Path | Target Latency | Impact of Excess |
|------|---------------|-----------------|
| VICIdial server to SIP carrier | <100ms | AMD false positives, delayed connections |
| Database queries (hopper fills) | <10ms | Hopper starvation, agent idle time |
| Web server to agent browser | <200ms | Agent screen lag, delayed dispositions |
| Inter-server cluster communication | <5ms | Keepalive failures, split-brain scenarios |

The carrier latency requirement is the most critical. If your VICIdial cloud instance is in us-east-1 and your carrier's nearest POP is in Chicago, you are looking at 15-25ms. That is fine. If your instance is in ap-southeast-1 (Singapore) and your carrier is US-only, you are looking at 180-250ms. That will destroy your [AMD accuracy](/blog/vicidial-amd-guide/) and create noticeable delays for agents.

**Rule of thumb:** Deploy your VICIdial server in the same region as your SIP carrier's nearest point of presence.

### Shared vCPU Performance

This is the dirty secret of cloud VICIdial. [Asterisk](/glossary/asterisk/) is extremely sensitive to CPU scheduling jitter. When Asterisk processes [RTP](/glossary/rtp/) audio packets, it needs consistent sub-millisecond CPU access. On bare metal, you get this automatically. On a cloud instance with shared vCPUs, your Asterisk process competes with other tenants' workloads for CPU time.

The symptoms are subtle and maddening: intermittent audio chopiness, occasional one-way audio that fixes itself, AMD accuracy that varies hour by hour. The fix is straightforward but more expensive: **use dedicated/compute-optimized instances, not general-purpose burstable instances.**

Do not run VICIdial on t3/t2 instances (AWS), e2 instances (GCP), or basic Droplets (DigitalOcean). These are burstable, meaning you get a baseline CPU allocation with occasional bursts. Asterisk needs consistent CPU, not bursts. Use c-series (compute-optimized) instances or dedicated host tenancy.

---

## AWS Deployment

AWS is the most common cloud platform for VICIdial deployments, and the most complex to configure correctly. Here is the complete architecture.

### Instance Sizing

| Role | Instance Type | Specs | Monthly Cost (us-east-1) |
|------|-------------|-------|------------------------|
| Single server (<25 agents) | c5.2xlarge | 8 vCPU, 16 GB RAM | ~$245/month |
| DB server (25-100 agents) | m5.xlarge | 4 vCPU, 16 GB RAM | ~$140/month |
| Telephony/dialer server | c5.2xlarge | 8 vCPU, 16 GB RAM | ~$245/month |
| Web server | c5.xlarge | 4 vCPU, 8 GB RAM | ~$124/month |

**Why c5.2xlarge minimum for telephony:** Each outbound agent with [predictive dialing](/glossary/predictive-dialing/) active generates roughly 3-5 concurrent Asterisk channels (the agent channel plus multiple outbound attempts at typical [dial ratios](/settings/auto-dial-level/)). At 50 agents with a 3:1 ratio, that is 150-200 concurrent channels. Asterisk processes these single-threaded per channel, but the overall load requires at least 6-8 dedicated vCPUs to avoid audio quality degradation.

**Why m5.xlarge for the database:** VICIdial's MySQL/MariaDB workload is memory-intensive, not CPU-intensive. The [hopper](/glossary/hopper/) queries, real-time report queries, and `vicidial_log` writes benefit from large RAM more than fast cores. The m5 series gives you a balance of memory and network bandwidth that the c5 series does not.

### Storage Configuration

**EBS gp3 is the correct choice for VICIdial.** Not gp2, not io1, not io2. Here is why:

- **gp3** gives you 3,000 IOPS and 125 MB/s throughput baseline, and you can provision up to 16,000 IOPS independently of volume size. VICIdial's storage pattern --- lots of small random writes (database) and large sequential writes ([call recordings](/glossary/recording/)) --- fits gp3 perfectly.
- **gp2** scales IOPS with volume size (3 IOPS/GB), so you need a 1TB volume just to get 3,000 IOPS. Wasteful.
- **io1/io2** are overkill for VICIdial unless you are running 200+ agents on a single database server.

Recommended volumes:

```
OS + VICIdial: 100 GB gp3, 3000 IOPS
Database (separate EBS): 200 GB gp3, 6000 IOPS (provisioned)
Recordings: 500 GB gp3, 3000 IOPS baseline
```

Mount the database volume at `/var/lib/mysql` and recordings at `/var/spool/asterisk/monitor/`. Separating these onto different EBS volumes gives you independent I/O paths and simplifies snapshots.

### VPC and Network Configuration

VICIdial requires a carefully designed VPC. Here is the architecture:

```
VPC: 10.0.0.0/16
├── Public Subnet (10.0.1.0/24)
│   ├── VICIdial Web Server (with Elastic IP)
│   └── NAT Gateway
├── Private Subnet (10.0.2.0/24)
│   ├── Database Server
│   └── Telephony Server(s)
└── Public Subnet (10.0.3.0/24)
    └── Telephony Server (if SIP registration requires public IP)
```

**Important decision:** Your telephony server needs a public IP for SIP registration with your carrier. You have two options:

1. **Telephony in a public subnet with an Elastic IP** --- simpler, the server has a direct public IP, SIP NAT traversal is straightforward.
2. **Telephony in a private subnet behind NAT Gateway** --- more secure, but adds another NAT layer that complicates SIP. Not recommended unless your security team absolutely requires it.

We strongly recommend option 1. Put the telephony server in a public subnet with an Elastic IP and use security groups (not network isolation) for access control.

### Security Groups

VICIdial needs these ports open. Get these wrong and you will spend hours debugging connectivity issues.

**Telephony Server Security Group:**

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 5060 | UDP | Carrier IPs | SIP signaling |
| 5061 | TCP | Carrier IPs | SIP TLS (if used) |
| 10000-20000 | UDP | 0.0.0.0/0 | RTP media |
| 4569 | UDP | Cluster servers | [IAX2](/glossary/iax2/) inter-server |
| 3306 | TCP | VPC CIDR | MySQL (from web/DB) |
| 443 | TCP | Agent IPs | WebRTC/ViciPhone |

**Web Server Security Group:**

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 80 | TCP | 0.0.0.0/0 | HTTP (redirect to HTTPS) |
| 443 | TCP | 0.0.0.0/0 | HTTPS agent interface |
| 8089 | TCP | Agent IPs | WebSocket for WebRTC |

**Database Server Security Group:**

| Port | Protocol | Source | Purpose |
|------|----------|--------|---------|
| 3306 | TCP | VPC CIDR only | MySQL/MariaDB |

**Critical:** The RTP port range (10000-20000) must be open to 0.0.0.0/0 on UDP. These are the ports that carry actual voice audio. If you restrict them to carrier IPs only, you will break calls whenever the carrier routes media through a different POP than signaling. Yes, this feels wrong from a security perspective. It is how SIP works.

### Elastic IP Configuration

Assign an Elastic IP to your telephony server and **never release it**. SIP trunks are registered against this IP. Your carrier has whitelisted this IP. Your Asterisk `externip` setting points to this IP. Changing it means updating carrier configs, Asterisk configs, firewall rules, and potentially re-registering all trunks. Treat your Elastic IP like a phone number --- it is part of your identity.

```bash
# Allocate and associate an Elastic IP via AWS CLI
aws ec2 allocate-address --domain vpc --region us-east-1
aws ec2 associate-address --instance-id i-0abc123def456 --allocation-id eipalloc-0abc123
```

### Terraform Example for AWS VICIdial

Here is a condensed Terraform configuration for the key resources in a single-server VICIdial deployment:

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_vpc" "vicidial" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  tags = { Name = "vicidial-vpc" }
}

resource "aws_subnet" "public" {
  vpc_id                  = aws_vpc.vicidial.id
  cidr_block              = "10.0.1.0/24"
  availability_zone       = "us-east-1a"
  map_public_ip_on_launch = true
}

resource "aws_internet_gateway" "gw" {
  vpc_id = aws_vpc.vicidial.id
}

resource "aws_security_group" "vicidial" {
  name   = "vicidial-sg"
  vpc_id = aws_vpc.vicidial.id

  ingress { from_port = 22;    to_port = 22;    protocol = "tcp"; cidr_blocks = ["YOUR_ADMIN_IP/32"] }
  ingress { from_port = 80;    to_port = 80;    protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 443;   to_port = 443;   protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 5060;  to_port = 5060;  protocol = "udp"; cidr_blocks = ["CARRIER_IP/32"] }
  ingress { from_port = 10000; to_port = 20000; protocol = "udp"; cidr_blocks = ["0.0.0.0/0"] }
  ingress { from_port = 8089;  to_port = 8089;  protocol = "tcp"; cidr_blocks = ["0.0.0.0/0"] }
  egress  { from_port = 0;     to_port = 0;     protocol = "-1";  cidr_blocks = ["0.0.0.0/0"] }
}

resource "aws_instance" "vicidial" {
  ami                    = "ami-XXXXXXXX" # AlmaLinux 9 or openSUSE Leap 15.6
  instance_type          = "c5.2xlarge"
  key_name               = "your-key-pair"
  subnet_id              = aws_subnet.public.id
  vpc_security_group_ids = [aws_security_group.vicidial.id]

  root_block_device {
    volume_type = "gp3"
    volume_size = 100
    iops        = 3000
  }
}

resource "aws_ebs_volume" "recordings" {
  availability_zone = "us-east-1a"
  size              = 500
  type              = "gp3"
  tags = { Name = "vicidial-recordings" }
}

resource "aws_volume_attachment" "recordings" {
  device_name = "/dev/xvdf"
  volume_id   = aws_ebs_volume.recordings.id
  instance_id = aws_instance.vicidial.id
}

resource "aws_eip" "vicidial" {
  instance = aws_instance.vicidial.id
  domain   = "vpc"
}
```

You will also need a route table associating the public subnet with the internet gateway. After `terraform apply`, SSH into the instance and follow the [VICIdial scratch install guide](/blog/vicidial-setup-guide/) for AlmaLinux 9, then apply the NAT configuration from the section above.

> **Cloud Deployment Without the Headaches.**
> ViciStack handles the infrastructure so you can focus on running campaigns. Bare metal performance, cloud-like simplicity. [Get a Custom Quote.](/pricing/)

---

## GCP Deployment

Google Cloud Platform offers comparable functionality to AWS with slightly different naming and a few VICIdial-relevant advantages.

### Instance Sizing

GCP's Compute Engine machine types map to AWS equivalents:

| Role | GCP Machine Type | Equivalent AWS | Monthly Cost |
|------|-----------------|---------------|-------------|
| Single server (<25 agents) | n2-standard-8 | c5.2xlarge | ~$260/month |
| DB server (25-100 agents) | n2-highmem-4 | m5.xlarge | ~$155/month |
| Telephony/dialer server | c2-standard-8 | c5.2xlarge | ~$275/month |
| Web server | n2-standard-4 | c5.xlarge | ~$135/month |

**GCP advantage: c2-standard-8.** The c2 series runs on dedicated 3.1 GHz Intel Cascade Lake processors with no CPU overcommit. This is as close to bare metal as you can get in a cloud VM, and it makes a measurable difference for Asterisk's real-time audio processing. If you are choosing between AWS c5 and GCP c2 specifically for VICIdial telephony, GCP's c2 has a slight edge in consistent CPU performance.

**GCP advantage: sustained use discounts.** Unlike AWS, where you need Reserved Instances or Savings Plans to get discounts, GCP automatically applies sustained use discounts when an instance runs more than 25% of the month. A c2-standard-8 running 24/7 gets a roughly 30% automatic discount. No commitment required.

### Storage

GCP Persistent Disks come in three relevant tiers:

- **pd-ssd**: SSD-backed, 30 IOPS/GB read, 30 IOPS/GB write. The equivalent of AWS gp3 with auto-scaling IOPS.
- **pd-balanced**: Lower-cost SSD option. Fine for recordings, not ideal for database.
- **pd-extreme**: Up to 120,000 IOPS. Overkill for VICIdial unless you have 300+ agents on one database.

Recommended configuration:

```bash
# Create instance with attached SSD persistent disks
gcloud compute instances create vicidial-server \
  --zone=us-central1-a \
  --machine-type=c2-standard-8 \
  --image-family=almalinux-9 \
  --image-project=almalinux-cloud \
  --boot-disk-size=100GB \
  --boot-disk-type=pd-ssd \
  --create-disk=name=vicidial-db,size=200GB,type=pd-ssd \
  --create-disk=name=vicidial-recordings,size=500GB,type=pd-balanced \
  --network-tier=PREMIUM \
  --tags=vicidial-server
```

### Firewall Rules

GCP uses network-level firewall rules instead of instance-level security groups. The tags-based approach is actually cleaner for VICIdial:

```bash
# SIP signaling from carrier
gcloud compute firewall-rules create vicidial-sip \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=udp:5060 \
  --source-ranges=CARRIER_IP_1/32,CARRIER_IP_2/32 \
  --target-tags=vicidial-server

# RTP media (must be open to all)
gcloud compute firewall-rules create vicidial-rtp \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=udp:10000-20000 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=vicidial-server

# HTTPS for agent interface
gcloud compute firewall-rules create vicidial-https \
  --direction=INGRESS \
  --priority=1000 \
  --network=default \
  --action=ALLOW \
  --rules=tcp:443,tcp:8089 \
  --source-ranges=0.0.0.0/0 \
  --target-tags=vicidial-server
```

### Static IP

```bash
# Reserve a static external IP
gcloud compute addresses create vicidial-ip --region=us-central1

# Assign to instance
gcloud compute instances add-access-config vicidial-server \
  --zone=us-central1-a \
  --access-config-name="External NAT" \
  --address=$(gcloud compute addresses describe vicidial-ip --region=us-central1 --format='value(address)')
```

Same rule as AWS: this IP is permanent. Do not release it.

---

## DigitalOcean Deployment

DigitalOcean is the simplest cloud option for VICIdial. Less flexibility than AWS or GCP, but dramatically less complexity. If you are running a small-to-medium operation (under 50 agents) and you want cloud without the AWS learning curve, DigitalOcean is the pragmatic choice.

### Droplet Sizing

| Role | Droplet Type | Specs | Monthly Cost |
|------|-------------|-------|-------------|
| Single server (<25 agents) | CPU-Optimized 8 vCPU | 8 vCPU, 16 GB RAM | $168/month |
| DB server | General Purpose 8 GB | 2 vCPU, 8 GB RAM | $68/month |
| Telephony server | CPU-Optimized 8 vCPU | 8 vCPU, 16 GB RAM | $168/month |

**Critical: Use CPU-Optimized Droplets.** DigitalOcean's Basic and General Purpose Droplets use shared vCPUs with burstable performance. CPU-Optimized Droplets provide dedicated vCPU cores, which is what Asterisk needs for consistent audio processing.

### Setup

```bash
# Create a CPU-Optimized Droplet via doctl CLI
doctl compute droplet create vicidial-server \
  --region nyc1 \
  --size c-8 \
  --image almalinux-9-x64 \
  --ssh-keys YOUR_SSH_KEY_FINGERPRINT \
  --wait

# Reserve a floating IP (DigitalOcean's equivalent of Elastic IP)
doctl compute floating-ip create --region nyc1

# Assign floating IP to droplet
doctl compute floating-ip-action assign FLOATING_IP DROPLET_ID
```

### Firewall

DigitalOcean Cloud Firewalls are simpler than AWS Security Groups:

```bash
doctl compute firewall create \
  --name vicidial-fw \
  --droplet-ids DROPLET_ID \
  --inbound-rules "protocol:tcp,ports:22,address:YOUR_ADMIN_IP/32 protocol:tcp,ports:80,address:0.0.0.0/0 protocol:tcp,ports:443,address:0.0.0.0/0 protocol:udp,ports:5060,address:CARRIER_IP/32 protocol:udp,ports:10000-20000,address:0.0.0.0/0 protocol:tcp,ports:8089,address:0.0.0.0/0" \
  --outbound-rules "protocol:tcp,ports:all,address:0.0.0.0/0 protocol:udp,ports:all,address:0.0.0.0/0"
```

### Managed Databases

DigitalOcean offers Managed MySQL databases, and this is genuinely tempting for VICIdial. Automated backups, failover, scaling --- everything you would normally do manually with MariaDB on a separate server.

**The catch:** VICIdial uses MEMORY tables (`vicidial_auto_calls`, `vicidial_hopper`, and several others defined in the [cluster configuration](/blog/vicidial-cluster-guide/)). MEMORY tables do not persist across restarts and require the `MEMORY` storage engine to be available. Most managed database services restrict which storage engines you can use. DigitalOcean's managed MySQL **does** support the MEMORY engine, but you need to verify this explicitly for your specific plan tier before committing.

Additionally, managed databases add 1-3ms of network latency compared to a local MySQL instance. For the [hopper](/glossary/hopper/) fill process, which runs every second and queries the `vicidial_list` table, this additional latency can reduce hopper efficiency at high agent counts.

**Our recommendation:** Use managed databases only if you are running fewer than 50 agents and prioritize operational simplicity over raw performance. For larger operations, run MariaDB on a dedicated Droplet within the same VPC.

---

## Cloud Architecture Patterns

The right architecture depends entirely on your agent count and growth trajectory. Here are the three patterns we see in production.

### Pattern 1: Single Server (Under 25 Agents)

```
┌──────────────────────────┐
│   Cloud Instance          │
│   (c5.2xlarge / c2-8)    │
│                           │
│   ┌─────────────────┐    │
│   │ Asterisk 18     │    │
│   │ Apache/PHP      │    │
│   │ MariaDB         │    │
│   │ VICIdial Screens│    │
│   └─────────────────┘    │
│                           │
│   Elastic/Static IP       │
└──────────────────────────┘
         │
         ▼
    SIP Carrier
```

This is the [ViciBox Express](/blog/vicidial-setup-guide/) equivalent in the cloud. Everything on one box. Simple, cheap, and perfectly adequate for small operations.

**When to use:** Startup operations, testing environments, or stable small teams that will not grow past 25 outbound agents.

**Estimated monthly cost:** $200-350 depending on provider and instance type.

### Pattern 2: Web + DB + Dialer Split (25-100 Agents)

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│  Web Server   │   │  DB Server    │   │ Telephony    │
│  (c5.xlarge)  │   │  (m5.xlarge)  │   │ (c5.2xlarge) │
│               │   │               │   │              │
│  Apache/PHP   │   │  MariaDB      │   │ Asterisk 18  │
│  Agent screens│   │  Lead data    │   │ SIP trunks   │
│  Reports      │   │  CDR/logs     │   │ Recordings   │
└──────┬───────┘   └──────┬───────┘   └──────┬───────┘
       │                   │                   │
       └───────────┬───────┘                   │
                   │         VPC               │
                   └───────────────────────────┘
                              │
                              ▼
                         SIP Carrier
```

This is the sweet spot for cloud VICIdial. Separating the database from the telephony server eliminates I/O contention between MariaDB queries and Asterisk recording writes. The web server handles agent screen rendering without competing for CPU with call processing.

**When to use:** Growing operations that have outgrown a single server but do not need full cluster redundancy.

**Estimated monthly cost:** $500-800 depending on provider and storage requirements.

### Pattern 3: Full Cluster (100+ Agents)

```
┌────────────┐  ┌────────────┐
│ Web Server  │  │ Web Server  │
│ (redundant) │  │ (redundant) │
└──────┬─────┘  └──────┬─────┘
       │               │
       └───────┬───────┘
               │
        ┌──────┴──────┐
        │  DB Server   │
        │  (primary)   │─── DB Replica (read)
        └──────┬──────┘
               │
    ┌──────────┼──────────┐
    │          │          │
┌───┴───┐ ┌───┴───┐ ┌───┴───┐
│ Dialer │ │ Dialer │ │ Dialer │
│ Svr 1  │ │ Svr 2  │ │ Svr 3  │
└────────┘ └────────┘ └────────┘
```

For the full details on [VICIdial cluster configuration](/blog/vicidial-cluster-guide/), see our dedicated guide. In a cloud context, the key additions are:

- **Placement groups** (AWS) or **sole-tenant nodes** (GCP) to ensure telephony servers run on different physical hosts
- **Cross-AZ deployment** for database redundancy
- **Auto Scaling Groups** for web servers (agent screens are stateless)
- **Application Load Balancer** in front of web servers for HTTPS termination

**Estimated monthly cost:** $1,200-2,500+ depending on agent count and redundancy level.

---

## Cost Comparison: Cloud vs. Bare Metal vs. Managed Hosting

Here is the honest cost breakdown for a 50-agent outbound operation running 8 hours/day, 5 days/week.

| Cost Component | AWS (us-east-1) | Bare Metal (colo) | ViciStack Managed |
|---------------|-----------------|-------------------|-------------------|
| Compute (web + DB + telephony) | $600-750/month | $200-350/month | Included |
| Storage (500 GB SSD + recordings) | $80-120/month | Included in server | Included |
| Data transfer (SIP + recordings) | $50-100/month | Included in colo | Included |
| Elastic IP / static IP | $3.65/month | Included | Included |
| Backups (EBS snapshots) | $30-50/month | Manual/custom | Included |
| **Total infrastructure** | **$764-1,024/month** | **$200-350/month** | **Custom quote** |

The cloud premium is 2-4x over bare metal for equivalent performance. That premium buys you elasticity, managed hardware, and geographic flexibility. Whether that is worth it depends on your operational profile.

**Where cloud wins on cost:** Operations that scale up/down seasonally. If you need 150 agents for 3 months and 30 agents the rest of the year, cloud reserved instances for the baseline plus on-demand for the surge is cheaper than owning 150-agent-capacity hardware year-round.

**Where bare metal wins on cost:** Steady-state operations. If your agent count is stable month over month, dedicated servers from OVH, Hetzner, or a traditional colo are dramatically cheaper and often faster due to dedicated CPU cores and local storage.

> **Get the Best of Both Worlds.**
> ViciStack delivers bare-metal performance at predictable monthly pricing. No cloud markup, no hardware management. [See Pricing.](/pricing/)

---

## Backup Strategies for Cloud VICIdial

Cloud backup is one area where cloud deployment genuinely shines over bare metal. Here are the strategies, ordered by importance.

### Database Backups

The VICIdial database is your most critical asset. Losing it means losing all campaign configurations, lead data, CDR history, and agent settings.

**EBS Snapshots (AWS):**

```bash
# Automated daily snapshot of the database volume
aws ec2 create-snapshot \
  --volume-id vol-0abc123def456 \
  --description "VICIdial DB backup $(date +%Y-%m-%d)" \
  --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=vicidial-db-daily}]'
```

Schedule this with a cron job or AWS Backup. Retain 30 days of daily snapshots and 12 months of monthly snapshots.

**mysqldump for logical backups:**

```bash
# Daily logical backup (complements EBS snapshots)
mysqldump --single-transaction --routines --triggers \
  --databases asterisk \
  | gzip > /backup/vicidial-db-$(date +%Y%m%d).sql.gz
```

Logical backups are portable across providers and allow granular table-level restores. EBS snapshots are faster for full restores. Use both.

### Recording Archival to S3

Call [recordings](/glossary/recording/) are the largest storage consumer and the most expensive to keep on high-performance EBS. Move them to S3 after they age past your active access window.

```bash
#!/bin/bash
# archive_recordings.sh - Move recordings older than 90 days to S3
ARCHIVE_DATE=$(date -d "-90 days" +%Y%m%d)
RECORDING_DIR="/var/spool/asterisk/monitor"
S3_BUCKET="s3://your-vicidial-recordings"

find ${RECORDING_DIR} -name "*.wav" -mtime +90 -exec \
  aws s3 mv {} ${S3_BUCKET}/$(date -d @$(stat -c %Y {}) +%Y/%m)/ \;
```

S3 Standard costs $0.023/GB/month. S3 Glacier Instant Retrieval costs $0.004/GB/month. For a 50-agent operation generating 300 GB/month of recordings, moving to Glacier after 90 days saves roughly $70/month versus keeping everything on EBS.

**S3 Lifecycle Rule:**

```json
{
  "Rules": [
    {
      "ID": "recordings-lifecycle",
      "Status": "Enabled",
      "Transitions": [
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ]
    }
  ]
}
```

### Configuration Backups

VICIdial's configuration lives in multiple places: the database, Asterisk config files, Apache configs, and cron jobs. Back up all of them:

```bash
#!/bin/bash
# backup_config.sh
BACKUP_DIR="/backup/config/$(date +%Y%m%d)"
mkdir -p ${BACKUP_DIR}

# Asterisk configs
tar czf ${BACKUP_DIR}/asterisk-config.tar.gz /etc/asterisk/

# VICIdial web configs
tar czf ${BACKUP_DIR}/vicidial-web.tar.gz /srv/www/htdocs/agc/ 2>/dev/null || \
tar czf ${BACKUP_DIR}/vicidial-web.tar.gz /var/www/html/agc/ 2>/dev/null

# Crontab
crontab -l > ${BACKUP_DIR}/crontab.txt

# System configs
tar czf ${BACKUP_DIR}/system-config.tar.gz /etc/my.cnf.d/ /etc/httpd/ /etc/apache2/

# Upload to S3
aws s3 sync ${BACKUP_DIR} s3://your-vicidial-backups/config/$(date +%Y%m%d)/
```

---

## Monitoring Cloud VICIdial

Running VICIdial in the cloud gives you access to native monitoring tools that bare metal does not have. Use them.

### CloudWatch (AWS)

Key metrics to alarm on:

| Metric | Threshold | What It Means |
|--------|-----------|--------------|
| CPUUtilization | >80% sustained 5 min | Asterisk is CPU-starved, audio quality degrading |
| EBSWriteLatency (DB volume) | >10ms | Database queries slowing, hopper fill impacted |
| NetworkIn/Out | Sudden drop | Network partition, carrier connectivity lost |
| StatusCheckFailed | Any | Instance hardware failure, immediate action needed |

```bash
# Create CloudWatch alarm for CPU
aws cloudwatch put-metric-alarm \
  --alarm-name "VICIdial-HighCPU" \
  --metric-name CPUUtilization \
  --namespace AWS/EC2 \
  --statistic Average \
  --period 300 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=InstanceId,Value=i-0abc123def456 \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:123456789:vicidial-alerts
```

### Grafana + Prometheus

For VICIdial-specific metrics that CloudWatch cannot see (agent talk time, hopper level, trunk utilization), deploy Grafana with a MySQL data source pointed at your VICIdial database:

```sql
-- Grafana query: Active calls right now
SELECT COUNT(*) as active_calls
FROM vicidial_auto_calls
WHERE status IN ('LIVE', 'XFER', 'CLOSER');

-- Grafana query: Hopper level by campaign
SELECT campaign_id, COUNT(*) as hopper_count
FROM vicidial_hopper
GROUP BY campaign_id;

-- Grafana query: Agent status distribution
SELECT status, COUNT(*) as agent_count
FROM vicidial_live_agents
GROUP BY status;
```

This gives you a real-time dashboard that combines infrastructure metrics (CPU, disk, network) with VICIdial operational metrics (agent utilization, hopper health, trunk capacity).

---

## Post-Deployment Checklist

After your cloud VICIdial instance is running, verify these items before putting agents on it:

- [ ] Elastic/static IP assigned and confirmed in Asterisk `externip`
- [ ] SIP registration successful (check `asterisk -rx "sip show peers"` or `pjsip show endpoints`)
- [ ] Two-way audio confirmed on test calls (call your cell phone, verify audio both directions)
- [ ] [RTP ports](/glossary/rtp/) open and receiving media (check with `tcpdump -i eth0 udp portrange 10000-20000`)
- [ ] Database accessible from all servers in the cluster (test with `mysql -h DB_IP -u cron -p`)
- [ ] Recording files being written (place a test call, check `/var/spool/asterisk/monitor/`)
- [ ] Agent login works via HTTPS (test with the default 6666/1234 credentials, then change them)
- [ ] Time synchronization configured (NTP/chrony --- critical for CDR accuracy and [callback scheduling](/glossary/callback/))
- [ ] Backups tested with actual restore (a backup you have not tested is not a backup)
- [ ] Monitoring alerts configured and verified (send a test alert)

---

## Frequently Asked Questions

### Can I run VICIdial on AWS Free Tier?

No. The t2.micro/t3.micro instances in the free tier have 1 vCPU and 1 GB RAM. VICIdial requires a minimum of 4 cores and 8 GB RAM for even a small deployment. Attempting to run Asterisk on a t2.micro will result in immediate resource exhaustion. You need at least a c5.xlarge for a minimal test environment.

### Is AWS Lightsail suitable for VICIdial?

Lightsail is essentially DigitalOcean running on AWS infrastructure. The $40/month 8 GB plan could technically run a tiny VICIdial instance for testing, but Lightsail uses shared vCPUs and has limited networking options. For production, use proper EC2 instances.

### How do I handle SIP registration failures in the cloud?

The most common cause is incorrect security group configuration. SIP uses UDP 5060 for signaling, and many carriers also expect the server to respond on the same port. Verify that UDP 5060 is open inbound from your carrier's IP ranges, and that outbound UDP is unrestricted. Also check that your Asterisk `externip` matches your Elastic IP exactly.

### Should I use a VPN between cloud VICIdial servers?

For multi-server clusters within the same VPC, no. The VPC provides network isolation and encryption is unnecessary for intra-cluster traffic. For cross-region clusters or hybrid cloud/on-prem setups, yes --- use WireGuard or AWS Site-to-Site VPN. [IAX2](/glossary/iax2/) inter-server traffic should be encrypted or tunneled if it crosses the public internet.

### Can I use Docker/Kubernetes for VICIdial?

You can, but you probably should not. Asterisk requires direct access to UDP ports, consistent CPU scheduling, and a persistent filesystem for recordings. Containerization adds networking complexity (SIP NAT is already hard enough without Docker's bridge networking adding another layer) and provides minimal benefit for a stateful telephony application. VICIdial is not a microservice --- it is a monolithic system that benefits from running directly on the OS.

### How do I migrate from bare metal to cloud?

The process is: (1) provision cloud infrastructure using this guide, (2) do a fresh VICIdial install on the cloud server, (3) export your database from bare metal with `mysqldump`, (4) import on the cloud server, (5) update Asterisk configs with new IP/NAT settings, (6) re-register SIP trunks with your carrier using the new IP, (7) migrate recordings via rsync or S3 upload. Plan for 2-4 hours of downtime for a clean cutover, or run parallel systems with a DNS-based switchover for zero-downtime migration.

### What about Azure for VICIdial?

Azure works, but it is less commonly used for VICIdial than AWS or GCP. The equivalent instance types are Fsv2 (compute-optimized) and Esv5 (memory-optimized). Azure's networking is more complex than AWS or GCP, and the SIP NAT traversal configuration requires additional steps with Azure's NAT Gateway. Unless your organization is already committed to Azure, AWS or GCP are simpler paths.

### How much bandwidth does VICIdial use in the cloud?

A single concurrent call using G.711 (the standard [codec](/glossary/codec/) for VICIdial) consumes approximately 87 kbps in each direction. For 50 concurrent calls, that is roughly 8.7 Mbps sustained. Add agent screen traffic (minimal), database replication (if clustered), and recording uploads, and a 50-agent operation typically uses 15-25 Mbps sustained. Cloud data transfer costs apply: AWS charges $0.09/GB for outbound data after the first 100 GB/month. For a 50-agent operation, expect $50-100/month in data transfer charges.

---

## About This Guide

This guide was originally published at [ViciStack.com](https://vicistack.com/blog/vicidial-cloud-deployment).

For more VICIdial and Asterisk guides, visit [vicistack.com](https://vicistack.com).

**Related Resources:**

- [Free VICIdial Audit](https://vicistack.com/free-audit/) — We review your configuration
- [ViciStack Pricing](https://vicistack.com/pricing/) — Managed VICIdial services
- [All ViciStack Guides](https://vicistack.com/blog/) — 70+ production-tested articles
