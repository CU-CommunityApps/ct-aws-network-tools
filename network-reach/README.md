# network-reach (ct-aws-network-tools)

Analyze network connectivity between a source in AWS to some destination

## Arguments

argument (short) | argument (long) | description | example
---- | ---- | ---- | ----
`-v` | `--verbose` | print extra info | n/a
`-s` | `--source-ec2-ip` | private IPv4 address of an EC2 instance | 10.92.76.45
`-r` | `--source-rds` | name of RDS instance | employee-db-prod
`-m` | `--source-dms` | name of DMS instance | replication-instances-1
`-d` | `--dest-ip`| (required) private or public IP address of destination | 172.217.12.164

One of `source-ec2-ip`, `source-rds`, `source-dms` must be provided. This tool assumes that the source AWS resource (RDS instance, DMS instance, or private IP address) resides in the region and the AWS account for which AWS credentials are configured.

The `dest-ip` must always be provided and can be a private or a public IPv4 address. If the `dest-ip` is in AWS and has both a private IP and a public IP address, you might want to run the tool twice, once with each, if you are trying to understand how network traffic might be flowing between the source and desination.

## Examples

```
$ ./reach.py --source-ec2-ip 10.92.77.247 --dest-ip 10.146.129.72
```

```
$ ./reach.py --source-ec2-ip 10.92.77.247 --dest-ip 172.217.9.228
```

```
$ ./reach.py --source-rds billing-detail --dest-ip 10.92.77.247
```

```
$ ./reach.py --source-dms replication-instance-1 --dest-ip 10.130.130.200`
```

## Tips

If both the source and destination are resources in AWS, you will get the most information by running the tool twice, swapping the source and destination on the second run. Each time the tool is run, it analyzes in the source-to-destination direction (in general).

This tool does not handle ALL networking situations. It is meant to gather basic information, present it, and flag obvious issues. Suggestions for improvements are welcome.

## Prerequisites

- [Python3](https://www.python.org/downloads/)
- [boto3](https://github.com/boto/boto3)
- [AWS credentials](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html)
  - IAM privileges must grant `ec2:Describe*`, `rds:Describe*`, `dms:Describe*` at minimum.
- [AWS SDK environment variables](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html)
  - Depending on your AWS credentials and CLI configuration, you may need to set `AWS_DEFAULT_PROFILE`, `AWS_DEFAULT_REGION`, or others

## Specific Tests

### All Sources
- Security Groups
  - PROBLEM when security groups applied to source do not appear to allow any traffic to/from the destination IP. Since this tool is not analyzing traffic over specific ports, passing this test does not gaurantee that the traffic you care about is able to flow on the ports you care about. E.g., if a source security group has any allow 0.0.0.0/0 rules, this test will always pass.
- Route Tables
  - PROBLEM if route to destination involves an Internet Gateway that is not "available" or attached.
  - PROBLEM if route to destination involves a Virtual Gateway that is not "avalable" or attached.
  - PROBLEM if route to destination involves a Virtual Gateway that does not have a Direct Connect Virtual Interface attached. (Thus, this tool will flag a problem when a VPC is using a VPN or a Direct Connect Gateway.)
  - PROBLEM if route to destination involves a Virtual Gateway with a Direct Connect Virtual Interface attached, but the Virtual Interface is not "available", or there are problems with BGP.
  - PROBLEM if route to destination involves a NAT with state other than "available".
  - PROBLEM if route to destination involves a VPC peering connection and the peering connection has status other than "active".
  - PROBLEM if route to destination involves some other type of route table target (i.e., not an Internet Gateway, Virtual Gateway, NAT, or peering connection). This may not be a true problem; it's more of a flag to note that the route hasn't been analyzed fully.
- Network ACL
  - PROBLEM when Network ACL applied to the source subnet does not appear to allow any traffic to/from the destination IP. Like the Security Group analysis, passing this test does not gaurantee that the traffic you care about is able to flow on the ports you care about. This tool is not analyzing traffic over specific ports.

### RDS Sources
- DB Subnet Group
  - WARNING when subnets in DB Subnet Group used by RDS instance do not use the same route table.
- Public RDS Instance
  - WARNING when RDS instance is public but destination address is private.

### DMS Sources
- DB Subnet Group
  - WARNING when subnets in DB Subnet Group used by RDS instance do not use the same route table.

## Future Enhancements

- [x] Gather/analyze information about Direct Connect and Internet Gateways
- [x] Gather/analyze information about Database Migration Service instances
- Specify source/destination port as an input parameter and scope analsys to that port
- Handle ELB/ALB/NLB as a "source"
- [X] Update this README with a list of specific checks that are made (and that can fail)
- Handle non-routable CIDR blocks in Route Table analysis. E.g., traffic to 10.1.2.3 won't be routed through a NAT Gateway.