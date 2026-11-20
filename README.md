# Shared File System using AWS EFS

A capstone project that replaces an on-premises NAS with a **highly available, encrypted, shared file system** built on **Amazon EFS** across two Availability Zones in a custom VPC.

![Architecture](images/architecture.png)

## Problem

An entertainment production company using on-premises NAS faced:

- Limited scalability
- High capital cost for hardware expansion
- No encryption at rest or in transit
- Manual effort to move rarely used media to cheaper storage

## Solution

| Requirement | Implementation |
|---|---|
| Scalability | Regional Amazon EFS with Elastic throughput |
| Cost | Lifecycle Management moves files to Standard-IA after 30 days |
| Encryption at rest | Customer-managed AWS KMS key (`efs-project-key`) |
| Encryption in transit | TLS mount using `amazon-efs-utils` and `stunnel` |
| Availability | Mount targets in `us-east-1a` and `us-east-1b`, automatic backups |
| Secure admin access | Private instance managed through AWS Systems Manager Session Manager (no public IP) |

## AWS Services

Amazon VPC, Amazon EFS, AWS KMS, Amazon EC2, AWS IAM, AWS Systems Manager, NAT Gateway, VPC Interface Endpoints.

## Deployed Resources

| Resource | Name | Details |
|---|---|---|
| VPC | `EFS-Project-vpc` | `10.0.0.0/16`, DNS hostnames enabled, 4 subnets in 2 AZs |
| KMS key | `efs-project-key` | Symmetric, encrypt/decrypt |
| EFS | `EFS-Shared-Storage` | Regional, KMS encrypted, General Purpose, Elastic throughput, backups on, IA after 30 days |
| EC2 | `FIRSTEC2` | t2.micro, Amazon Linux 2023, public subnet, `us-east-1a` |
| EC2 | `SECONDEC2` | t2.micro, Amazon Linux 2023, private subnet, `us-east-1b` |
| Security groups | `EFS-Project-EFS-SG`, `EFS-Project-EC2-SG`, `EFS-Project-SSM-Endpoint-SG` | NFS 2049, SSH 22 + ICMP, HTTPS 443 |
| IAM role | `EFS-Project-SSM-Role` | Session Manager access for `SECONDEC2` |
| VPC endpoints | `ssm`, `ssmmessages` | Interface endpoints |
| NAT gateway | `EFS-Temp-NAT` | Temporary, used to install packages on the private instance |

## Implementation Phases

1. **Networking** - custom VPC, public/private subnets in two AZs, Internet Gateway, route tables.
2. **Encryption and EFS** - KMS key, EFS file system, NFS security group.
3. **Compute and private access** - two EC2 instances, SSM role, VPC endpoints, NAT gateway.
4. **TLS mount and verification** - mount on both instances and test file sharing across AZs.

## Mount Commands

```bash
# FIRSTEC2 (us-east-1a)
mkdir -p /home/ec2-user/efs
mount -t efs -o tls fs-005c1285acefadaab:/ /home/ec2-user/efs
echo "Shared EFS Storage Test from FIRSTEC2" > /home/ec2-user/efs/shared-test.txt

# SECONDEC2 (us-east-1b) via Session Manager
mkdir -p /home/ssm-user/efs
sudo mount -t efs -o tls fs-005c1285acefadaab:/ /home/ssm-user/efs
echo "EFS connection successful" > /home/ssm-user/efs/efs-test.txt
cat /home/ssm-user/efs/shared-test.txt
```

> Use the full absolute path for the mount point. A relative path caused the error `mount point efs does not exist`.

## Verification Results

| Test | Result |
|---|---|
| Ping `FIRSTEC2` to `SECONDEC2` | 0% packet loss |
| TLS mount on both instances | Success (shown as `127.0.0.1:/` through the local stunnel proxy) |
| File written in AZ-a, read in AZ-b | Success |
| File written from AZ-b | Success |

![Cross-AZ test](images/screenshots/22-secondec2-mount-and-read.jpg)

All screenshots are in the [`images/screenshots`](images/screenshots) folder.

## Security Recommendations

- Limit the EFS security group source to the EC2 security group.
- Limit SSH to your own IP address.
- Add an `ec2messages` endpoint if the NAT gateway is removed.
- Use `/etc/fstab` (with `tls,_netdev`) for automatic remount after reboot.
- Never commit key pairs (`.pem`) or account details to the repository.

## Cleanup

NAT gateways and interface endpoints are billed hourly. After testing: terminate the EC2 instances, delete the mount targets and file system, delete the NAT gateway and release its Elastic IP, delete the VPC endpoints, then the security groups, subnets, route tables, Internet Gateway and VPC. Schedule deletion of the KMS key.

## Full Documentation

See the [`docs`](docs) folder for the complete PDF and Word documentation.

## Author

**Ahmed Ali** - AWS Cloud capstone project, November 2026.
