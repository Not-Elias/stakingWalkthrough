# staking_walkthrough
Find the process of setting up a validator node overwhelming? Don't worry, this tutorial will guide you step by step on how to do it!

This walkthrough will guide you through the essential steps to set up a validator node on the Holesky testnet, considering that Goerli is deprecated. Additionally, it will address common errors that may occur during the process. While heavily influenced by https://www.coincashew.com/coins/overview-eth/testnet-holesky-validator, this tutorial aims not to replace it but to share my personal experience, which may be beneficial for those with less expertise in this area. With that said, let's get started!

## Step 1: Hardware
The primary limiting factor for staking on the testnet, apart from technical expertise, lies in hardware requirements. Not only it's needed a decent computer, but also a continuous internet and power connection. In terms of hardware specifications, you need the following:

* 64-bit Linux (Ubuntu 22.04.1 is recommended).
* Dual-core CPU, preferably Intel Core i5–760 or AMD FX-8100 or higher.
* 16 GB RAM.
* 2 TB SSD.
Over time, the minimum hardware requirements are likely to escalate, as clients database increse.

Given these considerations, you essentially have two options to consider:

1. Purchase your own hardware (I recommend Intel NUC computers; available at https://www.geekom.es).
2. Rent a VPS (Virtual Private Server).
   
Personally, I opted for the latter as it tends to be more convenient. Specifically, I rented a VPS server from https://contabo.com/en/vps/, selecting a server with 12 vCPU cores, 48 GB RAM, 2 TB SSD, and 32 TB traffic. 

## Step 2: First steps
If you're using a VPS from a Windows computer, utilize tools like Putty or Mobaxterm (the one I'll be using) to establish a connection to the node. On the other hand, if you're operating from a Linux computer, initiate an SSH connection via:

```bash
ssh username@vps_ip_address
```

Simply log in if you're using a computer.

Once you are in, update and upgrade the node, and install all dependencies:

``` bash
   sudo apt-get update -y && sudo apt dist-upgrade -y

   sudo apt-get install git ufw curl ccze jq -y

   sudo apt-get autoremove

   sudo apt-get autoclean

   sudo reboot
```

Create a new user called ethereum. This step aligns with fundamental principles such as least privilege, security risk mitigation, and improved control. Do not do everyhting with root account, as this practice undermines security standards.

Add the new user and set the password.

```bash
   sudo useradd -m -s /bin/bash ethereum

   sudo passwd ethereum
```

Grant the new user administrative privileges by adding them to a superuser group.

```bash
   sudo usermod -aG sudo ethereum
```

Log in as this new user (switch to this new user whenever you need to interact with the validator node).

```bash
su ethereum
```

You can change your SSH configuration to automatically log in with this user by default. This way, you won't need to perform the previous step every time you log in.




















