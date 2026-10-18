# AWS Hands-on Lab: Troubleshoot and Resolve HTTP 502 on an Application Load Balancer

Yes. Let's recreate this as a real AWS troubleshooting exercise using an Application Load Balancer (ALB) + EC2 + Nginx on Amazon Linux 2023, then deliberately introduce a failure, diagnose it, resolve it, and verify the website works again.

We'll follow an interview-style production incident workflow, using AWS Console, SSH, Linux commands, and CloudWatch metrics.

[負載平衡器子網路和路由 - AWS 方案指引](https://images.openai.com/static-rsc-4/U7n1nsgJUYCon1obHVdETXmkcxH8e9Ves4G7Y-FgY--rQkvIAukFZ-h_ipZC4Tu9NIOQWkA1BoTK2j5QpohHPB306QeQNI4fWvSYK0THnNOA81qVyhPwL-e9C3fnQxzJE5HcLaytlNuXVCCgtBH9fMZoa8jwP41knJJgeTcBmX8?purpose=inline)

Lab architecture

- Internet → ALB → EC2 web server
- Target group with HTTP health checks
- Nginx serving a test website
- Security groups restricting backend traffic to the ALB

Incident to reproduce

The website is reachable through the ALB DNS name, but requests return HTTP 502 Bad Gateway after we intentionally stop the backend web service.

One important detail: stopping Nginx on an EC2 target may cause the ALB health check to mark it unhealthy, and requests may then return 503 instead of 502. To reproduce a reliable 502, we'll first create a healthy baseline, then use a controlled backend failure and observe the actual status code and target health. The exact result depends on the failure and ALB behavior.

## Phase 1 — Prepare the lab

### 1. Choose your lab settings

Use an existing VPC if you already have a suitable one, or create a dedicated lab VPC. The simplest setup uses two public subnets in different Availability Zones for the ALB and one EC2 target. For this exercise, keep the EC2 instance private if you already have a NAT/bastion/SSM access path; otherwise, a temporary public EC2 instance is easier to manage.

Use this configuration for the lab:

| Component           | Configuration                        |
| ------------------- | ------------------------------------ |
| Region              | `ap-south-1`                         |
| EC2 OS              | Amazon Linux 2023                    |
| Web server          | Nginx                                |
| Target port         | TCP 80                               |
| ALB listener        | HTTP 80                              |
| Target health check | HTTP `/health.html`                  |
| Instance type       | `t3.micro` if available and suitable |

Cost warning: An ALB is normally billed while it exists, and EC2, public IPv4 addresses, and data transfer can also incur charges. This is not a Free Tier–guaranteed lab. Delete the resources after testing.

## Phase 2 — Create the backend EC2 web server

1. Open the Amazon EC2 console in `ap-south-1`.
2. Launch an Amazon Linux 2023 instance.
3. Allow SSH (port 22) only from your own public IP if you need SSH access.
4. Create a security group named `alb-lab-ec2-sg`. Add an inbound HTTP rule for port 80 with the ALB security group as its source. You can create this rule after the ALB security group exists.
5. Connect to the EC2 instance using SSH or EC2 Instance Connect.

Install and start Nginx:

```
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

Create a test website and health-check endpoint:

```
echo '<h1>ALB Lab - Backend Healthy</h1>' \
  | sudo tee /usr/share/nginx/html/index.html

echo 'OK' \
  | sudo tee /usr/share/nginx/html/health.html
```

Verify the service locally:

```
sudo systemctl status nginx --no-pager
sudo ss -lntp | grep ':80'
curl -i http://localhost/
curl -i http://localhost/health.html
```

Expected results: Nginx is active, port 80 is listening, and both HTTP requests return `200 OK`.

If these checks fail, fix the backend before continuing.

## Phase 3 — Create the target group

Open EC2 → Target Groups → Create target group.

### Target group configuration

Target type

Instances

Target group name

`alb-lab-tg`

Protocol / Port

HTTP / 80

Health check path

`/health.html`

Success codes

`200`

Register your EC2 instance as a target on port 80 and create the target group. Wait for the instance's health to become Healthy. The health-check path must return an accepted status code for the target to pass its checks.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Elastic Load Balancing

+1



## Phase 4 — Create the Application Load Balancer

Open EC2 → Load Balancers → Create load balancer → Application Load Balancer.

Configure the following:

- Name: `alb-lab`
- Scheme: Internet-facing
- IP address type: IPv4
- VPC: the same VPC as the EC2 instance
- Mappings: two Availability Zones, each with a suitable subnet
- Listener: HTTP on port 80
- Default action: forward to `alb-lab-tg`

Create a security group named `alb-lab-alb-sg` with:

| Direction | Protocol/port | Source or destination                     |
| --------- | ------------- | ----------------------------------------- |
| Inbound   | HTTP 80       | `0.0.0.0/0` for this temporary public lab |
| Outbound  | HTTP 80       | `alb-lab-ec2-sg`                          |

Then update `alb-lab-ec2-sg` to allow inbound TCP 80 from `alb-lab-alb-sg`, not from the entire internet.

Make sure the ALB subnets have a route to an Internet Gateway and that the instance and ALB use the same VPC. For a public lab, ensure the EC2 instance can be reached through your chosen administrative access method.

AWS recommends allowing the ALB security group to communicate with the targets on both the application and health-check ports.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Elastic Load Balancing

+1



## Phase 5 — Verify the healthy baseline

Find the ALB's DNS name under its details. Open it in your browser:

`http://<your-ALB-DNS-name>`

You should see:

ALB Lab — Backend Healthy

Expected HTTP response: 200 OK

Now verify these three things:

1. ALB state is `Active`.
2. The target group reports the instance as `Healthy`.
3. The website works through the ALB DNS name, not just through `localhost` on EC2.

Do not proceed until the baseline is working.

## Phase 6 — Recreate the incident

We will simulate a backend outage by stopping Nginx. Only do this in your lab, not on a production server.

On the EC2 instance:

```
sudo systemctl stop nginx
sudo systemctl status nginx --no-pager
```

Test the backend:

```
curl --max-time 3 -i http://localhost/
```

The local request should fail because Nginx is stopped.

Now refresh the ALB DNS name repeatedly and open:

EC2 → Target Groups → `alb-lab-tg` → Targets.

Observe the following:

- The target may initially remain healthy until enough health checks fail.
- It will generally become unhealthy with a health-check failure reason.
- Client requests may return 502 or 503 depending on connection behavior and the target group's health state. Do not assume that stopping Nginx guarantees a 502.

AWS documents several ALB 502 causes, including target resets, unexpected target responses, and malformed HTTP responses.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://repost.aws\&sz=32)

AWS re:Post

+1



### Capture the incident evidence

Record what actually happens:

### Incident observation sheet

Select the results you observe in your lab.

HTTP status through ALB

502 Bad Gateway

503 Service Unavailable

504 Gateway Timeout

Other / connection failure

Target group health

Healthy

Unhealthy

Initial / still checking

Local request to localhost:80

Failed / connection refused

200 OK

Other response

Diagnose my results

## Phase 7 — Troubleshoot the incident systematically

Follow this sequence rather than changing several settings at once.

Check 1: Listener and rules

In the ALB console, confirm the HTTP:80 listener forwards to `alb-lab-tg`. Verify there is no unintended listener rule forwarding requests to another target group.

Check 2: Target health

Inspect the target health reason. `Target.Timeout` suggests the health check did not receive a timely response; `Target.ResponseCodeMismatch` means the response code was not accepted; `Target.FailedHealthChecks` indicates a failed check that needs further investigation.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Elastic Load Balancing

+1



Check 3: Security groups and network

Verify the ALB can send traffic to the target on port 80 and the EC2 security group permits that traffic from the ALB security group. Check network ACLs if connectivity still fails.

Check 4: Application and listening port

On EC2, run:

```
sudo systemctl status nginx --no-pager
sudo ss -lntp | grep ':80'
sudo nginx -t
curl -i http://localhost/health.html
```

If Nginx is stopped, check its logs before restarting it.

Check 5: Logs and metrics

Check `/var/log/nginx/error.log` and `/var/log/nginx/access.log`. In CloudWatch, examine `HTTPCode_ELB_502_Count`, `HTTPCode_Target_5XX_Count`, and `TargetConnectionErrorCount`. Enable ALB access logs to S3 if needed to identify the response source.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://repost.aws\&sz=32)

AWS re:Post

+1



## Phase 8 — Resolve the issue

For our simulated incident, the root cause is that the Nginx service was stopped.

### Step 1: Restart Nginx

On the EC2 instance:

```
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx --no-pager
```

If it fails to start, inspect the error before proceeding:

```
sudo journalctl -u nginx -n 50 --no-pager
sudo nginx -t
sudo tail -n 50 /var/log/nginx/error.log
```

Correct any configuration errors, then start the service again.

### Step 2: Verify the backend

```
curl -i http://localhost/
curl -i http://localhost/health.html
```

Both requests should return HTTP `200 OK`.

If the local request succeeds but the ALB still fails, investigate network access, security groups, the target port, the health-check path, and the target group's protocol.

### Step 3: Wait for target health recovery

In the AWS Console, open:

EC2 → Target Groups → `alb-lab-tg` → Targets.

Wait for the target to return to Healthy. Health recovery takes time because the ALB uses configured health-check intervals and thresholds.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Elastic Load Balancing



### Step 4: Test through the ALB again

```
curl -i http://<your-ALB-DNS-name>/
```

Expected response:

```
HTTP/1.1 200 OK

<h1>ALB Lab - Backend Healthy</h1>
```

Your exact response headers may differ. Refresh the website in your browser as well.

## Phase 9 — Advanced challenge: Identify the source of a real 502

Once the lab works, use AWS's diagnostic approach to distinguish a load-balancer-generated 502 from a backend-generated 502.

| Evidence                                                 | What it suggests                                                  |
| -------------------------------------------------------- | ----------------------------------------------------------------- |
| `HTTPCode_ELB_502_Count` increases                       | The ALB generated 502 responses                                   |
| `HTTPCode_Target_5XX_Count` increases                    | The target generated 5xx responses                                |
| Access log `elb_status_code=502`, `target_status_code=-` | The ALB generated the error before receiving a target HTTP status |
| Access log shows both codes as 502                       | The target returned 502 to the ALB                                |

Interpret metrics alongside the access logs and time of the incident. A single metric by itself may not establish the root cause.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://repost.aws\&sz=32)

AWS re:Post



For an additional controlled exercise, change the target group's health-check path to `/wrong-path`. Watch the target become unhealthy because the endpoint returns a non-matching response. Restore `/health.html` and confirm recovery.

## Phase 10 — Clean up AWS resources

To avoid ongoing charges, delete the lab when finished:

1. Delete the ALB.
2. Delete the target group after it is no longer in use.
3. Terminate the lab EC2 instance.
4. Delete the lab security groups after their dependencies are removed.
5. Remove any lab-only resources you created, such as unused public IP addresses, S3 log buckets, or a dedicated VPC.

## Interview-ready final answer

> “I would first confirm whether the 502 is generated by the ALB or by the backend using CloudWatch metrics and ALB access logs. Next, I would inspect the listener rules, target-group health and reason codes, security-group and network ACL rules, backend port and protocol, and application logs. I would reproduce the issue safely in a test environment, identify the root cause, restore the backend or correct the misconfiguration, and verify recovery through the ALB DNS name. Finally, I would monitor error rates and target health to confirm stability and document the root cause and preventive actions.”

Your first milestone: Complete Phases 2–5 and confirm that the website returns `200 OK` through the ALB. Then introduce the failure in Phase 6. This gives you a working baseline, a controlled incident, and a verified resolution rather than simply memorizing troubleshooting commands.

# Diagnosis: AWS ALB returns HTTP 502

Based on your observations, we have a strong initial lead:

- ALB response: `502 Bad Gateway`
- Target group: `Unhealthy`
- EC2 localhost test: Failed

The first priority is to restore the backend web server. Since `localhost:80` fails on the EC2 instance itself, troubleshoot Nginx or the application before changing the ALB configuration.

The likely root cause is that the web service is stopped, failed to start, or is not listening on port 80. We should confirm which one before making changes.

## Step 1 — Check the backend service

SSH into your EC2 instance and run these commands in order.

```
sudo systemctl status nginx --no-pager -l
```

Then check whether anything is listening on port 80:

```
sudo ss -lntp | grep ':80'
```

Finally, test the backend directly:

```
curl -v --max-time 5 http://127.0.0.1/
```

Interpret the results:

| Observation            | Likely cause                                                       |
| ---------------------- | ------------------------------------------------------------------ |
| `inactive (dead)`      | Nginx is stopped                                                   |
| `failed`               | Startup or configuration error                                     |
| `Connection refused`   | Nothing is accepting connections on that address and port          |
| `Connection timed out` | Investigate local firewall or network behavior                     |
| HTTP `404`             | Server responds, but the requested path may be missing             |
| HTTP `200`             | Backend works locally; continue with ALB-to-target troubleshooting |

## Step 2 — Apply the likely fix

If the service status shows `inactive (dead)`, run:

```
sudo systemctl start nginx
sudo systemctl enable nginx
sudo systemctl status nginx --no-pager
```

If Nginx is not installed, install it first:

```
sudo dnf install -y nginx
sudo systemctl enable --now nginx
```

If Nginx fails to start, don't keep restarting it. Check the reason:

```
sudo nginx -t
sudo journalctl -u nginx -n 50 --no-pager
sudo tail -n 50 /var/log/nginx/error.log
```

Correct the reported configuration or startup problem, then start Nginx again.

## Step 3 — Verify the backend is healthy

Run:

```
curl -i http://127.0.0.1/
curl -i http://127.0.0.1/health.html
sudo ss -lntp | grep ':80'
```

If your lab uses `/health.html`, make sure it exists and returns `200 OK`:

```
echo 'OK' | sudo tee /usr/share/nginx/html/health.html
```

The expected result is that Nginx listens on port 80 and both requests return successful HTTP responses.

## Step 4 — Check the ALB target health

In AWS Console:

1. Open EC2 → Target Groups → `alb-lab-tg`.
2. Select Targets.
3. Inspect the unhealthy target's Status details.
4. Confirm the target port is `80` and health-check path is `/health.html`.

If the target remains unhealthy, use its reason code to choose the next step. AWS documents these reasons and their corresponding checks.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Elastic Load Balancing

+1



| Target health reason          | Next action                                                                    |
| ----------------------------- | ------------------------------------------------------------------------------ |
| `Target.FailedHealthChecks`   | Verify Nginx status and listening port                                         |
| `Target.ResponseCodeMismatch` | Test `/health.html` and correct the expected status codes                      |
| `Target.Timeout`              | Check security groups, network ACLs, and response delays                       |
| `Target.NotInUse`             | Check target-group association, listener rules, and enabled Availability Zones |

## Step 5 — Verify the full request path

Once the target becomes `Healthy`, test from your Windows PowerShell terminal:

```
curl.exe -i http://<your-ALB-DNS-name>/
```

Expected: `HTTP/1.1 200 OK` and your website content.

If localhost succeeds but the ALB still returns 502, check whether the EC2 security group allows inbound TCP 80 from the ALB security group, and verify that the target group uses HTTP on port 80. ALB access logs and CloudWatch metrics can help distinguish a connection reset from a backend-generated error.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://repost.aws\&sz=32)

AWS re:Post

+1



## Your next action

Run this single command first and share its output:

```
sudo systemctl status nginx --no-pager -l
```

That will tell us whether to start the service, repair its configuration, or investigate a different cause. We can then resolve the incident based on actual evidence rather than making unnecessary changes.
