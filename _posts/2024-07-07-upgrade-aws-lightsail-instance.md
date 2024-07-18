---
layout: post
title: Upgrade AWS Lightsail instance plan for a running service
date: 2024-07-07 20:29
category: "Production"
tags: ["aws", "lightsail", "upgrade bundle", "upgrade plan", "discord", "https", "static ip", "migrate", "instance", "cloud"]
description: Steps to migrate a running service on an AWS Lightsail instance to an upgraded plan.
---

<!-- TOC start (generated with https://github.com/derlin/bitdowntoc) -->

- [Introduction](#introduction)
- [Steps](#steps)
   * [Take a snapshot, and create new instance](#take-a-snapshot-and-create-new-instance)
   * [Wait for AWS to handle the migration](#wait-for-aws-to-handle-the-migration)
   * [Add Firewall rule for HTTPS](#add-firewall-rule-for-https)
   * [Rebuild your server again](#rebuild-your-server-again)
   * [Detach static IP from older instance](#detach-static-ip-from-older-instance)
   * [Attach static IP to newer instance](#attach-static-ip-to-newer-instance)
   * [Delete the older instance](#delete-the-older-instance)
- [Conclusion](#conclusion)
- [References](#references)

<!-- TOC end -->

<!-- TOC --><a href="#" name="introduction"></a>
## Introduction

Recently I had to migrate my already running service in an AWS Lightsail instance to a larger one, for one of my side projects. I am just dumping my steps so that it might come in handy for future me or anyone who plans to do such an upgrade, because their current plan/bundle is not enough anymore.

<!-- TOC --><a href="#" name="steps"></a>
## Steps

<!-- TOC --><a href="#" name="take-a-snapshot-and-create-new-instance"></a>
### Take a snapshot, and create new instance
Go to the Lightsail console, and go to the `Snapshots` tab. For your instance which needs to be upgraded, click on the 3 dots on the instance summary, and click `Create new instance`.

This will open up a new page. Choose the new upgraded plan, eneter name for the new instance, and click on `Create` keeping all the defaults.

<!-- TOC --><a href="#" name="wait-for-aws-to-handle-the-migration"></a>
### Wait for AWS to handle the migration

This will take you to a new page to manage the new instance. AWS will take care of taking a complete snapshot of the instance, and then spawning a new upgraded instance, and then applying that snapshot to the newly launched instance

<!-- TOC --><a href="#" name="add-firewall-rule-for-https"></a>
### Add Firewall rule for HTTPS

If you have HTTPS enabled in your existing service, then from the instance management page, go to the `Networking` tab. Under the `IPv4 Firewall` section, add a new rule. You can add something like:

Application	| Protocol | 	Port or range / Code |	Restricted to
HTTPS	|   TCP	     |       443             |       Any IPv4 address

<!-- TOC --><a href="#" name="rebuild-your-server-again"></a>
### Rebuild your server again

With the new snapshot, you might want to ensure that things are running as before. So, re-run your build steps:

For instance, if you are running a self hosted discord, run something like:

```bash
./launcher rebuild app
./discourse-doctor
./discourse-setup
```

At this point, you will have 2 versions of your service running, however, the older one would still be serving traffic.

<!-- TOC --><a href="#" name="detach-static-ip-from-older-instance"></a>
### Detach static IP from older instance

If you have a static IP assigned to your older Lightsail instance, detach the static Ip from it. For this, go to your older Lightsail instance management page, and go to `Networking` tab. Under the `IPv4 networking` section, you will see a static IP assigned. Go ahead and click on `Detach` to de-link it from the older instance.

<!-- TOC --><a href="#" name="attach-static-ip-to-newer-instance"></a>
### Attach static IP to newer instance

Now, go to the `Networking` tab in the Lightsail dashboard (**NOT** your Lightsail instance management page). Your static IP would be listed there. Click on it to enter it's management page, and then under `Attach to an instance` section, attahc it to the new instance you just created.

Now all your traffic would be getting served from the new instance. Verify the same by making calls to your service, and using the `Metrics` tab from the new Lightsail instance management page.

<!-- TOC --><a href="#" name="delete-the-older-instance"></a>
### Delete the older instance

Go to the older Lightsail instance page, and then click on `Delete` and delete your older instance and everything attached to it.

>Congratulations, you have now successfully migrated your service to an upgraded Lightsail instance bundle.
{:.prompt-info}

<!-- TOC --><a href="#" name="conclusion"></a>
## Conclusion

The steps help migrate an existing service hosted on Lightsail instance to an upgraded one very smoothly. 

<!-- TOC --><a href="#" name="references"></a>
## References

[AWS Lightsail Upgradation Document](https://docs.aws.amazon.com/en_us/lightsail/latest/userguide/how-to-create-larger-instance-from-snapshot-using-console.html)
