# AWS Security Agent Lab — OWASP Juice Shop

Deploy [OWASP Juice Shop](https://owasp.org/www-project-juice-shop/) on AWS EC2 using CloudFormation. This lab demonstrates **AWS security agent capabilities** against a deliberately vulnerable web application in an isolated environment.

## Architecture

```
Internet
   │
   ├─ Cloudflare DNS (optional) ──► A record → Elastic IP
   │
   └─ EC2 (Amazon Linux 2023)
         ├─ nginx :80  ──► Juice Shop container :3000 (localhost only)
         └─ Elastic IP (stable public address)
```

| Component | Purpose |
|-----------|---------|
| **EC2** | Runs Docker + nginx on Amazon Linux 2023 |
| **Elastic IP** | Stable public IP for DNS and access after reboot |
| **Security Group** | Allows SSH (22) and HTTP (80) from the internet |
| **nginx** | Reverse proxy on port 80 → Juice Shop on 127.0.0.1:3000 |
| **Cloudflare** | Optional DNS for a custom domain |

## Prerequisites

- AWS account with permissions for EC2, CloudFormation, and Elastic IP
- EC2 key pair in the **same region** as the stack
- A **public subnet** with a route to an Internet Gateway (`0.0.0.0/0 → igw-...`)
- (Optional) Domain managed in Cloudflare

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| `KeyName` | Yes | — | EC2 key pair name for SSH |
| `VpcId` | Yes | — | VPC containing your subnet (e.g. default VPC) |
| `SubnetId` | Yes | — | Public subnet ID |
| `DomainName` | No | *(empty)* | FQDN for nginx, e.g. `shop.example.com` |
| `InstanceType` | No | `t3.micro` | `t3.micro`, `t3.small`, or `t3.medium` |
| `AmiId` | No | `ami-08f44e8eca9095668` | Amazon Linux 2023 x86_64 AMI |

### Finding VPC and subnet IDs

1. **VPC** → **Subnets** → select your subnet
2. Copy **Subnet ID** and **VPC ID** from the details panel

Example:

- `SubnetId`: `subnet-xxxxxxxx`
- `VpcId`: `vpc-xxxxxxxx`

## Deploy via AWS Console

1. Open **CloudFormation** → **Create stack** → **With new resources**
2. **Upload a template file** → select `juice-shop.yaml`
3. **Stack name:** e.g. `juice-shop-lab`
4. Fill in parameters:
   - `KeyName` — your key pair
   - `VpcId` — your VPC ID
   - `SubnetId` — your public subnet ID
   - `DomainName` — optional, e.g. `shop.yourdomain.com`
5. Click **Next** through options (defaults are fine)
6. **Submit** and wait for status `CREATE_COMPLETE`
7. Open the **Outputs** tab and copy **ElasticIp** and **WebsiteURL**

Allow **2–3 minutes** after the stack completes for UserData to install Docker, pull the image, and start nginx.

## Cloudflare DNS setup

After the stack completes:

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com)
2. Select your domain → **DNS** → **Records** → **Add record**

| Field | Value |
|-------|-------|
| **Type** | `A` |
| **Name** | `shop` (for `shop.yourdomain.com`) |
| **IPv4 address** | **ElasticIp** from stack Outputs |
| **Proxy status** | **DNS only** (grey cloud) recommended for lab |

3. Save and wait a few minutes for DNS propagation
4. Open `http://shop.yourdomain.com`

## HTTPS with Cloudflare (optional)

The template serves HTTP on port 80. You can add HTTPS at the Cloudflare edge without changing the template:

1. Set the DNS record to **Proxied** (orange cloud)
2. Go to **SSL/TLS** → set mode to **Full**
3. Visit `https://shop.yourdomain.com`

Cloudflare terminates HTTPS; traffic from Cloudflare to your EC2 stays on HTTP port 80.

For end-to-end TLS on EC2, you would need to install a Cloudflare Origin Certificate and configure nginx for HTTPS. That is not included in the current template.

## Stack outputs

| Output | Description |
|--------|-------------|
| `InstanceId` | EC2 instance ID |
| `ElasticIp` | Stable public IP — use for Cloudflare A record |
| `WebsiteURL` | `http://<domain>` or `http://<elastic-ip>` |
| `CloudflareDns` | Reminder of the A record to create |
| `SSHCommand` | SSH command using `ec2-user` |

## SSH access

```bash
ssh -i your-key.pem ec2-user@<ElasticIp>
```

Amazon Linux 2023 uses **`ec2-user`**, not `ubuntu`.

## Verify Juice Shop is running

After the stack completes, SSH into the instance and check:

```bash
sudo docker ps
sudo systemctl status nginx
curl -I http://127.0.0.1:3000
curl -I http://127.0.0.1
```

## Cleanup

Deleting the local `juice-shop.yaml` file does **not** remove AWS resources.

To tear down everything the stack created:

1. **CloudFormation** → select your stack → **Delete**
2. Wait for status `DELETE_COMPLETE`

This removes the EC2 instance, Elastic IP, and security group.

Manually remove the Cloudflare DNS record if you created one.

## Security warning

OWASP Juice Shop is **intentionally vulnerable**. This template exposes SSH and HTTP to the internet (`0.0.0.0/0`).

- Use only in a **disposable lab account**
- Do not deploy in production or with sensitive data
- Delete the stack when finished
- Consider restricting SSH to your IP if you modify the security group

## Files

| File | Description |
|------|-------------|
| `juice-shop.yaml` | CloudFormation template |
| `README.md` | This guide |
