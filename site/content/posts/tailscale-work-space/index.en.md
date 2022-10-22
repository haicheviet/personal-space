---
title: "Tailscale for workspace"
date: 2022-10-22T07:33:29+00:00
lastmod: 2022-10-22T07:33:29+00:00
description: "This article shows how to use Firecracker to create cluster VMs with container."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Firecracker", "Container", "QEMU", "Ignite"]
categories: ["Container"]


---

The most importance practice in cloud security is private access and control pannel for managage user that whom service that user can get accessed with. But too much control in private access and adminatration cause the developer's coding experience is exponential low and hard for new developer to get a hands on the service they need without contact to devops person. Especially more services require more maitaince and the resource for keeping everything is secrurity is the huge cost both in human resource and net cloud cost. Tailscale is the new VPN service that makes the devices and applications you own accessible anywhere in the world, securely and effortlessly. Today we will experienment tailscale and some guide setup for most of the developer need

<!--more-->

## My goal

Currently my working enviroment require both bastion host and VPN to access both the dev server and production database. The setup is simple as first but as the team and service grow, the complexity of the setup is growing respectively. Hands on new dev is never easy and despite I already set up [Dev VM](https://haicheviet.com/cluster-vm-with-firecracker/) that have all the enviroment and package, the network config is the most confuse and hard to debug for junior dev. My problems can be listing as:

- Some HPC servers is in the office building and team member can only access the server via office network. Emailing the adminatrator network for the working VPN is hastile and even after requesting for years, my request have not been fullfied cause not enough member requests :'(. I've try Cloudflare Tunnel and Ngrok but the setup is not instruitive and the service have to be register in domain but my only need is access service when login.

- AWS cloud service of our platform is mostly in private subnet. For the team members to access the cloud database, they have to access via bastion host or using VPN. But bastion host is really compliated as the service grow and require ssh config to memorize all the service IP. VPN is ok but when using VPN, you can not access to other network such as internal HPC servers or other VPN network.

My cram about the network configure is really build up for a year and seeing new dev frustation when debuging in local is really not good. Tailscale is my comeup solution for solving what I really need.


## Tailscale hands on
Login tailscale is relative easy, go to this [page](https://login.tailscale.com/start) to register. But currently taiscale only support creat account with certain SSO identity providers (eg Goole or Microsoft). If you need other support providers, please follow [this instruction](https://tailscale.com/kb/1013/sso-providers/)

<image here>

After create account and login, you will be navigate to admin page with the listing service 

![Admin page](admin-page.webp "Admin page")

In the above image is the listing devices that I'm using, each device have it own IPv4 address that can be access when login tailscale

## Tailscale device register

Tailscale helps you connect your devices together. To regsiter your device to tailscale cluster, download tailscale clien both in your host and client device. Tailscale works seamlessly with Linux, Windows, macOS, Raspberry Pi, Android, Synology, and more. Download Tailscale and log in on the device.

https://tailscale.com/download

Here is how I install tailscale in Fedora server:

### Step 1: Install the tail scale

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

## Tailscale as network layer

Tailscale support `subnet router` (previously called a relay node or relaynode) to access these devices from Tailscale. **Subnet routers act as a gateway**, relaying traffic from your Tailscale network onto your physical subnet. Subnet routers respect features like access control policies, which make it easy to migrate a large network to Tailscale without installing the app on every device.

![Subnet router](admin-page.webp "Subnet router")

Setting up a subnet router is realtive easy and you can read [this guide](https://tailscale.com/kb/1019/subnets/#setting-up-a-subnet-router) for more specific. Below is the detail process that I setup in AWS cloud

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

Check your instance router is succesfull configure

```bash
$ tailscale ip -4
100.83.201.24
```

This step is not required if using `autoApprovers`.

Open the machines page in the [admin console](https://login.tailscale.com/admin/machines), and locate the device that advertised subnet routes. You can look for the Subnets badge in the machines list. Using the `... icon` at the end of the table, select `Edit route settings`. This will open up the Edit route settings panel.

![Edit route settings](subnet-route-setting.webp "Edit route settings")


Approve each route individually by clicking the toggle to the left of the route.

{{< admonition tip >}}

You may prefer to disable key expiry on your server to avoid having to periodically reauthenticate

{{< / admonition >}}


### Step 4: Use your subnet routes from other machines

You can traceroute and SSH instances(10.0.26.12) behind the subnet route(100.83.201.24).

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

- Tailscale is a paid service and you have to depend on Tailscale to manage admin server. Thankfully taiscale also provide opensource hosting [Headscale](https://github.com/juanfont/headscale) solution and you can configure all the admin page on your own.
- For the final wrap up, you can use [terraform](https://registry.terraform.io/modules/hardfinhq/tailscale-subnet-router/aws/latest?tab=resources) to managgte tailscale and automate all the process I listed above.