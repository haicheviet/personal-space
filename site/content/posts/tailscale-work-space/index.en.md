---
title: "Tailscale for workspace"
date: 2022-06-23T21:57:40+08:00
lastmod: 2022-06-23T16:45:40+08:00
description: "This article shows how to use Firecracker to create cluster VMs with container."
resources:
- name: "featured-image"
  src: "featured-image.webp"

tags: ["Firecracker", "Container", "QEMU", "Ignite"]
categories: ["Container"]


---

Recently I was asked to join the recruitment process and onboarding of new members. As the team and our project grow both in size and complexity, our document to hand on projects is complicated and only work in certain operating system.
So to make sure the onboarding process is fast and reproducible, I have to come up with a new plan to create an isolate enviroment for coding and less learning curve as possible.

<!--more-->

## My goal

> An isolate enviroment where every member get their own resource and custom preinstall package

Our project in [Jobhopin](https://jobhopin.com/) combines multiple languages (Rust, Python, ...) and the process of creating virtualenv is quite tedious with multiple attempts to make it work. Usually, It took 2-3 weeks for newcomers to learn about our current projects and work effectively

## Tailscale as localhost?

## Tailscale as bastion host?