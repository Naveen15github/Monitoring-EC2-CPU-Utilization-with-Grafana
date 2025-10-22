# Monitoring-EC2-CPU-Utilization-with-Grafana
# 🚀 Monitoring EC2 CPU Utilization with Grafana (CloudWatch Integration) 🖥️📈


Overview ✨
-----------
This project shows a full, production-ready implementation to deploy Grafana on an AWS EC2 instance and integrate it with CloudWatch to monitor EC2 CPU utilization (and other metrics). The README below is a single, copy/pasteable playbook you can run from the command line to reproduce everything end-to-end: provisioning, OS setup, Grafana install, CloudWatch integration, dashboard provisioning, image assets, TLS, hardening, alerting, and cleanup.

This README includes:
- Full implementation steps (AWS CLI + EC2 + IAM + Grafana + CloudWatch)
- Exact commands for Amazon Linux 2 and Ubuntu
- Grafana provisioning files (datasource + dashboard) examples
- Security & hardening guidance, automated provisioning ideas, and troubleshooting

Prerequisites ✅
---------------
- AWS account with permissions to create EC2, IAM roles/policies, Security Groups and S3 (optional).  
- AWS CLI v2 installed and configured (`aws configure`) with a user having the required permissions.  
- jq installed (recommended) for parsing CLI output.  
- SSH client (for EC2 access).  
- A local git repo where you want to store README and images (optional).  
- (Optional) Terraform / CloudFormation if you want to automate infra provisioning later.

High-level architecture
-----------------------
EC2 (Grafana) --uses--> IAM Instance Profile --> CloudWatch API  
Grafana pulls CloudWatch metrics and renders dashboards on port 3000. Optionally, an NGINX reverse proxy / ALB provides HTTPS and authentication.
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)
![Alt text describing the image](assets/images/image1.png)


Full implementation steps (end-to-end)
-------------------------------------

Part A — Create keypair & security group (from your workstation)
----------------------------------------------------------------
1. Choose region and variables:
```bash
export AWS_REGION=ap-south-1
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" --query "Vpcs[0].VpcId" --output text)
export KEY_NAME="grafana-key-$(date +%s)"
export SG_NAME="grafana-sg-$(date +%s)"
export YOUR_IP_CIDR="203.0.113.5/32"   # <-- replace with your IP
```

2. Create an SSH key pair (store the private key locally):
```bash
aws ec2 create-key-pair --key-name "$KEY_NAME" --query 'KeyMaterial' --output text > ${KEY_NAME}.pem
chmod 600 ${KEY_NAME}.pem
```

3. Create Security Group and open ports (22 for SSH, 3000 for Grafana, 80/443 for reverse proxy if used):
```bash
SG_ID=$(aws ec2 create-security-group --group-name "$SG_NAME" --description "Grafana SG" --vpc-id "$VPC_ID" --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 22 --cidr $YOUR_IP_CIDR
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 3000 --cidr $YOUR_IP_CIDR
# optional HTTP/HTTPS for reverse proxy
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 80 --cidr $YOUR_IP_CIDR
aws ec2 authorize-security-group-ingress --group-id $SG_ID --protocol tcp --port 443 --cidr $YOUR_IP_CIDR
echo "Security Group $SG_ID created"
```

Part B — Launch EC2 instance (Amazon Linux 2 example)
-----------------------------------------------------
1. Select an AMI (Amazon Linux 2) and subnet (use default)
```bash
AMI_ID=$(aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2 --region $AWS_REGION --query 'Parameters[0].Value' --output text)
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query "Subnets[0].SubnetId" --output text)
INSTANCE_TYPE=t3.small
```

2. Launch instance:
```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type "$INSTANCE_TYPE" \
  --key-name "$KEY_NAME" \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --count 1 \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Grafana-Server}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "Launched EC2 instance: $INSTANCE_ID"
aws ec2 wait instance-running --instance-ids $INSTANCE_ID
EC2_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Public IP: $EC2_PUBLIC_IP"
```

Part C — Create IAM role + policy for CloudWatch read (recommended)
-------------------------------------------------------------------
Create a least-privilege policy for Grafana to read CloudWatch metrics and describe EC2.

1. Save this policy locally as `cw-read-policy.json`:
```json
{
  "Version":"2012-10-17",
  "Statement":[
    {
      "Effect":"Allow",
      "Action":[
        "cloudwatch:ListMetrics",
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:GetMetricWidgetImage",
        "cloudwatch:DescribeAlarms",
        "ec2:DescribeInstances",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents"
      ],
      "Resource":"*"
    }
  ]
}
```

2. Create role and instance profile and attach policy:
```bash
ROLE_NAME="GrafanaCloudWatchRole-$(date +%s)"
TRUST_POLICY='{
  "Version":"2012-10-17",
  "Statement": [{"Effect": "Allow","Principal": {"Service": "ec2.amazonaws.com"},"Action": "sts:AssumeRole"}]
}'

aws iam create-role --role-name "$ROLE_NAME" --assume-role-policy-document "$TRUST_POLICY"
aws iam put-role-policy --role-name "$ROLE_NAME" --policy-name CloudWatchReadPolicy --policy-document file://cw-read-policy.json
aws iam create-instance-profile --instance-profile-name "$ROLE_NAME"
aws iam add-role-to-instance-profile --instance-profile-name "$ROLE_NAME" --role-name "$ROLE_NAME"
# Associate with EC2 instance
aws ec2 associate-iam-instance-profile --instance-id $INSTANCE_ID --iam-instance-profile Name=$ROLE_NAME
```

Note: You can attach the role during launch by specifying `--iam-instance-profile Name=...` in `run-instances`.

Part D — SSH into the EC2 instance
----------------------------------
```bash
ssh -i ${KEY_NAME}.pem ec2-user@${EC2_PUBLIC_IP}
```

Part E — Install Grafana (Amazon Linux 2)
-----------------------------------------
Run on the EC2 instance (as ec2-user or root via sudo):

```bash
# Update OS
sudo yum update -y

# Optional utilities
sudo yum install -y git jq

# Add Grafana repo
cat <<'EOF' | sudo tee /etc/yum.repos.d/grafana.repo
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
EOF

# Install Grafana OSS (or grafana-enterprise if you have a license)
sudo yum install -y grafana

# Start and enable Grafana
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server --no-pager
```

Ubuntu/Debian variant (if your EC2 uses Ubuntu):
```bash
sudo apt-get update -y
sudo apt-get install -y apt-transport-https software-properties-common wget
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
echo "deb https://packages.grafana.com/oss/deb stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt-get update -y
sudo apt-get install -y grafana
sudo systemctl daemon-reload
sudo systemctl enable --now grafana-server
sudo systemctl status grafana-server --no-pager
```

Verify Grafana is listening (on the instance):
```bash
sudo ss -ltnp | grep 3000
```

Part F — (Optional) Install & configure CloudWatch Agent for host-level metrics
-------------------------------------------------------------------------------
If you want more granular system metrics (disk, mem, processes), install the CloudWatch agent:

```bash
# Amazon Linux 2
wget https://s3.amazonaws.com/amazoncloudwatch-agent/amazon_linux/amd64/latest/amazon-cloudwatch-agent.rpm
sudo rpm -U ./amazon-cloudwatch-agent.rpm

sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-config-wizard
# Edit config if needed: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json -s
```

Part G — Secure Grafana (basic hardening)
-----------------------------------------
1. Change default admin password at first login (http://EC2_PUBLIC_IP:3000). Default is admin/admin.  
2. Consider an NGINX reverse proxy + TLS or put Grafana behind an ALB with ACM certificate. Example quick NGINX with Let's Encrypt (high level):

Install nginx and certbot, configure proxy_pass to http://localhost:3000, enable basic auth or allow only your IP, and use certbot for TLS. (This is shown in the "Advanced TLS" section below.)

Part H — Configure Grafana CloudWatch data source (two options)
----------------------------------------------------------------
Recommended: use IAM Instance Profile (no keys in Grafana)

- UI method (Grafana):
  1. Login to Grafana at http://EC2_PUBLIC_IP:3000 (admin/admin).
  2. Configuration -> Data Sources -> Add data source -> CloudWatch.
  3. Auth Provider:
     - For instance profile: Set "Auth Provider" to "Credentials (Access & secret key)" and choose "Use IAM Role" or leave credentials blank and Grafana will use instance metadata (Grafana version dependent). Alternatively select "Default" or "Credentials" -> "SigV4" depending on Grafana version.
  4. Set Default Region and Save & Test -> "Data source is working".

- Provisioning method (auto):
  Create a provisioning file: grafana/provisioning/datasources/cloudwatch.yaml

Example (place on the EC2 Grafana server under /etc/grafana/provisioning/datasources/cloudwatch.yaml):
```yaml
apiVersion: 1

datasources:
  - name: CloudWatch
    type: cloudwatch
    access: proxy
    isDefault: true
    jsonData:
      authType: default  # 'default' uses env/instance metadata
      defaultRegion: ap-south-1
      assumeRoleArn: ""
    secureJsonData: {}
    version: 1
    editable: true
```
Notes:
- `authType: default` will let Grafana use EC2 instance profile. In some versions you must use `authType: credentials` with empty keys to force metadata usage. If not working, use the UI to test and experiment.

Part I — Provision a sample CPU dashboard (auto-import)
-------------------------------------------------------
You can provision dashboards so Grafana creates them at startup. Example:

1. Create a dashboard JSON (minimal example saved to grafana/provisioning/dashboards/cpu-dashboard.json). Example (very small fragment showing single CPU graph panel):

```json
{
  "id": null,
  "uid": "ec2-cpu-uid",
  "title": "EC2 CPU Utilization (AutoProvisioned)",
  "panels": [
    {
      "type": "graph",
      "title": "CPU utilization per instance [%]",
      "gridPos": { "h": 8, "w": 24, "x": 0, "y": 0 },
      "targets": [
        {
          "namespace": "AWS/EC2",
          "metricName": "CPUUtilization",
          "dimensions": { "InstanceId": "$__all" },
          "statistics": [ "Average" ],
          "period": 300
        }
      ]
    }
  ],
  "schemaVersion": 27,
  "version": 1
}
```

2. Create provisioning YAML to load dashboards automatically (place under /etc/grafana/provisioning/dashboards/dashboards.yaml):

```yaml
apiVersion: 1
providers:
  - name: 'EC2 CPU dashboards'
    orgId: 1
    folder: 'AutoProvisioned'
    type: file
    disableDeletion: false
    options:
      path: /var/lib/grafana/dashboards
```

3. Copy your dashboard JSON to `/var/lib/grafana/dashboards/` (create directory) and restart Grafana:
```bash
sudo mkdir -p /var/lib/grafana/dashboards
sudo cp ./grafana/provisioning/dashboards/cpu-dashboard.json /var/lib/grafana/dashboards/
sudo systemctl restart grafana-server
```

Part J — Create the dashboard via the Grafana UI (manual steps)
---------------------------------------------------------------
If you prefer the UI:
1. Create -> Dashboard -> Add new panel.  
2. Data source: CloudWatch.  
3. Namespace: AWS/EC2.  
4. Metric: CPUUtilization.  
5. Dimension: InstanceId (choose instances).  
6. Statistic: Average (or Maximum if you prefer).  
7. Period: e.g., 60 or 300.  
8. Save dashboard.

Part K — Test: Generate CPU load to see metrics
------------------------------------------------
On any EC2 instance you want to see CPU spikes for:

```bash
# simple load for 60 seconds
yes > /dev/null & sleep 60; kill $!

# alternative using stress (install if needed)
sudo yum install -y stress  # or apt-get on Ubuntu
stress --cpu 2 --timeout 60
```

Refresh Grafana dashboard and observe spikes.

Part L — Add alerting in Grafana
--------------------------------
- In Grafana create an Alert Rule for your CPU panel:
  - Condition: Average(cpu) > 80 for 5 minutes
  - Add contact points: Slack, Email, or PagerDuty
- For Grafana OSS pre-9 versions: use panel-based alerting; for Grafana 9+ use unified alerting and contact points.

Part M — Optional: Front Grafana with NGINX and Let's Encrypt TLS (basic)
-------------------------------------------------------------------------
1. Install nginx & certbot:
```bash
sudo yum install -y nginx
# or on Ubuntu: sudo apt-get install -y nginx python3-certbot-nginx
sudo systemctl enable --now nginx
```

2. Example Nginx server block (reverse proxy to Grafana on localhost:3000):
Create `/etc/nginx/conf.d/grafana.conf`:
```nginx
server {
  listen 80;
  server_name grafana.example.com;

  location / {
    proxy_pass http://127.0.0.1:3000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
  }
}
```

3. Obtain TLS cert with certbot:
```bash
sudo certbot --nginx -d grafana.example.com
```

4. After certbot, it will update nginx conf and Grafana will be served over https.

Part N — Add images and icons to the repo (images 1–8)
-----------------------------------------------------
From the repo root:
```bash
mkdir -p assets/images assets/icons
# copy local files into assets/images, rename to image1.png ... image8.png
cp /local/path/to/screenshot1.png assets/images/image1.png
# ... repeat for images 2..8

# add icons (optional)
curl -L -o assets/icons/grafana-icon.svg https://raw.githubusercontent.com/grafana/grafana/main/public/img/grafana_icon.svg
curl -L -o assets/icons/aws-architecture-icons.zip "https://d1.awsstatic.com/architecture-icons/Architecture-Icons.zip"
unzip assets/icons/aws-architecture-icons.zip -d assets/icons/aws-raw
# pick EC2/CloudWatch svgs and place in assets/icons/
```

Optimize images (optional):
```bash
# Install ImageMagick if needed
sudo apt-get install -y imagemagick   # or sudo yum install -y ImageMagick
for f in assets/images/*.png; do
  convert "$f" -resize 1200x -quality 85 "assets/images/opt-$(basename $f)"
done
# inspect and replace originals if happy
```

Commit images:
```bash
git add assets/images assets/icons
git commit -m "chore: add screenshots (images 1-8) and icons"
git push origin main
```

Part O — Troubleshooting checklist
----------------------------------
- Grafana not starting: `sudo journalctl -u grafana-server -f` and `sudo systemctl status grafana-server`  
- Port 3000 not reachable: check EC2 security group and `ss -ltnp | grep 3000` and OS firewall (`sudo iptables -L` or `sudo ufw status`)  
- CloudWatch data errors: ensure IAM role attached and policy includes cloudwatch:GetMetricData and ec2:DescribeInstances  
- No metrics: confirm CloudWatch metrics exist for that instance (CloudWatch default provides CPUUtilization) and that region matches  
- Provisioning files not applied: ensure provisioning YAML syntax is correct and files are located under `/etc/grafana/provisioning/*` and dashboards in configured path

Part P — Cleanup to avoid charges
---------------------------------
```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID
# Delete security group
aws ec2 delete-security-group --group-id $SG_ID
# Remove IAM role & instance profile (careful)
aws iam remove-role-from-instance-profile --instance-profile-name "$ROLE_NAME" --role-name "$ROLE_NAME"
aws iam delete-instance-profile --instance-profile-name "$ROLE_NAME"
aws iam delete-role-policy --role-name "$ROLE_NAME" --policy-name CloudWatchReadPolicy
aws iam delete-role --role-name "$ROLE_NAME"
# Remove key pair
aws ec2 delete-key-pair --key-name "$KEY_NAME"
rm -f ${KEY_NAME}.pem
```

Advanced / Optional automation ideas
-----------------------------------
- Terraform or CloudFormation to deploy EC2 + SG + IAM + Instance Profile + userdata that installs Grafana and copies provisioning files.  
- Use a configuration management tool (Ansible) to manage Grafana packages/configs across instances.  
- Use Grafana provisioning to automate datasources and dashboards for consistent deployments.  
- Store Grafana dashboards and provisioning files in the repo and mount them into the container or copy them during startup.

Example Grafana datasource provisioning (repeat)
------------------------------------------------
Place this YAML into `/etc/grafana/provisioning/datasources/cloudwatch.yaml` to auto configure CloudWatch datasource (edit for your Grafana version):

```yaml
apiVersion: 1
datasources:
  - name: CloudWatch
    type: cloudwatch
    access: proxy
    isDefault: true
    jsonData:
      authType: default
      defaultRegion: ap-south-1
      assumeRoleArn: ""
    secureJsonData: {}
```

Example minimal dashboard JSON (for automatic import)
-----------------------------------------------------
See earlier `cpu-dashboard.json` example. Keep in `/var/lib/grafana/dashboards/` or the configured path used in provisioning YAML.

Security & best practices
-------------------------
- Always use IAM roles (instance profile) instead of storing AWS keys in Grafana.  
- Restrict Security Group access to known IPs instead of 0.0.0.0/0.  
- Run Grafana behind an HTTPS reverse proxy (ALB/NGINX) with TLS (use ACM for ALB).  
- Use Grafana authentication (LDAP/OAuth/SAML) for multi-user setups.  
- Keep Grafana and OS patches up to date and monitor Grafana logs.  
- Rotate secrets and follow least privilege for IAM policies.

