
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

You can change your SSH configuration to automatically log in with this user by default. This way, you won't need to perform the previous step every time you log in. You should do the next steps in /home/ethereum directory.

Now, let's proceed with the installation of Chrony. Chrony is an implementation of the Network Time Protocol (NTP) designed to synchronize your computer's time with NTP servers. Given that the consensus client relies on precise timing to execute attestations and generate blocks effectively, it's important that your node's time remains accurate.
```bash
   # To install crhony
   sudo apt-get install chrony -y

   # To check the current status of chrony
   chronyc tracking
```

Set your timezone.
```bash
   sudo dpkg-reconfigure tzdata
```
It will pop up a simple interface where u have to select where you live, if all has been done correctly, it should output your local time whilelist the universal time.

Let's create a jwtsecret file, this file contains a hexadecimal string which ensures correct authenticated communications between both clients.
```bash
   # Store the jwtsecret file at /secrets
   sudo mkdir -p /secrets
   
   # Create the jwtsecret file
   openssl rand -hex 32 | tr -d "\n" | sudo tee /secrets/jwtsecret
   
   # Enable read access
   sudo chmod 644 /secrets/jwtsecret
```
Enabling a firewall is not obligatory, but it's highly recommended for enhancing security. Similarly, installing fail2ban is an important step, as it automatically blocks IP addresses after a certain number of failed login attempts. If u want to proceed with this, refer to https://www.coincashew.com/coins/overview-eth/testnet-holesky-validator/step-2-configuring-node.


## Step 3: Installing Nethermind
After installing any client, I encourage you to do your own research. It's always a good practise to use minority client to avoid no-finality risks, in my case, I will use Nethermind + Lighthosue.
![image](https://github.com/Not-Elias/staking_walkthrough/assets/58786035/7cd31dfe-4420-4337-a296-3528671337f4)
You can refer to the client diversity distribution chart at clientdiversity.org to assess the distribution of different clients.

```bash
   # Create a service user.
   sudo adduser --system --no-create-home --group execution
   # Create data directory.
   sudo mkdir -p /var/lib/nethermind
   # Assign ownership.
   sudo chown -R execution:execution /var/lib/nethermind
   # Install dependencies.
   sudo apt update
   sudo apt install ccze curl libsnappy-dev libc6-dev jq libc6 unzip -y
```

Download the binaries.

```bash
   RELEASE_URL="https://api.github.com/repos/NethermindEth/nethermind/releases/latest"
   # Latest version
   BINARIES_URL="$(curl -s $RELEASE_URL | jq -r ".assets[] | select(.name) | .browser_download_url" | grep linux-x64)"
   
   echo Downloading URL: $BINARIES_URL
   
   cd $HOME
   # Download
   wget -O nethermind.zip $BINARIES_URL
   # Unzip
   unzip -o nethermind.zip -d $HOME/nethermind
   # Cleanup
   rm nethermind.zip

   # Install the binaries
   sudo mv $HOME/lighthouse /usr/local/bin/lighthouse
```

Create a systemd unit file for the execution client.
```bash
   sudo nano /etc/systemd/system/execution.service
```

And paste the following information (the documentation part is not necessary, but i will not delete it to encourage you to visit their page):
```
   [Unit]
   Description=Nethermind Execution Layer Client service for Holesky
   Wants=network-online.target
   After=network-online.target
   Documentation=https://www.coincashew.com
   
   [Service]
   Type=simple
   User=execution
   Group=execution
   Restart=on-failure
   RestartSec=3
   KillSignal=SIGINT
   TimeoutStopSec=900
   WorkingDirectory=/var/lib/nethermind
   Environment="DOTNET_BUNDLE_EXTRACT_BASE_DIR=/var/lib/nethermind"
   ExecStart=/usr/local/bin/nethermind/nethermind \
     --config holesky \
     --datadir="/var/lib/nethermind" \
     --Network.DiscoveryPort 30303 \
     --Network.P2PPort 30303 \
     --Network.MaxActivePeers 50 \
     --JsonRpc.Port 8545 \
     --JsonRpc.EnginePort 8551 \
     --Metrics.Enabled true \
     --Metrics.ExposePort 6060 \
     --JsonRpc.JwtSecretFile /secrets/jwtsecret
     
   [Install]
   WantedBy=multi-user.target
```

Everytime you modify the systemd file, you should run the following command:
```bash
   sudo systemctl daemon-reload
```
Auto-start boot time, in case your execution client stops.
```bash
   sudo systemctl enable execution
```

Start your execution client.
```bash
   sudo systemctl start execution
```

Execution command cheatsheet:
```bash
   # View logs, my favourite one :)
   sudo journalctl -fu execution | ccze
   # Start
   sudo systemctl start execution
   # Stop
   sudo systemctl stop execution
   # Restart
   sudo systemctl restart execution
   # View status
   sudo systemctl status execution
   # Reset Database
   sudo systemctl stop execution
   sudo rm -rf /var/lib/nethermind/*
   sudo systemctl restart execution
```
Resetting the database should only be considered in specific scenarios such as recovering from a corrupted database due to power outages or hardware failures, reducing disk space usage by re-syncing, or upgrading to a new storage format. It's advisable to use this option as a last resort due to the significant time required for block synchronization. However, if any of the mentioned reasons apply to your situation, don't hesitate to proceed with the reset.

When reviewing the logs at this stage, it's common to encounter errors because the consensus client may not be fully synchronized yet.











