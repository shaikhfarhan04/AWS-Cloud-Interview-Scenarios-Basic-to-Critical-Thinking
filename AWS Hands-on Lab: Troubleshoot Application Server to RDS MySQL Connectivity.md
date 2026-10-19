# AWS Hands-on Lab: Troubleshoot Application Server to RDS MySQL Connectivity

We’ll build a real AWS environment where a web application runs on an EC2 instance and connects to an Amazon RDS MySQL database through a private network. Then we’ll deliberately break the connectivity, investigate the evidence, and fix the incident like a DevOps or Cloud Support Engineer.

This lab is designed around the interview scenario you provided, with actual AWS resources and commands rather than a theory-only exercise.

[Cloud Architecture Uncovered:  A Blueprint for Modern IT Solutions](https://images.openai.com/static-rsc-4/QEEM9jxrf_hITjZDjfZCNhASXnhhFGiLcY4JYJnL-paYeVjoY5bot-XItzrUDnDhaAa9FXLtaR_Wn8_z50A6qYuj34XyAn7IW1tQwF6S6cjvrXrX6NFF_hzCobM36Pvh4Top4OGoehXBFE_m2JGqEC82uwfnjGZ8YqfQdAi9BQE?purpose=inline)

[cloudchipr.com](https://cloudchipr.com/blog/cloud-architecture-uncovered-a-blueprint-for-modern-it-solutions?utm_source=chatgpt.com)

## What you will build

Your browser

HTTP access to the application on EC2

EC2 application server

Amazon Linux 2023 + Python Flask + MySQL client

Application security group: allows HTTP and controlled administration

Private VPC connection — TCP 3306

Amazon RDS for MySQL

Private database subnets, no public access

Database security group: MySQL inbound only from the EC2 security group

## 1. Lab plan

| Component        | Configuration                                                  |
| ---------------- | -------------------------------------------------------------- |
| AWS Region       | `ap-south-1` (Mumbai)                                          |
| Network          | One dedicated VPC                                              |
| Application      | EC2, Amazon Linux 2023, Flask                                  |
| Database         | Amazon RDS MySQL                                               |
| Database port    | TCP `3306`                                                     |
| Network security | Separate application and database security groups              |
| Database access  | Private only                                                   |
| Troubleshooting  | DNS, TCP, security groups, routing, authentication, SQL grants |

For a cost-conscious lab, use a small eligible EC2 instance and the smallest RDS instance class currently available for your account and region. RDS can incur charges even when idle, and free-tier eligibility depends on your account. We’ll create resources in stages and delete them after testing.

Important: Don't open MySQL port `3306` to `0.0.0.0/0`, and don't make the RDS database publicly accessible just to solve a connection issue.

## 2. Create the AWS network and security groups

We’ll use the AWS Management Console for the first deployment so you can see exactly how each service is configured.

Open the [AWS VPC Console](https://console.aws.amazon.com/vpc/home?region=ap-south-1\&utm_source=chatgpt.com) and select Mumbai (`ap-south-1`).

### Step 2.1 — Choose a VPC

For the quickest lab, use your account's existing default VPC, provided it has subnets in at least two Availability Zones. This avoids creating a NAT Gateway, which can add substantial cost.

Record these values:

- VPC ID
- Two subnet IDs in different Availability Zones
- Subnet route table
- VPC CIDR

If you want a fully isolated, custom network instead, we can build a separate VPC with public and private subnets as a follow-up.

### Step 2.2 — Create the application security group

Open [EC2 Security Groups](https://console.aws.amazon.com/ec2/home?region=ap-south-1\&utm_source=chatgpt.com#SecurityGroups) and create:

### Security group: `rds-lab-app-sg`

VPC: your selected VPC

Inbound rules

| Type       | Port | Source                 |
| ---------- | ---- | ---------------------- |
| SSH        | 22   | My IP                  |
| Custom TCP | 5000 | My IP, for lab testing |

The SSH rule is for administration. Port 5000 exposes the Flask test app directly, so restrict it to your IP and remove it after the lab. A production deployment would normally put an ALB in front of the application.

Leave the default outbound rule in place for this learning lab.

### Step 2.3 — Create the database security group

Create another security group:

### Security group: `rds-lab-db-sg`

Inbound rules

| Type         | Port | Source           |
| ------------ | ---- | ---------------- |
| MySQL/Aurora | 3306 | `rds-lab-app-sg` |

The source must be the application security group, not your public IP and not `0.0.0.0/0`.

AWS security groups are stateful firewalls. Referencing the application security group allows the database to accept MySQL connections from associated application instances, subject to the other network and database requirements.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Getting Started Guide

+1



## 3. Launch the EC2 application server

Open the [EC2 Console](https://console.aws.amazon.com/ec2/home?region=ap-south-1\&utm_source=chatgpt.com#Instances:).

Select Launch instance and configure:

| Setting               | Value                                                      |
| --------------------- | ---------------------------------------------------------- |
| Name                  | `rds-lab-app-01`                                           |
| AMI                   | Amazon Linux 2023                                          |
| Instance type         | A small eligible instance, such as `t3.micro` if available |
| VPC                   | The VPC selected in Step 2                                 |
| Subnet                | A subnet with internet access for package installation     |
| Auto-assign public IP | Enabled for this temporary lab                             |
| Security group        | `rds-lab-app-sg`                                           |
| IAM role              | Optional; use a least-privilege role if required           |

Create or select an SSH key pair, then launch the instance. Keep the SSH source restricted to your own IP.

Connect to the instance through SSH or EC2 Instance Connect. On Amazon Linux 2023, install the required tools:

```
sudo dnf update -y
sudo dnf install -y python3 python3-pip mariadb105
python3 --version
mysql --version
```

AWS documents `mariadb105` as the MySQL-compatible command-line client package for Amazon Linux 2023.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Relational Database Service

+1



## 4. Create the RDS MySQL database

Open the [Amazon RDS Console](https://console.aws.amazon.com/rds/home?region=ap-south-1\&utm_source=chatgpt.com).

Select Databases → Create database.

Use these settings:

| Setting                | Value                                                    |
| ---------------------- | -------------------------------------------------------- |
| Creation method        | Standard create                                          |
| Engine                 | MySQL                                                    |
| Template               | Dev/Test, if available                                   |
| DB instance identifier | `rds-lab-mysql`                                          |
| Master username        | `labadmin`                                               |
| Master password        | Generate a strong, unique password and store it securely |
| Instance class         | Smallest suitable class currently offered                |
| Storage                | Smallest supported allocation                            |
| VPC                    | Same VPC as the EC2 instance                             |
| DB subnet group        | Subnets spanning at least two Availability Zones         |
| Public access          | No                                                       |
| VPC security group     | `rds-lab-db-sg`                                          |
| Database port          | `3306`                                                   |
| Initial database name  | `labdb`                                                  |

Leave optional features such as Multi-AZ, Performance Insights add-ons, and enhanced monitoring off unless you need them for a separate exercise. Review backup and storage settings before creating the database.

Wait until the database status is `Available`.

AWS's RDS setup documentation covers database creation, EC2 connectivity, and the endpoint you need for the client connection.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.aws.amazon.com\&sz=32)

Amazon Relational Database Service

+1



Once it is available, open the database details and copy:

- Endpoint (hostname)
- Port
- VPC ID
- DB subnet group
- Associated security group
- Publicly accessible status

Do not use the database's IP address in the application. Use its DNS endpoint.

## 5. Build the Flask application that connects to MySQL

Now we’ll make the EC2 instance run a small application that reports database connectivity and reads a table from RDS.

### Step 5.1 — Create the application

On the EC2 instance:

```
mkdir -p ~/rds-connect-lab
cd ~/rds-connect-lab

python3 -m venv .venv
source .venv/bin/activate

pip install Flask PyMySQL
```

Using a virtual environment keeps the Python dependencies isolated from the system Python packages.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://flask.palletsprojects.com\&sz=32)

Flask Documentation (3.1.x)



Create the application file:

```
nano app.py
```

Paste the following code:

```
import osimport pymysqlfrom flask import Flask, jsonifyapp = Flask(__name__)def get_db_connection():    return pymysql.connect(        host=os.environ["DB_HOST"],        port=int(os.getenv("DB_PORT", "3306")),        user=os.environ["DB_USER"],        password=os.environ["DB_PASSWORD"],        database=os.getenv("DB_NAME", "labdb"),        connect_timeout=5,        read_timeout=5,        write_timeout=5    )@app.route("/")def home():    return """    <h1>AWS RDS Connectivity Lab</h1>    <p>Try /health and /db-check</p>    """@app.route("/health")def health():    return jsonify(status="ok", application="rds-connect-lab"), 200@app.route("/db-check")
```

Save and exit Nano with `Ctrl+O`, Enter, then `Ctrl+X`.

### Step 5.2 — Configure the database connection

Set the endpoint you copied from RDS. Replace the example hostname and password with your actual values.

```
export DB_HOST='your-rds-endpoint'
export DB_PORT='3306'
export DB_USER='labadmin'
export DB_PASSWORD='your-actual-password'
export DB_NAME='labdb'
```

Do not commit these credentials to GitHub or put them in your source code. These environment variables are suitable for this temporary interactive lab; for a longer-lived deployment, use AWS Secrets Manager or another approved secrets-management method.

### Step 5.3 — Start the application

```
cd ~/rds-connect-lab
source .venv/bin/activate
python app.py
```

Leave the process running. Open these URLs from your computer, substituting your EC2 public IP:

- `http://YOUR-EC2-PUBLIC-IP:5000/`
- `http://YOUR-EC2-PUBLIC-IP:5000/health`
- `http://YOUR-EC2-PUBLIC-IP:5000/db-check`

Expected database response:

```
{
  "database": "labdb",
  "mysql_version": "8.0.x",
  "status": "connected"
}
```

The exact MySQL version will depend on the version selected when you created RDS.

If `/health` works but `/db-check` fails, that is a useful distinction: the web application is running, but the database operation is not succeeding.

## 6. Verify connectivity directly from EC2

Open a second SSH session to the application instance.

Set the hostname in a shell variable:

```
export DB_HOST='your-rds-endpoint'
```

### Check A — DNS resolution

```
nslookup "$DB_HOST"
```

Expected result: the hostname resolves to an address.

If `nslookup` is unavailable, try:

```
getent hosts "$DB_HOST"
```

If the hostname does not resolve, verify the endpoint spelling, RDS status, and VPC DNS settings.

### Check B — TCP port connectivity

```
nc -vz -w 5 "$DB_HOST" 3306
```

If `nc` is missing, install the package that provides it or use the MySQL client for a connection test.

A successful TCP connection confirms that the port is reachable from the EC2 instance. It does not prove that the database username or password is valid.

### Check C — MySQL authentication

```
mysql -h "$DB_HOST" -P 3306 \
  -u "$DB_USER" -p "$DB_NAME"
```

Enter the password at the prompt. Don't append the password to the command itself.

At the MySQL prompt, run:

```
SELECT DATABASE(), CURRENT_USER(), VERSION();
SHOW TABLES;
EXIT;
```

The database can be reachable even if authentication fails. That is why we test DNS, TCP, and SQL authentication separately.

## 7. Create the incident: deliberately break the RDS connection

First, confirm that `/db-check` returns HTTP 200 and that the MySQL client connects successfully. Then we’ll simulate real production-style incidents.

## Incident simulator

Choose an incident to reproduce in your AWS lab. The expected evidence and fix will update below.

Select an incident

Incident 1 — Incorrect database security group

Incident 2 — Incorrect endpoint hostname

Incident 3 — Incorrect database password

Incident 4 — Database permission failure

Network incident

Simulate: In `rds-lab-db-sg`, replace the inbound MySQL source `rds-lab-app-sg` with a different security group that is not attached to the EC2 instance.

Expected evidence: `nslookup` succeeds, but `nc -vz -w 5 "$DB_HOST" 3306` typically times out.

Resolution: Restore the inbound TCP `3306` rule with source `rds-lab-app-sg`. Retest TCP connectivity and then `/db-check`.

For Incident 1, if you want to reproduce the failure without changing the security group, you can also temporarily remove the MySQL inbound rule altogether. Restore the rule immediately after recording the failure.

## 8. Troubleshooting decision tree

Use this sequence during the incident and in interviews.

Start: `/db-check` fails

Record the time, HTTP status and application logs

1\. Does the endpoint resolve?

- No → verify the hostname, endpoint, VPC DNS settings and database status.
- Yes → test TCP port 3306.

2\. Does TCP connect?

- Timeout → inspect security groups, subnet routes, NACLs and network reachability.
- Connection refused → verify the destination, port and database availability; this is less typical for a filtered RDS connection.
- Success → test database authentication.

3\. Can MySQL authenticate and execute queries?

- Access denied → check username, password, authentication configuration and host-specific grants.
- Permission error → check the user's database privileges.
- Query succeeds → investigate application configuration, connection pooling, timeouts and application logs.

For deeper network troubleshooting, use [AWS Reachability Analyzer](https://console.aws.amazon.com/vpc/home?region=ap-south-1\&utm_source=chatgpt.com#ReachabilityAnalyzer:) where supported, and check VPC Flow Logs if enabled. These can help identify network paths and filtering, but they do not replace checking MySQL credentials and grants.

## 9. Monitoring strategy

Configure basic monitoring while the lab is running.

| AWS service            | What to monitor                                             | Why                                                    |
| ---------------------- | ----------------------------------------------------------- | ------------------------------------------------------ |
| CloudWatch EC2 metrics | CPU utilization and status checks                           | Detect unhealthy or overloaded application instances   |
| RDS CloudWatch metrics | `DatabaseConnections`, `CPUUtilization`, `FreeStorageSpace` | Identify connection pressure and resource problems     |
| RDS events             | Database status and maintenance events                      | Detect database lifecycle or availability issues       |
| Application logs       | Failed DB connections and exceptions                        | Identify authentication and application-level failures |
| VPC Flow Logs          | Rejected traffic, where available                           | Help investigate network filtering                     |

Start with a CloudWatch alarm for RDS CPU utilization and a low-storage alarm. For connection failures, publish a custom application metric or configure log-based metric filters; ordinary RDS metrics do not automatically tell you why an individual login failed.

## 10. Clean up the lab

This step matters because RDS instances, EC2 instances, storage, and networking components can incur charges.

1. Stop the Flask application with `Ctrl+C`.
2. Delete the RDS database after confirming you no longer need its data. For a disposable lab, disable final snapshots only if you are comfortable permanently discarding the data.
3. Terminate `rds-lab-app-01`.
4. Remove lab-only security groups after they are no longer attached to resources.
5. Delete any additional resources you created specifically for the lab and review your AWS billing dashboard.

Do not delete a VPC or subnet that is shared with other projects.

## Your lab completion checklist

Progress

### 0/7

EC2 application launches and responds on /health

RDS MySQL is Available and not publicly accessible

Successful DNS resolution and TCP 3306 connection

Successful SQL authentication and query

Reproduced a security group timeout and fixed it

Tested an authentication failure separately

Recorded CloudWatch metrics and cleaned up resources

Interview-ready conclusion: “I isolate the issue layer by layer: DNS resolution, TCP connectivity, database authentication, and SQL permissions. I verify the RDS endpoint and status, then inspect the VPC path and security group rules. I keep RDS private and allow port 3306 only from the application security group. After correcting the root cause, I retest the database connection and the application endpoint, verify logs and monitoring, and document the incident.”

Useful official references: [RDS MySQL connectivity guide](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_GettingStarted.CreatingConnecting.MySQL.html?utm_source=chatgpt.com) and [EC2-to-RDS web application tutorial](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/TUT_WebAppWithRDS.html?utm_source=chatgpt.com).

Note: I can't provision resources inside your AWS account directly from this chat. The steps above create the actual AWS lab when you execute them in your account.
