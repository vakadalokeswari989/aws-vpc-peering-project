# AWS VPC Peering Project

**Name:** Lokeswari V | **Batch:** Batch 1 | **Region:** ap-south-1 (Mumbai) | **Date:** 30-10-2026

Full report: [Lokeswari_EWB_VPC_Peering.pdf](Lokeswari_EWB_VPC_Peering.pdf)

## Overview
Two VPCs are connected with VPC peering so that two private EC2 instances can talk to each other using private IPs only. One public bastion host in VPC A is the only entry point; the private servers are reached with SSH ProxyJump. No NAT gateway is used and the private servers have no public IP.

## Architecture
![Architecture](screenshots/architecture.png)

## IP Plan and Resource Inventory
| Resource | CIDR / IP | ID |
|---|---|---|
| ewb-vpc-a | 10.10.0.0/16 | vpc-0167a2323ec7f602f |
| ewb-a-public | 10.10.1.0/24 | subnet-078b288a888f4a590 |
| ewb-a-private | 10.10.2.0/24 | subnet-016a19ca45ba6c444 |
| ewb-vpc-b | 10.20.0.0/16 | vpc-0322851fee12de5a4 |
| ewb-b-private | 10.20.1.0/24 | subnet-0428a86f733844f2d |
| ewb-igw-a | attached to VPC A | igw-078c244219093f26d |
| ewb-rt-a-public | - | rtb-0a126363e86b584b1 |
| ewb-rt-a-private | - | rtb-07257232a0b38fdeb |
| ewb-rt-b-private | - | rtb-04c369eee658bb812 |
| ewb-a-b-peer | VPC A to VPC B | pcx-0d24fdabbdddcfe91 |
| ewb-bastion | 10.10.1.10 + public IP | i-0ded36efc198c8e1a |
| ewb-app-a | 10.10.2.10 (no public IP) | i-06c909174aed02f8c |
| ewb-app-b | 10.20.1.10 (no public IP) | i-0eb16c46963e015bc |

All instances: t3.micro, Amazon Linux 2023 (`ami-03765515c80e0c765`).

![VPCs](screenshots/01-vpc-list.png)
![Subnets](screenshots/02-subnet-list.png)
![Instances](screenshots/07-cp02-instances-running.png)

## Route Tables
Before peering (Checkpoint 01):

![](screenshots/03-cp01-rt-a-public-routes.png)
![](screenshots/04-cp01-rt-a-public-assoc.png)
![](screenshots/05-cp01-rt-a-private-routes.png)
![](screenshots/06-cp01-rt-a-private-assoc.png)

After peering (Checkpoint 04), one route added to each table:

| Route table | Added route |
|---|---|
| ewb-rt-a-private | 10.20.0.0/16 to pcx-0d24fdabbdddcfe91 |
| ewb-rt-b-private | 10.10.0.0/16 to pcx-0d24fdabbdddcfe91 |
| ewb-rt-a-public | 10.20.0.0/16 to pcx-0d24fdabbdddcfe91 |

![](screenshots/13-cp03-peering-active.png)
![](screenshots/14-cp04-rt-a-private-routes.png)
![](screenshots/15-cp04-rt-b-private-routes.png)
![](screenshots/16-cp04-rt-a-public-routes.png)
![](screenshots/17-cp04-rt-b-private-assoc.png)
![](screenshots/18-cp04-rt-a-private-assoc.png)
![](screenshots/19-cp04-rt-a-public-assoc.png)

## Security Group Rules
| Security group | Type / port | Source |
|---|---|---|
| ewb-sg-bastion | SSH 22 | my public IP /32 |
| ewb-sg-app-a | SSH 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-a | ICMP echo request | 10.20.1.10/32 (App-B) |
| ewb-sg-app-b | SSH 22 | 10.10.1.10/32 (bastion) |
| ewb-sg-app-b | ICMP echo request | 10.10.2.10/32 (App-A) |

![](screenshots/11-sg-bastion-inbound-rules.png)
![](screenshots/12-sg-app-b-inbound-rules.png)

## Test Results
1. **Baseline (before peering):** ping from App-A to App-B failed, 100% packet loss.

   ![](screenshots/20-test01-baseline-ping-fail.png)
2. **Forward ping App-A to App-B:** 4 received, 0% loss.

   ![](screenshots/21-test02-forward-ping-success.png)
3. **Reverse ping App-B to App-A and SSH to App-B through the bastion:** 4 received, 0% loss.

   ![](screenshots/22-test03-04-ssh-app-b-and-reverse-ping.png)
4. **No public IP on App-A and App-B:**

   ![](screenshots/09-app-a-no-public-ip.png)
   ![](screenshots/10-app-b-no-public-ip.png)
5. **Controlled failure (ICMP rule removed) and recovery:** 100% loss, then 0% loss after restoring the rule.

   ![](screenshots/23-test06-controlled-failure-and-recovery.png)
   ![](screenshots/24-sg-app-b-after-rule-change.png)

## Explanation
- **Forward path:** App-A (10.10.2.10) sends a ping to 10.20.1.10. The route 10.20.0.0/16 to the peering connection in `ewb-rt-a-private` sends it into VPC B, and the security group of App-B allows ICMP echo request from 10.10.2.10/32.
- **Return path:** App-B replies to 10.10.2.10 using the route 10.10.0.0/16 to the peering connection in `ewb-rt-b-private`. Security groups are stateful, so the reply is allowed automatically.
- **Why the failure happened and how restoring the rule fixed it:** with the ICMP rule removed from `ewb-sg-app-b`, packets still had a route but App-B dropped them, so there was no reply. Adding the same rule back allowed the echo request again and the ping recovered.
- **Why the bastion needs a peering route:** with ProxyJump the bastion opens its own SSH connection to App-B, so its subnet must know how to reach 10.20.0.0/16.

## Cleanup
Cleanup evidence will be added here after all resources are deleted.

## What I Learned
- A peering connection only creates the link. Both route tables need routes, and the security groups must allow the traffic.
- A baseline failure test proves the later success comes from my configuration.
- Private servers can be managed through a bastion with SSH ProxyJump without copying the private key onto any instance.
- Peering needs non-overlapping CIDRs and is not transitive, so VPC B cannot use VPC A's internet gateway.
