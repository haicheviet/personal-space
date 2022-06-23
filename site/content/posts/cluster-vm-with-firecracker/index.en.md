---
weight: 5
title: "Cluster VM with firecracker"
date: 2022-06-23T21:57:40+08:00
lastmod: 2022-06-23T16:45:40+08:00
description: "This article shows how to use Firecracker to create cluster VMs with container."
resources:
- name: "featured-image"
  src: "featured-image.png"

tags: ["Firecracker", "Container", "Qemu", "Ignite"]
categories: ["Container"]


---

Recently after being leader position, I was asked join the recruiment process and onboarding new members. As the team and our project grow both in size and complexity, our document to hand on projects is complicated and only work in certain ubuntu and python version.
So to make sure the on boarding process is fast and reproducible, I have to come up a new plan to create a isolate enviroment for coding and less learning curve as possible

<!--more-->

## My goal: A isolate enviroment where every member get their own resource and custom preinstall package

Our project in Jobhopin combine multiple languages (Rust, Python, ...) and the process of create virtualenv is quiet large and only work in linux system. Here is sample of one Docker file

## why not use containers image?

Firstly I encourage team to use Docker and docker-compose to code and debug project but at nature of AI team that combine both engineer and scientist. The science team find it hard to debug in docker and took a lot of time for new memmber to learn and and make use of docker image.
I wanted to mimic a real production machine that the member has root access to – I wanted folks to be able to set sysctls, use nsenter, make iptables rules, configure networking with ip, run perf, basically literally anything.

## why not use virtual machine?

I've tried some vm vendor (Qemu and Vmware) to create per vm per member but too much problem in the process:

* VM boosting time is slow and image size is too large
* Lack of API and the snapshot vm have to be manual create without any reproduce code

I want our member only need to provide their credential with custom VM size and instantly launch a new virtual machine.

## Firecracker can start a VM in less than a second with base OCI container

Initially when I read about Firecracker being released, I thought it was just a tool for cloud providers to use that provide security rather than bare docker, but I didn’t think that it was something that I could directly use it to create a dev VM.

After a few reading information, I just shock with how fast Firecracker is in boosting VM [ref](https://lwn.net/Articles/775736)
> The VMM process starts up in around 12ms on AWS EC2 I3.metal instances. Though this time varies, it stays under 60ms. Once the guest VM is configured, it takes a further 125ms to launch the init process in the guest. Firecracker spawns a thread for each VM vCPU to use via the KVM API along with a separate management thread. The memory overhead of each thread (excluding guest memory) is less than 5MB.

Some comperations between Firecracker and QEMU [ref](https://news.ycombinator.com/item?id=25883837)
> By comparison: Firecracker is purpose-built in Rust for this one task, provides no BIOS, and offers only network, block, keyboard, and serial device support --- with tiny drivers (the serial support is less than 300 lines of code).

Firecracker integrate with existing container tooling, making adoption rather painless and easy to use. After some research in tooling to manage firecreaker VM, I choose to use [Ignite](https://github.com/weaveworks/ignite) that cli command is very similar to docker

> With Ignite, you pick an OCI-compliant image (Docker image) that you want to run as a VM, and then just execute `ignite run` instead of `docker run`

Install ignite and start a fresh VM is very simple, there’s basically 3 steps:

`Step 1`: Check your system is enable KVM and virtualization and install Ignite in here [Installing-guide](https://github.com/weaveworks/ignite/blob/main/docs/installation.md)

```bash
$ ignite version
Ignite version: version.Info{Major:"0", Minor:"8", GitVersion:"v0.10.0", GitCommit:"...", GitTreeState:"clean", BuildDate:"...", GoVersion:"...", Compiler:"gc", Platform:"linux/amd64"}
Firecracker version: v0.22.4
Runtime: containerd
```

`Step 2`: Create a VM sample config.yaml

```yaml
apiVersion: ignite.weave.works/v1alpha4
kind: VM
metadata:
  name: haiche-vm
spec:
  cpus: 2
  memory: 1GB
  diskSize: 6GB
  image:
    oci: weaveworks/ignite-ubuntu
  ssh: true
```

`Step 3`: Start your vm server under 5 second

```bash
$ sudo ignite run --config config.yaml

INFO[0001] Created VM with ID "3c5fa9a18682741f" and name "haiche-vm" 
```

Wolla :insert_icon_here: you've succeedfully created a new VM
To list the running `VMs`, enter:

```bash
$ ignite ps
VM ID                   IMAGE                           KERNEL                                  CREATED SIZE    CPUS    MEMORY          STATE   IPS             PORTS   NAME
3c5fa9a18682741f        weaveworks/ignite-ubuntu:latest weaveworks/ignite-kernel:5.10.51        63m ago 4.0 GB  2       1.0 GB          Running 172.17.0.3              haiche-vm
```

Once the VM is booted, it will have its network configured and will be accessible from the host (thanks to the IP assigned to the bridge) via password-less SSH and with sudo permissions

## SSH into the VM
### To SSH into a `VM`, enter:
```bash
$ ignite ssh haiche-vm
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 5.10.51 x86_64)
...
root@3c5fa9a18682741f:~#
```
To exit SSH, just quit the shell process with exit.


### To enter the vm server via ssh cli and rsa key
add your public key to `~/.ssh/authorized_keys` in new boosted VM or update config and create new VM with default path to pub key

```yaml
spec:
  ...
  ssh: path/your/id_rsa.pub
```

and then ssh to your vm

```bash
$ ssh -i path/your/id_rsa root@172.17.0.3
Welcome to Ubuntu 18.04.2 LTS (GNU/Linux 5.10.51 x86_64)
...
root@3c5fa9a18682741f:~#
```

## How I extend container to reduce repeatable setup process
After suscefully create VM, mostly I will install conda and some package to run my project. But I don't want to repeatly install conda and create new enviroment every time I create new VM. Here is my step to extend base Ubuntu image and use it to create better VM

Dockerfile

```Docker
FROM weaveworks/ignite-ubuntu

ENV PATH="/root/miniconda3/bin:${PATH}"
ARG PATH="/root/miniconda3/bin:${PATH}"

ARG DOCKER_IMAGE_APP

RUN apt-get update -qq && \
    apt-get update -y && \
    apt-get install git vim rsync \
        build-essential curl -y

RUN wget \
    https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh && \
    mkdir /root/.conda && \
    bash Miniconda3-latest-Linux-x86_64.sh -b && \
    rm -f Miniconda3-latest-Linux-x86_64.sh

RUN conda create -n haiche python=3.7

SHELL ["conda", "run", "-n", "haiche", "/bin/bash", "-c"]

RUN conda install fastapi

RUN which python && python -c "import fastapi"

RUN conda init bash && echo "source activate haiche" >> ~/.bashrc
```

minconda.yaml

```yaml
apiVersion: ignite.weave.works/v1alpha4
kind: VM
metadata:
  name: haiche-minconda-vm
spec:
  cpus: 2
  memory: 6GB
  diskSize: 30GB
  image:
    oci: haiche/ubuntu-minconda
  ssh: path/your/id_rsa.pub
```

Here is some tricky parts, the current ignite don't support local docker image build. I have to push the image to public register [Docker hub](https://hub.docker.com/) to succesfully pull to ignite image.
To start new VM

```bash
$ sudo ignite run --config minconda.yaml
...
INFO[0002] Created image with ID "cae0ac317cca74ba" and name "haiche/ubuntu-minconda" 
INFO[0004] Created VM with ID "c1ab652804e664ed" and name "haiche-minconda-vm" 
```


If you use private registery such as ECR, run the command above with `--runtime=docker` to pull private registery

Test our new vm with conda enviroment

```bash
$ ssh -i path/your/id_rsa root@172.17.0.4
...
(haiche) root@c1ab652804e664ed:~# python
Python 3.7.13 (default, Mar 29 2022, 02:18:16)
[GCC 7.5.0] :: Anaconda, Inc. on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> import fastapi
>>> fastapi.__version__
'0.74.1'
>>>
```

I really liked the configuration file approach for doing vmware because I found it easier to be able to see everything all in one place. Now the member can provide the config file and pubkey and I can create a fresh VM in instant

## Cloud supports nested virtualization

Another question I had in mind: “ok, where am I going to run these Firecracker VMs in production?“. The funny thing about running a VM in the cloud is that cloud instances are already VMs. Running a VM inside a VM is called “nested virtualization” and not all cloud providers support it – for example AWS only support nested virtualization in `bare-metal` instance which is ridiculously high prices.

GCP supports nested virtualization but not on default, you have to enable this feature in create vm section. DigitalOcean support nested virtualization on default even on their smallest droplets

Some open questions:

A few things still stuck on my mind with this approach:

* Currently firecracker doesn't support snapshot but will support in near future https://github.com/firecracker-microvm/firecracker/issues/1184

* I still don't understant clearly about why Firecracker is so fast