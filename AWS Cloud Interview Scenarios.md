# AWS Cloud Interview Scenarios: Basic to Critical Thinking

This interview preparation guide is designed for AWS Cloud Engineer, Junior DevOps Engineer, Cloud Support Engineer, and L1/L2 Infrastructure Support roles. It focuses on real-world incidents, logical troubleshooting, root-cause analysis, and interview-ready answers rather than just definitions.

The goal is to help you answer questions like: “Your application is down in production. What will you check first, and how will you fix it without affecting customers?”

## How to answer any AWS troubleshooting scenario

Use this five-step framework in interviews:

1\. Assess the impact

Is the issue affecting one user, one server, one Availability Zone, or the entire application?

2\. Gather evidence

Check CloudWatch metrics and logs, application logs, health checks, recent changes, and AWS service events.

3\. Isolate the layer

Identify whether the problem is DNS, networking, security, compute, application, database, or permissions.

4\. Restore service safely

Apply the smallest safe fix, preserve evidence, validate recovery, and avoid unnecessary production changes.

5\. Prevent recurrence

Document the root cause, add monitoring and alerts, automate recovery where appropriate, and create a runbook.

## Level 1 — Basic AWS scenarios

These test your fundamentals and are common in fresher and junior cloud interviews.

Scenario 1

EC2 · Connectivity

Your EC2 instance is running, but you cannot connect through SSH. What will you check?

Interview-ready answer:

“First, I will confirm the instance state, status checks, public IP, and subnet routing. Then I will verify that the security group allows inbound TCP port 22 from my authorized public IP. I will check the network ACL, internet gateway route, operating system firewall, and SSH service. I will also confirm that I am using the correct username and private key.”

Troubleshooting order:

1. EC2 instance and system status checks.
2. Public IP or reachable private network path.
3. Security group inbound rule.
4. Route table and internet gateway, if using public IPv4.
5. Network ACL and OS firewall.
6. SSH daemon, username, key permissions, and authentication logs.

Critical point: A public IP alone does not make an instance reachable. For private instances, use an approved path such as Systems Manager Session Manager, a bastion host, or VPN.

Scenario 2

S3 · Access

An S3 bucket exists, but a user receives `AccessDenied`. Why?

Interview-ready answer:

“I will identify the exact API operation and principal, then evaluate the IAM identity policy, bucket policy, permission boundaries, session policies, and organization-level SCPs. I will also check explicit denies, KMS permissions if the object is encrypted, and whether the request uses the correct account and object key.”

Key distinctions:

- `ListBucket` permission applies to listing bucket contents.
- `GetObject` permission applies to reading objects.
- An explicit deny overrides an allow.
- Cross-account access generally requires appropriate authorization on both sides.
- I will use CloudTrail and policy simulation where applicable.

Avoid: Making the bucket public just to resolve an access issue.

Scenario 3

CloudWatch · Monitoring

Your EC2 CPU utilization reaches 95%. What do you do?

Interview-ready answer:

“I will check whether CPU usage is sustained or a temporary spike. I will correlate it with application response times, traffic, memory, disk I/O, and recent deployments. On Linux, I can use `top`, `ps`, and `vmstat` to identify the process consuming CPU. Based on the root cause, I may optimize the process, scale out behind a load balancer, or resize the instance.”

Commands:

```
top
ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head
uptime
free -h
vmstat 1 5
```

Critical point: CPU above 90% is a symptom, not the root cause. Check traffic, runaway processes, and application errors before scaling.

Scenario 4

IAM · Permissions

An EC2 application needs to access S3. Would you store AWS access keys in the application code?

Interview-ready answer:

“No. I will attach an IAM role to the EC2 instance through an instance profile and grant only the required S3 permissions. The AWS SDK can obtain temporary credentials automatically. I will verify the role, trust relationship, policy scope, and any explicit denies.”

Example principle: grant access to a specific bucket and required object prefix rather than `s3:*` on all resources.

Scenario 5

EBS · Storage

Your Linux EC2 server reports “No space left on device.” What will you check?

Interview-ready answer:

“I will check filesystem capacity and inode usage first. Then I will locate large directories and log files, verify whether deleted files are still held open by processes, and check application logs. If capacity is genuinely insufficient, I will expand the EBS volume and filesystem following a safe change procedure.”

```
df -h
df -i
sudo du -xhd1 /var /home 2>/dev/null
lsblk
```

Critical point: `df -h` can show free space while `df -i` shows 100% inode usage. These are different problems.

## Level 2 — Intermediate AWS troubleshooting

These scenarios test whether you can connect multiple AWS services and follow a logical troubleshooting sequence.

Scenario 6

VPC · Internet access

An EC2 instance has a public IP but cannot access the internet. What could be wrong?

Answer:

“I will check the subnet's route table for a default route to an internet gateway, verify that the internet gateway is attached to the VPC, and check outbound security group rules and network ACLs. I will also verify that the OS firewall and DNS configuration are working.”

Check the route:

```
Destination       Target
10.0.0.0/16       local
0.0.0.0/0         igw-xxxxxxxx
```

This is an example of a public-subnet route table. The instance also needs a suitable public IPv4 address and applicable network rules.

Follow-up: If it is a private subnet, the usual IPv4 internet-egress design uses a NAT gateway in a public subnet, with a route from the private subnet to that NAT gateway. A NAT gateway does not provide unsolicited inbound internet connectivity.

Scenario 7

ALB · HTTP errors

Your website returns HTTP 502 through an Application Load Balancer. How do you troubleshoot it?

Answer:

“I will check the target group's health, ALB access logs, target response behavior, and application logs. I will verify that the target group uses the correct protocol and port, the targets accept traffic from the ALB security group, and the application is listening on the expected interface and port.”

Investigate in this order:

1. ALB listener and listener rules.
2. Target group health and health-check path.
3. ALB-to-target security group rules.
4. Application process and listening port.
5. Backend protocol, TLS configuration, and application logs.

Important distinction: A 502 commonly indicates an invalid or failed response from a target or a connection problem. A 503 can indicate that the ALB has no usable registered targets. A 504 often points to a timeout while waiting for a target response. Confirm the cause using logs and metrics rather than relying only on the status code.

Scenario 8

RDS · Connectivity

Your application server cannot connect to an RDS MySQL database. What will you check?

Answer:

“I will confirm the database endpoint, port, status, and network path. I will verify that the RDS security group allows inbound TCP 3306 from the application's security group. Then I will check subnet routing, network ACLs, DNS resolution, database credentials, and MySQL grants.”

From the application server:

```
nslookup your-db-endpoint
nc -vz your-db-endpoint 3306
```

Interpret the evidence:

- Timeout: Often network reachability, routing, or filtering.
- Connection refused: The destination is reachable, but the connection is not accepted at that port or endpoint.
- Access denied: Usually credentials, authentication configuration, or database grants.
- Unknown host: Endpoint typo or DNS resolution issue.

Critical point: Do not make RDS publicly accessible as the first troubleshooting step. Prefer private connectivity within the VPC.

Scenario 9

Auto Scaling · Availability

Your Auto Scaling group keeps terminating and launching EC2 instances. Why?

Answer:

“I will inspect Auto Scaling activity history, EC2 status checks, target group health, scaling policies, launch template configuration, and bootstrap logs. I will determine whether the instances are failing health checks, running out of resources, or failing application initialization.”

Common root causes:

- Incorrect AMI or launch template.
- User-data script failure.
- Health-check grace period too short for startup.
- Incorrect ALB health-check path or port.
- Missing IAM permissions or inaccessible package repositories.
- Scaling policies that react to noisy metrics.

Fix: Address the actual failure first. Increasing the health-check grace period may help slow startup, but it will not fix an unhealthy application.

Scenario 10

Route 53 · DNS

Your application works using its IP address but not its domain name. What will you investigate?

Answer:

“I will inspect DNS resolution using `dig` or `nslookup`, verify the Route 53 record and hosted zone, confirm the record points to the correct ALB or endpoint, and check whether the domain's registrar has the correct authoritative name servers. I will also check TTL and DNS caching.”

```
nslookup app.example.com
dig app.example.com
```

If DNS resolves correctly but HTTPS fails, move to the TLS certificate, ALB listener, and application configuration. DNS success alone does not guarantee that the website is reachable.

Scenario 11

S3 · Event-driven architecture

An S3 upload is successful, but your Lambda function does not process the file. What do you check?

Answer:

“I will check the S3 event notification configuration, bucket and Lambda Region, event type, object prefix and suffix filters, Lambda resource-based permission, and CloudWatch logs. I will also check whether the function is invoked but failing during execution.”

If the Lambda processes the event but cannot write metadata to DynamoDB, I will check its execution role and table permissions separately.

Critical point: S3 event notifications can be delivered more than once. Design the processing logic to handle duplicate events safely, for example by using an idempotency key.

## Level 3 — Advanced scenarios and root-cause analysis

These are closer to L2/L3 support and DevOps interviews. Your answer should demonstrate prioritization, evidence-based diagnosis, and safe recovery.

Scenario 12

Production · Latency

The application is slow, but EC2 CPU utilization is only 20%. What could be the problem?

Answer:

“Low CPU does not mean the application is healthy. I will check ALB response time, request count, 5xx errors, memory and swap, disk latency, network performance, thread pools, connection pools, and downstream dependencies such as RDS.”

I will compare application latency with database query latency to isolate the bottleneck.

| Evidence                       | Possible cause                            |
| ------------------------------ | ----------------------------------------- |
| High ALB target response time  | Slow application or downstream dependency |
| High database connections      | Connection pool exhaustion                |
| High disk queue or latency     | Storage bottleneck                        |
| Many requests waiting, low CPU | Locking, thread starvation, network waits |
| Latency rises with traffic     | Capacity limit or inefficient code        |

Interview tip: Explain how you will measure the bottleneck before changing instance sizes.

Scenario 13

IAM · Security

An application worked yesterday but suddenly receives `AccessDenied` when calling an AWS API. Nothing changed in the application code.

Answer:

“I will check recent IAM and infrastructure changes, CloudTrail events, the identity currently used by the application, and any permission-boundary, SCP, session-policy, or resource-policy changes. I will verify whether temporary credentials expired or the application started using a different role.”

I will compare the failing API action and resource ARN with the expected permissions, then make the smallest authorized policy change.

Critical thinking: No application-code change does not mean nothing changed. Credentials, policies, resource ownership, organization controls, and session context can all change independently.

Scenario 14

RDS · Performance

Your application was healthy, but suddenly database queries take 10 seconds instead of 100 milliseconds. What will you do?

Answer:

“I will check database CPU, free memory, storage latency, connections, locks, slow queries, and recent schema or deployment changes. I will use Performance Insights or the available database monitoring tools to identify expensive queries and wait events. I will determine whether the issue is a query regression, resource saturation, or connection exhaustion.”

Immediate mitigation could include rolling back a problematic release, stopping a known runaway workload, or optimizing an expensive query after validating the evidence.

Avoid: Restarting RDS immediately without checking the impact. Restarting can cause downtime and may not fix the underlying query or resource issue.

Scenario 15

Disaster recovery

An Availability Zone becomes unavailable. How will you maintain application availability?

Answer:

“I will confirm the scope of the incident, check AWS Health and service metrics, and verify whether healthy targets remain in another Availability Zone. If the architecture is Multi-AZ, I will validate load balancer target health and capacity in surviving zones. I will check database failover behavior and application reconnection handling.”

Prevention includes:

- ALB across multiple Availability Zones.
- Auto Scaling capacity distributed across zones.
- RDS Multi-AZ where appropriate.
- Tested backup and disaster recovery procedures.
- Capacity headroom for surviving infrastructure.

Critical point: Multi-AZ deployment improves resilience, but it does not guarantee zero downtime or protect against every regional failure.

Scenario 16

CloudWatch · Alerting

CloudWatch shows no alarm, but customers report the website is down. How can this happen?

Answer:

“The monitoring may not cover the failing component, the alarm threshold may be too permissive, the metric may be delayed or missing, or the monitored resource may be healthy while the user-facing transaction fails. I will test the application from the user perspective and inspect ALB, application, DNS, and dependency metrics.”

I would add suitable alarms and synthetic checks for availability, latency, and error rates, with notifications through the team's incident-response channel.

Critical point: Monitor customer-visible outcomes, not only infrastructure utilization.

## Level 4 — Critical thinking and logical troubleshooting

These scenarios test your judgment. There may be multiple possible causes, so the best answer explains what you will verify before making a change.

Scenario 17

Production outage

Your website is down. The manager says, “Fix it immediately.” What are your first five actions?

Interview-ready answer:

“I will first assess customer impact and establish incident severity. I will check the ALB health and error rates, recent deployments and infrastructure changes, and application and dependency health. I will communicate the impact to stakeholders and, if a recent change is strongly implicated, use the approved rollback procedure. After restoring service, I will verify the customer journey and investigate the root cause.”

My priority order:

1. Establish impact and incident ownership.
2. Check health checks, error rates, and recent changes.
3. Isolate the failing layer using logs and metrics.
4. Restore service using the safest reversible action.
5. Validate recovery, communicate status, and preserve evidence.

What the interviewer is testing: Whether you prioritize recovery and communication over randomly restarting services.

Scenario 18

Network logic

EC2-A can connect to EC2-B, but EC2-B cannot initiate a connection to EC2-A. Why?

Interview-ready answer:

“I will compare the direction, destination port, and source and destination security groups for both flows. Security groups are stateful, so return traffic for an allowed connection is permitted automatically, but a new connection initiated in the reverse direction requires its own inbound and outbound rules. I will also check network ACLs, host firewalls, and application listeners.”

Logic to remember: An allowed connection in one direction does not imply that a new connection in the reverse direction is allowed.

Scenario 19

NAT · Connectivity

EC2 instances in a private subnet can reach RDS but cannot download software packages. What does this tell you?

Interview-ready answer:

“It suggests that private VPC connectivity is at least partly working, but internet egress may not be configured. I will inspect the private subnet route table, NAT gateway state and public-subnet route, and outbound security group and network ACL rules. I will also check DNS and whether the package repository itself is reachable.”

The private subnet typically needs a route such as:

```
0.0.0.0/0 → nat-xxxxxxxx
```

The NAT gateway's public subnet needs a route to an internet gateway, and the gateway must be available. If the architecture is intentionally isolated, I would evaluate VPC endpoints or an internal package repository instead.

Critical point: Successful RDS connectivity proves only that the database path works; it does not prove internet connectivity.

Scenario 20

S3 · Security incident

You discover that an S3 bucket containing confidential files has accidentally become public. What will you do?

Interview-ready answer:

“I will treat this as a security incident. I will immediately follow the incident-response procedure to restrict unauthorized access, using appropriate bucket policies and S3 Block Public Access controls after assessing dependencies. I will preserve CloudTrail and relevant access logs, determine exposure scope, identify the configuration change, and notify the security team. I will rotate credentials only if evidence indicates they were compromised.”

After containment, I will verify public access is blocked, investigate potential data access or exfiltration, and implement preventive controls such as organization-level guardrails and configuration monitoring.

Critical point: Blocking access stops further exposure but does not undo any data that may already have been accessed.

Scenario 21

Cost optimization

Your AWS bill increases by 300% overnight. How do you investigate?

Interview-ready answer:

“I will use Cost Explorer to compare daily spending by service, region, usage type, and linked account. I will inspect recent resource changes and billing data to identify the source, then check for unexpected EC2 instances, NAT gateway traffic, data transfer, EBS volumes, snapshots, logs, or Kubernetes workloads. I will stop or remove resources only after verifying ownership and production impact.”

My actions:

- Identify which service and usage category increased.
- Compare the increase against deployments and resource creation.
- Check Cost Anomaly Detection alerts and relevant billing reports.
- Contain unauthorized or unnecessary resource usage safely.
- Add budgets, alerts, and tagging to improve accountability.

Critical point: A budget alert notifies you; it does not automatically stop spending.

Scenario 22

High availability · Database

RDS fails over to a standby, but the application remains unable to connect. Why?

Interview-ready answer:

“I will check the database's current status and endpoint, connection errors, DNS resolution, and application connection-pool behavior. I will verify that the application uses the RDS endpoint rather than a hard-coded IP address. I will also confirm that clients retry connections with sensible backoff and can recover from stale pooled connections.”

Possible causes include:

- Application connection pools holding stale connections.
- DNS caching or hard-coded database IPs.
- Retry logic missing or too aggressive.
- Authentication or TLS configuration problems.
- Database recovery still in progress.

Critical point: Multi-AZ failover changes database infrastructure, but the application must still handle interrupted connections.

Scenario 23

Safe remediation

An EC2 instance is unhealthy, but it is the only server running a critical application. Would you reboot it immediately?

Interview-ready answer:

“Not without understanding the impact. I will confirm whether the instance is recoverable, whether it stores any state locally, and whether backups or replacement capacity exist. I will gather available diagnostics and determine whether rebooting, recovering the instance, or launching a replacement is the safest option. I will preserve data and coordinate any required outage.”

Critical point: The right recovery action depends on whether the workload is stateless, whether data is durable, and what recovery options are available. An automatic reboot can make some incidents worse.

## Level 5 — Architecture and senior-style scenarios

Scenario 24: Design a highly available AWS web application

You have a website, application backend, and MySQL database. How would you design it for production?

[AWS: Creating a Highly Available Environment | by NJUGUNAMOSES | Medium](https://images.openai.com/static-rsc-4/z-BkPgI35aJrzcwPujjiW-ufMMVqZZkQRLpCZ4YWNnqJst5yF5lt8uuSM26hUlWGat-I-x8m3IhqZzPQKSwIjpv1l4V1TMhwZEEXL3Al7Oyugh-Zm-5atO6qXw66j7gl5nNTqrDwea9wuPV1j5rH4H0yGVYWpLtiKU4koTiQcjg?purpose=inline)

[medium.com](https://medium.com/%40njugunamoses765/aws-creating-a-highly-available-environment-358127620f2c)

Interview-ready answer:

“I would create a VPC across at least two Availability Zones, with public subnets for the ALB and private subnets for the application servers and database. I would use an Auto Scaling group behind the ALB, an RDS Multi-AZ deployment, IAM roles, security-group references, CloudWatch monitoring, and automated backups. I would add Route 53, ACM for HTTPS, and AWS WAF if the application's threat model requires it.”

Important design decisions:

- No direct public access to application instances or RDS.
- Restrict database access to the application security group.
- Use multiple Availability Zones for resilient compute and database placement.
- Define recovery objectives, backup retention, and tested restore procedures.
- Plan for secrets management, patching, deployments, and cost controls.

Scenario 25: A deployment introduces errors for 20% of users

Would you troubleshoot the new version or roll it back?

Interview-ready answer:

“I will compare error rates, latency, and target health against the previous version, then check the deployment timeline and application logs. If evidence indicates the release caused a production regression, I will follow the rollback or traffic-shifting procedure to restore the known-good version. I will validate recovery before debugging the new release in a safe environment.”

Useful approaches include rolling deployments, blue/green deployments, and canary releases. The correct choice depends on application architecture, state compatibility, and rollback safety.

Critical point: Database schema changes and data migrations must be backward-compatible if rollback is expected to work safely.

Scenario 26: Your AWS account may be compromised

CloudTrail shows unfamiliar API calls and new compute resources in an unexpected region. What is your response?

Interview-ready answer:

“I will activate the security incident-response process, preserve logs and evidence, and coordinate containment with the authorized security team. I will identify the affected identities and resources, revoke or disable compromised access as appropriate, remove unauthorized resources safely, and inspect CloudTrail, IAM changes, and data access. I will then rotate affected credentials, review trust relationships, and strengthen detection and prevention controls.”

I would also review root-account security, MFA, access keys, cross-account roles, SCPs, and GuardDuty findings where available.

Critical point: Avoid deleting evidence or indiscriminately disabling every identity before understanding the incident and coordinating containment.

## Level 6 — Practice mock interview

Now test whether you can diagnose incidents independently. Read each question, choose your response, and reveal the suggested reasoning.

## AWS troubleshooting quiz

0/5 answered

1\. Users receive HTTP 504 errors, but the ALB is healthy. What do you check first?

Restart the ALB

Target response time, application logs, downstream dependencies and timeout behavior

Open every inbound port

Increase EC2 instance size immediately

2\. EC2 cannot read an S3 object, but its instance role has an S3 policy. What is the next logical step?

Make the bucket public

Attach AdministratorAccess

Check the exact API action, resource ARN, bucket policy, explicit denies and encryption permissions

Recreate the EC2 instance

3\. RDS is reachable from one EC2 instance but not another in the same VPC. What should you compare?

Only the database password

Both instances' security groups, routes, subnet network ACLs, DNS and host firewalls

The AWS account name

The RDS backup retention period

4\. A production application fails immediately after a deployment. What is usually the safest initial response when the release is confirmed as the cause?

Delete the database

Disable all monitoring

Use the approved rollback or traffic-shifting procedure, then validate recovery

Change several security groups at once

5\. CloudWatch CPU is normal, but the application is unavailable. What principle should guide you?

Normal CPU proves the server is healthy

Restart every AWS service

Investigate end-to-end availability, networking, application health and dependencies

Disable health checks

Check answers

## What to prepare before your AWS interview

Focus on these topics in this order:

| Priority              | Topics                                                       |
| --------------------- | ------------------------------------------------------------ |
| 1. Fundamentals       | EC2, S3, IAM, VPC, security groups, EBS                      |
| 2. Networking         | Subnets, route tables, IGW, NAT, NACL, DNS                   |
| 3. Troubleshooting    | ALB, Auto Scaling, RDS, Linux commands, CloudWatch           |
| 4. Production support | Incident response, RCA, rollback, backups, disaster recovery |
| 5. Advanced           | Multi-AZ, IAM security, Terraform, CI/CD, cost optimization  |

A strong answer should always include four things: the suspected cause, the evidence you would check, the safe remediation, and the preventive action.

For your hands-on preparation, practise these scenarios on a small AWS lab, especially EC2 connectivity, ALB target health, RDS access, IAM permissions, and CloudWatch alarms. Use budget alerts and clean up billable resources after testing.
