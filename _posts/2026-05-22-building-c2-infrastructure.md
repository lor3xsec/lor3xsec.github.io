---
title: "Building a Resilient C2 Infrastructure: AWS, Nginx and Azure API Management"
date: 2026-05-22
categories: [Red Team, Infrastructure]
tags: [c2, redteam,]
excerpt: "The objective of this research is to conceal a C2 server from analysts and defenders by leveraging cloud services as a facade."
---

## Goal

The objective of this research is to conceal a C2 server from analysts and defenders by
leveraging cloud services as a facade. The approach combines an AWS-hosted backend with
Azure API Management as a redirection layer — backed by Microsoft infrastructure, which makes
traffic appear legitimate to most detection tools.

## Architecture Overview

The setup consists of three main components:

- **Backend**: EC2 instance on AWS running Havoc C2 + Nginx as reverse proxy
- **Domain**: registered via AWS Route53 and linked to the EC2 instance
- **Redirector**: Azure API Management acting as a fronting layer via `azure-api.net`

---

## Step 1: Setting Up the AWS Backend

### Launch an EC2 Instance

From the AWS console, navigate to EC2 → Instances → Launch Instances and configure:

- OS: Ubuntu
- SSH key: RSA, `.pem` format
- Security group: allow SSH only from your IP
- Storage: 25GB (up to 30GB free tier)

### Register a Domain on Route53

Navigate to Route53 → Register Domain, search for an available domain and complete checkout.

Once registered, create a **Hosted Zone** for the domain and add two records:

- **A Record** → public IP of your EC2 instance
- **NS Record** → nameservers from Route53 → Registered Domains → your domain

### Firewall Rules

Add inbound rules on the EC2 security group for HTTP (port 80) and HTTPS (port 443) from `0.0.0.0/0`.

---

## Step 2: SSL Certificate and Nginx Configuration

Connect to the instance via SSH and install the required packages:

```bash
apt-get install nginx apache2 python3-certbot-apache
```

Generate the Let's Encrypt certificate:

```bash
sudo certbot certonly --authenticator standalone -d YOUR-ROUTE53-DOMAIN \
  --agree-tos --register-unsafely-without-email
```

Remove Apache:

```bash
sudo apt-get purge apache2 apache2-utils
```

Create the Nginx configuration:

```bash
sudo nano /etc/nginx/sites-available/nginx.conf
```

Paste the following, replacing the domain and private IP with your values:

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate /etc/letsencrypt/live/YOUR-DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/YOUR-DOMAIN/privkey.pem;

    location /prova/test/ {
        if ($http_user_agent != 'Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36') {
            return 302 https://portal.azure.com/;
        }
        proxy_pass https://YOUR-EC2-PRIVATE-IP:10443;
    }
}
```

> The User-Agent check adds a defensive layer: any request that does not match the expected
> agent gets redirected to the Azure portal, hiding the C2 backend from analysts.

Enable the config and restart Nginx:

```bash
ln -s /etc/nginx/sites-available/nginx.conf /etc/nginx/sites-enabled/nginx_server.conf
sudo service nginx stop && sudo service nginx start
sudo nginx -t
```

---

## Step 3: Installing Havoc C2

Follow the official installation guide at [havocframework.com/docs/installation](https://havocframework.com/docs/installation),
then build and run the TeamServer:

```bash
git clone https://github.com/HavocFramework/Havoc.git
cd Havoc/teamserver
go mod download golang.org/x/sys
go mod download github.com/ugorji/go
cd ..
make ts-build
./havoc server --profile ./profiles/havoc.yaotl -v --debug
```

Build and run the client on your local Kali machine:

```bash
make client-build
./havoc client
```

Connect to the TeamServer using the public IP of your EC2 instance on port **40056**.
Remember to add a firewall rule allowing only your IP on that port.

---

## Step 4: Azure API Management as Redirector

Azure API Management (APIM) allows you to expose your C2 backend through a legitimate
`azure-api.net` domain, making C2 traffic blend with normal Microsoft API traffic.

### Create the APIM Instance

1. In the Azure Marketplace, search for **API Management** and click Create
2. Fill in: resource group, region, organization name, admin email
3. Select the **Developer** pricing tier for lab purposes
4. Wait 30-60 minutes for provisioning
5. Note down your **Gateway URL** (e.g. `yourlab.azure-api.net`)

### Configure the API

Once created, go to **APIs → Add API → HTTP** and configure:

- **Display name**: forwarder
- **Web Service URL**: your Route53 domain (e.g. `https://your-domain.com`)
- **API URL suffix**: leave empty
- **URL scheme**: HTTPS

Add a POST operation:

- Select **Add Operation**
- Set the URI to match what you configured in Nginx (e.g. `/prova/test/`)

### Disable Subscription Requirement

Go to your API → **Settings** → uncheck **Subscription Required**. This is necessary for the
beacon to connect without an API key.

### Configure the Havoc Listener

```
Listeners {
    Http {
        Name     = "azure-api"
        Hosts    = ["yourlab.azure-api.net"]
        HostBind = "YOUR-EC2-PRIVATE-IP"
        PortBind = 10443
        PortConn = 443
        Secure   = true
        UserAgent = "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/96.0.4664.110 Safari/537.36"
        Uris     = ["/prova/test/"]
        Headers  = []
        Response { Headers = [] }
    }
}
```

> **Important**: the URI and User-Agent in the listener must exactly match what you configured
> in Nginx. Any mismatch will cause the request to be redirected to the Azure portal instead of
> reaching the C2.

---

## Final Notes

This setup routes C2 traffic through `azure-api.net`, a Microsoft-owned domain that is
commonly allowlisted in enterprise environments. Combined with the Nginx User-Agent filter,
the infrastructure provides a solid layer of concealment for red team operations.

For multi-listener setups, create separate Havoc profiles and launch the TeamServer with
the desired one:

```bash
./havoc server --profile ./profiles/azure-api.yaotl -v
```
