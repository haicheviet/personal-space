---
title: "Tailscale for your friendly work VPN"
date: 2022-10-22T07:33:29+00:00
lastmod: 2022-10-23T07:33:29+00:00
description: "This article shows how to use Tailscale and what can we achieve in Tailscale platform."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Tailscale", "VPN", "Work Tool", "Bastion Host", "Cloud Security", "Subnet Router"]
categories: ["VPN"]


---

The most crucial practice in cloud security is private access and a control panel for managing users with whom service users can access. However, too much control in private access and administration causes the developer's coding experience to be exponentially low. It makes it hard for new developers to get a hands on the service they need without contacting the DevOps team. Especially more services require more maintenance from both sides, and the resource for keeping everything in standard security practice requires a lot in the workforce and net cloud cost. Tailscale is the new VPN service that offers to solve this challenge at a competitive cost. Tailscale makes the devices, applications and network you own accessible anywhere in the world, securely and effortlessly. Today we will experience Tailscale and some guide setup for most developer needs.

<!--more-->

## My goal

Currently, my working environment requires Bastion host and VPN to access the dev server and production database. The setup is simple at first, but as the team and service grow, the complexity of the setup is growing respectively. Hands-on new dev is never easy, and despite I already set up [Dev microVM](https://haicheviet.com/cluster-vm-with-firecracker/) with all the necessary environment and packages, the network config is the most confusing and complicated to debug for junior dev. The current process's problems can be listed as:

- Some [HPC servers](https://en.wikipedia.org/wiki/High-performance_computing) are in the office building, and team members can only access the servers via the office network. Emailing the administrator network for the working VPN is hostile, and even after requesting for years, my request has not been fulfilled cause of insufficient member requests :'(.
- I have tried Cloudflare Tunnel and Ngrok, but the setup is not instructive, and the service must be registered in the specific domain, but my only need is to access the service via email, nothing more.
- AWS cloud service of our platform is mainly in the private subnet. For the team members to access the cloud database, they must access via Bastion host or VPN.
- Bastion host will get complicated quickly as the service grows and require nested ssh config to memorize all the service IPs.
- VPN is ok, but you can only use one VPN at a time and can not access another network, such as internal HPC servers or other VPN networks.

My cram about the network configuration has been built up for years, and seeing new dev frustration when debugging local is not a pleasant sight. Tailscale is my new solution for solving all the problems I listed. The team experience has been outstanding, and everything can be accessed and configured easily.


## Tailscale hands on

Login tailscale is relative easy, go to this https://login.tailscale.com/start to register. But currently taiscale only support creat account with certain SSO identity providers (eg Goole or Microsoft). If you need other support providers, please follow [this instruction](https://tailscale.com/kb/1013/sso-providers/).

![Login page](tailscale-login.webp "Login page")

After create account and login, you will be navigate to admin page with the listing service.

![Admin page](admin-page.webp "Admin page")

The above image is the listing devices I am using; each device has its IPv4 address that can be accessed when logging into Tailscale.

## Tailscale device register

Tailscale helps you connect your devices together. To regsiter your device to tailscale cluster, download tailscale clien both in your host and client device. Tailscale works seamlessly with Linux, Windows, macOS, Raspberry Pi, Android, Synology, and more. Download Tailscale and log in on the device.

https://tailscale.com/download

Here is how I install tailscale in Fedora server:

### Step 1: Install Tailscale

```bash
$ curl -fsSL https://tailscale.com/install.sh | sh
```

### Step 2: Connect my device to Tailscale

Via webrowser

```bash
$ sudo tailscale up

To authenticate, visit:

      https://login.tailscale.com/a/abcde
```

Via key in [your setting keys](https://login.tailscale.com/admin/settings/keys)

```bash
$ sudo tailscale up --authkey tskey-abcdef1432341818
```

### Step 3: Verify your connection

Check that you can ping your new device from your personal Tailscale machine (Windows, macOS, etc). You can find the Tailscale IP in the admin console, or by running this command on the new register machine.

```bash
$ tailscale ip -4
100.83.201.24 # Your IPv4 in tailscale
```

Ping the new register device in your client machine

```bash
$ ping 100.83.201.24

Pinging 100.83.201.24 with 32 bytes of data:
Reply from 100.83.201.24: bytes=32 time=77ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
Reply from 100.83.201.24: bytes=32 time=32ms TTL=255
```

The magic of Tailscale happens when it’s installed on multiple devices. Add more of your devices and share Tailscale with your peers to grow your private network. 

Add more machines to your network by repeating step 2 or by [inviting others to join your network](https://tailscale.com/kb/1064/invite-team-members/).

Even more, you can sharing between device and team member via [TailDrop](https://tailscale.com/kb/1106/taildrop/#enabling-taildrop-for-your-network)

{{< video src="https://tailscale.com/kb/1106/taildrop/taildrop.mp4" loop=true autoplay=true >}}

## Tailscale as a network layer

Tailscale support `subnet router` (previously called a relay node or relaynode) to access these devices from Tailscale. **Subnet routers act as a gateway**, relaying traffic from your Tailscale network onto your physical subnet. Subnet routers respect features like access control policies, which make it easy to migrate a large network to Tailscale without installing the app on every device.

![Subnet router](subnets.webp "Subnet router")

Setting up a subnet router is relatively easy and you can read [this guide](https://tailscale.com/kb/1019/subnets/#setting-up-a-subnet-router) for more specifics. Below is the detailed process that I set up in the AWS cloud.

### Step 1: Create an EC2 instance router

First, create an EC2 instance running Amazon Linux on either x86 or ARM. Tailscale produces Linux packages containing binaries for both architectures, and the AWS ARM instances are very cost effective. When setting the security policy, allow UDP port 41641 to ingress from any source. This will enable direct connections, to minimize latency.

![Security Group](security-group.webp "Security Group")

### Step 2: Configure tailscale subnet router

Then ssh to the system and follow the steps to [install Tailscale on Amazon Linux](https://tailscale.com/kb/1052/install-amazon-linux-2/) and configure subnet routing. When running `tailscale up`, pass your VPC subnet address to `--advertise-routes`.

![VPC subnet address](subnet-address.webp "VPC subnet address")

As the above image, the subnet range VPC is 10.0.0.0/16:

```bash
$ tailscale up --advertise-routes=10.0.0.0/16 --accept-dns=false
```

{{< admonition tip >}}

For EC2 instances it is generally best to let Amazon handle the DNS configuration, not have Tailscale override it, so we added --accept-dns=false.

{{< /admonition >}}


### Step 3: Enable subnet router in admin

Check your instance router is successfully configured

```bash
$ tailscale ip -4
100.83.201.24
```

This step is not required if using ***autoApprovers***.

Open the machines page in the [admin console](https://login.tailscale.com/admin/machines), and locate the device that advertised subnet routes. You can look for the Subnets badge in the machines list. Using the `...` icon at the end of the table, select `Edit route settings`. This will open up the Edit route settings panel.

![Edit route settings](subnet-route-setting.webp "Edit route settings")


Approve each route individually by clicking the toggle to the left of the route.

{{< admonition tip >}}

You may prefer to disable key expiry on your server to avoid having to periodically reauthenticate.

{{< / admonition >}}


### Step 4: Use your subnet routes from other machines

You can traceroute and SSH private instances (10.0.26.12) behind the subnet route (100.83.201.24).

```bash

$ traceroute 10.0.26.12
traceroute to 10.0.26.12 (10.0.26.12), 64 hops max, 52 byte packets
 1  100.83.201.24 (100.83.201.24)  27.135 ms  17.935 ms  18.342 ms
 2  10.0.26.12 (10.0.26.12)  19.396 ms  18.852 ms  18.364 ms

$ ssh /path/to/private_key ec2-user@10.0.26.12
```

If RDS instances are created without public accessibility, the hostname is resolved with a private IP address outside the VPC. This IP address belongs to the advertised routes.

You can access RDB instances from the Tailscale network in the same way.

```bash
$ dig +short database.xxx.ap-southeast-1.rds.amazonaws.com
10.0.3.176

$ mysqlsh --uri=admin@database.xxx.ap-southeast-1.rds.amazonaws.com:3306
...
 MySQL  database-1.c75dvgjlkohb.ap-southeast-1.rds.amazonaws.com:3306 ssl  JS >
 
```

{{< admonition tip >}}

The Windows, macOS, Android, iOS, etc clients all accept advertised routes by default, but Linux clients need to use `tailscale up --accept-routes=true` to use the routes being advertised by the subnet router in AWS.

{{< /admonition >}}

## Some afterthought

- Using Tailscale has been so blessed that I can not return to the traditional way anymore. Every service can be accessed by its private domain, no more mess up the environment cause the network switch or port is already in use.
- Tailscale is a paid service, and you must depend on Tailscale to manage the admin server. Thankfully, Taiscale also provides a solution to open-source hosting [Headscale](https://github.com/juanfont/headscale), and you can configure all the admin server on your own.
- For the final wrap-up, you can use [Terraform](https://registry.terraform.io/modules/hardfinhq/tailscale-subnet-router/aws/latest?tab=resources) to manage Tailscale and automate all the processes I listed above.
