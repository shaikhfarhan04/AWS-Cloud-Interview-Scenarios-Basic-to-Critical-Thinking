# AWS EC2 SSH Troubleshooting Lab — Amazon Linux

Let's turn this scenario into a hands-on, interview-ready troubleshooting exercise. We will follow the same sequence an AWS Cloud or DevOps Engineer would use in a real production incident: identify the failure, isolate the layer, verify the root cause, and restore access safely.

## 1. Understand the scenario

EC2 instance: Running

OS: Amazon Linux 2023 · Connection: SSH · Port: TCP 22

Running

Incident: You cannot connect to the instance using SSH.

```
ssh -i "my-key.pem" ec2-user@<PUBLIC_IP>
```

Replace the example key filename and public IP with your actual values.

A running instance does not guarantee that SSH is working. The problem could be in the network, AWS security configuration, operating system, SSH service, or authentication.

## 2. Follow this troubleshooting flow

Step 1 — Check EC2 health

Instance state and status checks

Step 2 — Check network reachability

IP address, security group, routes, NACL

Step 3 — Check the operating system

Firewall, sshd service, system logs

Step 4 — Check authentication

Username, private key, file permissions

Step 5 — Verify SSH access is restored

## 3. Step-by-step troubleshooting

### Step 1 — Check EC2 instance health

Open the AWS Console → EC2 → Instances → select your instance.

Verify:

- Instance state is `Running`.
- Instance status checks are `2/2 checks passed`.
- The instance has the expected public IPv4 address, if connecting over the internet.
- The system and instance status checks do not show failures.

[What is the Compute resources? How to create an EC2 Instance and download the PEM file to generate the PPK (key pair) from it? - DEV Community](https://images.openai.com/static-rsc-4/a5zwVkpKJcf567Avy6yPmvuMASoawbCXEbXAKEgJ7H2caQLFNkIkq0qSR3oUYnVqR6uBKvWO6sSPXSrlc3M4J4c6nb6GuL5oXFRjdgiHRNKSagYd__FVh7qyI6yqG3reUTTLvv1A_SqInTjvxU5Yjww05v68-fMsL-UNi56akzk?purpose=inline)

[dev.to](https://dev.to/aws-builders/explain-what-is-the-compute-resources-create-an-ec2-instance-and-download-the-pem-file-to-generate-the-ppk-key-pair-from-it-bf4?utm_source=chatgpt.com)

If status checks fail: investigate the underlying host, boot process, memory pressure, filesystem issues, or networking problems indicated by the failed check. Review EC2 system logs and screenshots where available.

### Step 2 — Test SSH from your Windows laptop

Open PowerShell and run:

```
Test-NetConnection <PUBLIC_IP> -Port 22
```

Example:

```
Test-NetConnection 54.123.45.67 -Port 22
```

Interpret the result:

| Result                    | Meaning                                                             |
| ------------------------- | ------------------------------------------------------------------- |
| `TcpTestSucceeded: True`  | TCP port 22 is reachable; check SSH authentication next.            |
| `TcpTestSucceeded: False` | A network path, firewall, routing, or SSH listener issue may exist. |

Do not assume a failed TCP test proves the security group is the problem.

### Step 3 — Verify the security group

Navigate to EC2 → instance → Security tab → attached security group → Inbound rules.

Check for a rule similar to this:

| Type | Protocol | Port | Source                          |
| ---- | -------- | ---- | ------------------------------- |
| SSH  | TCP      | 22   | Your authorized public IP `/32` |

To find your current public IP, use a trusted public-IP service or your organization's approved network tools.

Important: Do not open SSH to `0.0.0.0/0` as a permanent fix. If your public IP changed, update the source to your current authorized IP.

Also check whether the instance has multiple security groups attached. Their inbound rules are combined.

### Step 4 — Check the route table and internet gateway

If you connect directly over the internet using a public IPv4 address, verify:

- The subnet's associated route table has `0.0.0.0/0` pointing to an attached internet gateway (`igw-...`).
- The internet gateway is attached to the correct VPC.
- The instance has a public IPv4 address or Elastic IP.
- The subnet and VPC are the intended ones.

A public IP alone does not guarantee reachability.

If the instance is in a private subnet, use an approved access path such as AWS Systems Manager Session Manager, a bastion host, or a VPN. A private instance generally should not be exposed directly to the internet just to fix SSH.

### Step 5 — Check the network ACL

Open VPC → Network ACLs and identify the ACL associated with the instance's subnet.

Check both inbound and outbound rules. Network ACLs are stateless, so return traffic must also be permitted.

Typical requirements for direct SSH from the internet:

| Direction | Protocol | Port                                    | Purpose          |
| --------- | -------- | --------------------------------------- | ---------------- |
| Inbound   | TCP      | 22                                      | SSH request      |
| Outbound  | TCP      | Ephemeral ports, typically `1024–65535` | Response traffic |

Rules are evaluated in numerical order, and an earlier matching deny rule can block traffic. Ensure the source and destination ranges match your actual connection path.

### Step 6 — Check the operating system and SSH service

If the network appears correct, use Session Manager or another approved administrative path to access the instance.

Run these commands on Amazon Linux 2023:

```
# Check SSH service status
sudo systemctl status sshd

# Check whether SSH is listening on port 22
sudo ss -lntp | grep ':22'

# Validate SSH configuration
sudo sshd -t

# Review recent SSH-related logs
sudo journalctl -u sshd -n 100 --no-pager
```

If the service is stopped, first inspect the logs and configuration. If it is safe to restart:

```
sudo systemctl restart sshd
sudo systemctl status sshd
```

If `sshd -t` reports an error, correct the invalid configuration before restarting. Do not disable the firewall indiscriminately; inspect the active firewall rules and make the smallest necessary change.

### Step 7 — Verify the username, key, and authentication

For an Amazon Linux EC2 instance, the default username is usually `ec2-user`.

From Windows PowerShell:

```
ssh -vvv -i "C:\Keys\my-key.pem" ec2-user@<PUBLIC_IP>
```

The verbose output helps distinguish connection failures from authentication failures.

On Linux or macOS, restrict private-key permissions if required:

```
chmod 400 my-key.pem
```

For Windows, ensure your private key is accessible only to authorized users according to your organization's security policy.

Also verify that:

- You are using the private key corresponding to the public key configured for the instance.
- You are using the correct username and IP address.
- The instance's `authorized_keys` file and home-directory permissions are correct.
- The account has not been locked or otherwise restricted.

A lost private key cannot simply be downloaded again from EC2. Use an approved recovery method rather than replacing keys or changing access controls blindly.

## 4. Identify the root cause from the SSH error

Use the exact error message to narrow the investigation.

| SSH error                               | Likely causes                                                   | Next action                                          |
| --------------------------------------- | --------------------------------------------------------------- | ---------------------------------------------------- |
| `Connection timed out`                  | SG, NACL, route, wrong IP, firewall, network path               | Troubleshoot network connectivity                    |
| `Connection refused`                    | SSH service stopped, no listener on port 22, active reject rule | Check `sshd` and firewall                            |
| `Permission denied (publickey)`         | Wrong key, username, or authorized keys                         | Troubleshoot authentication                          |
| `WARNING: UNPROTECTED PRIVATE KEY FILE` | Private-key permissions too broad                               | Correct key file permissions                         |
| `No route to host`                      | Routing or network reachability problem                         | Check network path and firewall                      |
| `Host key verification failed`          | Server host key changed or known-hosts mismatch                 | Verify server identity before updating `known_hosts` |

## 5. Recovery options if SSH is completely unavailable

If you cannot SSH and the operating system is still running, consider these options in order:

Option A — Systems Manager Session Manager

Preferred when configured. Requires appropriate IAM permissions, SSM Agent, and connectivity to Systems Manager endpoints.

Option B — EC2 Serial Console

Useful for certain boot and OS-level failures, where the instance type, configuration, permissions, and account prerequisites support it.

Option C — Offline recovery

For severe OS or configuration failures, use an approved recovery workflow, potentially involving a snapshot, a rescue instance, or attaching the root volume to another instance. Plan for downtime and protect data.

## 6. Hands-on interview practice

## Incident #001: SSH connection failure

Assume the EC2 instance is running. Select the observed conditions and determine your next troubleshooting action.

1\. What error do you receive?

Connection timed out

Connection refused

Permission denied (publickey)

2\. What do the EC2 status checks show?

2/2 checks passed

One or more checks failed

3\. Does the instance have a public IPv4 address?

Yes

No

4\. Is Systems Manager Session Manager already working?

Yes

No

Analyze this incident

## 7. Final interview-ready answer

> “First, I will check the EC2 instance state and both status checks. Next, I will verify the public IP or private network path, security group inbound rule for TCP port 22, subnet route table, internet gateway, and network ACL.
>
> If network connectivity is available but SSH still fails, I will use Systems Manager or another approved access method to check the operating system firewall, SSH daemon status, SSH configuration, and authentication logs. I will verify the correct username, private key, and file permissions.
>
> Finally, I will fix the identified root cause, test SSH connectivity again, and document the incident. I will avoid opening port 22 to everyone or rebooting the instance before collecting diagnostic evidence.”

Interview tip: Always distinguish a network timeout from an authentication failure. That shows the interviewer you troubleshoot systematically instead of randomly changing AWS settings.
