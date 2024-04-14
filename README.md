
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
It will pop up a simple interface where u have to select where you live, if all has been done correctly, it should output your local time along with the universal time.

Let's create a jwtsecret file, this file contains an hexadecimal string which ensures correct authenticated communications between both clients.
```bash
   # Store the jwtsecret file at /secrets
   sudo mkdir -p /secrets
   
   # Create the jwtsecret file
   openssl rand -hex 32 | tr -d "\n" | sudo tee /secrets/jwtsecret
   
   # Enable read access
   sudo chmod 644 /secrets/jwtsecret
```
Enabling a firewall is not mandatory, but it's highly recommended for enhancing security. Similarly, installing fail2ban is an important step, as it automatically blocks IP addresses after a certain number of failed login attempts. If u want to proceed with this, refer to https://www.coincashew.com/coins/overview-eth/testnet-holesky-validator/step-2-configuring-node.


## Step 3: Installing Nethermind
After installing any client, I encourage you to do your own research. It's always a good practise to use minority client to avoid no-finality risks, in my case, I will use a Nethermind + Lighthosue combination.
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


## Step 4: Installing Lighthouse
```bash
   # Create a service user.
   sudo adduser --system --no-create-home --group consensus
   # Create data directory.
   sudo mkdir -p /var/lib/lighthouse
   # Assign ownership.
   sudo chown -R consensus:consensus /var/lib/lighthouse
   # Install dependencies.
   sudo apt update
   sudo apt install curl ccze jq -y
```

Download the binaries.
```bash
   RELEASE_URL="https://api.github.com/repos/sigp/lighthouse/releases/latest"
   # Latest version
   BINARIES_URL="$(curl -s $RELEASE_URL | jq -r ".assets[] | select(.name) | .browser_download_url" | grep x86_64-unknown-linux-gnu.tar.gz$)"
   
   echo Downloading URL: $BINARIES_URL
   
   cd $HOME
   # Download
   wget -O lighthouse.tar.gz $BINARIES_URL
   # Untar
   tar -xzvf lighthouse.tar.gz -C $HOME
   # Cleanup
   rm lighthouse.tar.gz

   # Install the binaries
   sudo mv $HOME/lighthouse /usr/local/bin/lighthouse
```

Create a systemd unit file for the consensus client.
```bash
   sudo nano /etc/systemd/system/consensus.service
```

And paste the following:
```bash
   [Unit]
   Description=Lighthouse Consensus Layer Client service for Holesky
   Wants=network-online.target
   After=network-online.target
   Documentation=https://www.coincashew.com
   
   [Service]
   Type=simple
   User=consensus
   Group=consensus
   Restart=on-failure
   RestartSec=3
   KillSignal=SIGINT
   TimeoutStopSec=900
   ExecStart=/usr/local/bin/lighthouse bn \
     --datadir /var/lib/lighthouse \
     --network holesky \
     --staking \
     --validator-monitor-auto \
     --metrics \
     --checkpoint-sync-url=https://holesky.beaconstate.ethstaker.cc \
     --port 9000 \
     --quic-port 9001 \
     --http-port 5052 \
     --target-peers 80 \
     --metrics-port 8008 \
     --execution-endpoint http://127.0.0.1:8551 \
     --execution-jwt /secrets/jwtsecret
   
   [Install]
   WantedBy=multi-user.target
```

Everytime you modify the systemd file, you should run the following command:
```bash
   sudo systemctl daemon-reload
```
Auto-start boot time, in case your consensus client stops.
```bash
   sudo systemctl enable consensus
```

Start your execution client.
```bash
   sudo systemctl start consensus
```

Consensus command cheatsheet:
```bash
   # View logs, my favourite one :)
   sudo journalctl -fu consensus | ccze
   # Start
   sudo systemctl start consensus
   # Stop
   sudo systemctl stop consensus
   # Restart
   sudo systemctl restart consensus
   # View status
   sudo systemctl status consensus
   # Reset Database
   sudo systemctl stop consensus
   sudo rm -rf /var/lib/lighthouse/beacon
   sudo systemctl restart consensus
```

Okay, so when you're setting things up, give it a few minutes for the client and the consensus clients to sync properly. While you're watching the logs, don't be surprised if you spot a few errors or warnings popping up.
1. If you see a SIGILL error, it's probably because your CPU isn't quite jiving with the latest Lighthouse version. No worries, just grab the Lighthouse portable build and install it.
2. Now, if you notice you've got a low peer count while checking the consensus logs (less than 10 peers connected, it's time to troubleshoot:
If you're running things on your own hardware, make sure your Wi-Fi provider's firewall isn't blocking with your ports. If everything looks clear there, move on to the next step.
For VPS setups, double-check that the provider's firewall isn't blocking any ports. Even if it's not, there might still be some filtering going on. In that case, change the peer ports in both the execution client and the consensus client.

To make those changes, do the following:
```bash
   # Open the execution file
   sudo nano /etc/systemd/system/execution.service
   # Change -Network.DiscoveryPort and --Network.P2PPort to another one, try adding the following port: 30305
   # Execute the following command
   sudo systemctl daemon-reload

   # Open the consensus file
   sudo nano /etc/systemd/system/consensus.service
   # Change --port and --quic-port, try adding the following: 9002 and 9003 respectivally.
   # Execute the following command
   sudo systemctl daemon-reload
```
Wait for a few minutes, and check again. Everyhting should be ok now :)

## Step 5: Before installing the validator

The first step is to acquire Holesky ethers. You'll need 32 ETH to create one validator. Simply put an address that YOU own into the following faucet:

https://holesky-faucet.pk910.de

After a few hours, you should have approximately 33 Holesky ethers in total (32 for the validator and the remainder for covering gas fees).

For added security, consider using a hardware wallet address to store your withdrawal address, and a browser dApp wallet like Metamask to hold the 32 ETH for the validator. However, for testnet purposes, I'll be using Metamask for both functions. Remember, this approach is for testing only and should not be used on the mainnet.

Now, let's proceed with creating your staking keys, which your validator will use to engage with the blockchain. I'll opt for the staking-deposit-cli method, as I believe it's the most reliable approach.

```bash
   # Download staking-deposit-cli
   #Install dependencies
   sudo apt install jq curl -y
   
   #Setup variables
   RELEASE_URL="https://api.github.com/repos/ethereum/staking-deposit-cli/releases/latest"
   BINARIES_URL="$(curl -s $RELEASE_URL | jq -r ".assets[] | select(.name) | .browser_download_url" | grep linux-amd64.tar.gz$)"
   BINARY_FILE="staking-deposit-cli.tar.gz"
   
   echo "Downloading URL: $BINARIES_URL"
   
   cd $HOME
   #Download binary
   wget -O $BINARY_FILE $BINARIES_URL
   #Extract archive
   tar -xzvf $BINARY_FILE -C $HOME
   #Rename
   mv staking_deposit-cli*amd64 staking-deposit-cli
   cd staking-deposit-cli
```

Make the mnemonic. Write this down on paper several times and keep it safe, without your mnemonic phrase, you won't be able to withdraw your ETH!:

```bash
   ./deposit new-mnemonic --chain holesky --execution_address <HARDWARE_WALLET_ADDRESS>
   # Replace <HARDWARE_WALLET_ADDRESS> for your own withdrawal address.
```
A simple interface will pop up:
1. Choose your language.
2. Repeat your withdrawal address.
3. Choose the language of the mnemonic word list (feel free to pick the one you're most comfortable with).
4. Choose how many new validators you wish to run (1 in my case).
5. Create a keystore password that secures your validator keystore files (rembember it).
6. Repeat your keystore password for confirmation.
7. Write down your 24 word mnemonic seed.
8. Type your mnemonic.

You should get:

"Success!
Your keys can be found at: /home/username/staking-deposit-cli/validator_keys"

It is advisable to verify the mnmeonic seed, so let's proceed with that:

```bash
   #Make temp directory to verify seeds
   mkdir -p ~/staking-deposit-cli/verify_seed
   #Re-generate keys
   ./deposit existing-mnemonic --chain holesky --folder verify_seed --execution_address <HARDWARE_WALLET_ADDRESS>
```
1. Choose your language.
2. Repeat your withdrawal address.
3. Type your mnemonic seed.
4. Since this is the first time generating keys, enter the index number as 0.
5. Repeat the index to confirm, 0.
6. Enter how many validators you with to run (same as before).
7. Enter any keystore password, since this is temporary and will be deleted.

Compare the deposit_data files:

```bash
   diff -s validator_keys/deposit_data*.json verify_seed/validator_keys/deposit_data*.json
```
You should get the following:

Files validator_keys/deposit_data-x.json and verify_seed/validator_keys/deposit_data-x.json are identical

Clean up:
```bash
   rm -r verify_seed
```

The files we have generated are:
1. **Keystore file:** Controls the validator's ability to sign transaction and can be recreated from your mnemonic recovery phrase. Keep this file secret.
2. **Deposit contract**: Public information about your validator and can also be recreated from your mnemonic recovery phrase.

After completing the previous steps, the next task is to deposit the 32 Holesky ETH in the launchpad (https://holesky.launchpad.ethereum.org/en/). However, before proceeding, we need to transfer our deposit_data-x.json file located in the validator_keys directory. Here are the options available:
1. If you're using your own hardware, no additional action is required.
2. If you're using a VPS:
   2.1. Windows connecting to VPS: Use WinSCP, known for its secure and user-friendly interface. Avoid using Filezilla due to concerns about potential malware.
   2.2. Linux connecting to VPS: Employ the SCP command for file transfer.

After completing the previous steps, proceed to the launchpad and connect your Metamask wallet. Ensure that the deposit contract address is 0x4242424242424242424242424242424242424242. During the transaction to the deposit contract, there should be NO warnings or errors displayed.

After completing the transaction, visit this page https://holesky.beaconcha.in and enter your public key to monitor the validator process. Please note that it may take approximately 10 days on the pending state for it to become active!

![image](https://github.com/Not-Elias/staking_walkthrough/assets/58786035/cf7db10c-8dc5-4c67-a4f6-7ae1f3040f18)


## Step 6: Installing the validator

```bash
    # Create a service user.
   sudo adduser --system --no-create-home --group validator
   # Create data directory.
   sudo mkdir -p /var/lib/lighthouse/validators
```

Import your validator keys using the keystore file. Make sure to enter the keystore password accurately. If you're setting up multiple validator nodes, avoid importing the validator keys into multiple clients simultaneously to prevent potential slashing. If you're transferring validators to a new setup or a different client, ensure the previous validator keys are deleted before proceeding.
```bash
   sudo lighthouse account validator import \
     --network holesky \
     --datadir /var/lib/lighthouse \
     --directory=$HOME/staking-deposit-cli/validator_keys \
     --reuse-password
```

If everything went good, your validator's public key will be shown. I encourage you to save it for future actions.
```bash
   sudo lighthouse account_manager validator list \
     --network holesky \
     --datadir /var/lib/lighthouse
```

Setup ownership permissions, including hardening the access to this directory.
```bash
   sudo chown -R validator:validator /var/lib/lighthouse/validators
   sudo chmod 700 /var/lib/lighthouse/validators
```

Let's create another systemd file, this time for the consensus client.
```bash
   sudo nano /etc/systemd/system/validator.service
```

And paste the following:
```bash
   [Unit]
   Description=Lighthouse Validator Client service for Holesky
   Wants=network-online.target
   After=network-online.target
   Documentation=https://www.coincashew.com
   
   [Service]
   Type=simple
   User=validator
   Group=validator
   Restart=on-failure
   RestartSec=3
   KillSignal=SIGINT
   TimeoutStopSec=900
   ExecStart=/usr/local/bin/lighthouse vc \
     --network holesky \
     --beacon-nodes http://localhost:5052 \
     --datadir /var/lib/lighthouse \
     --graffiti="<YOUR_GRAFFITI_GOES_HERE>" \
     --metrics \
     --metrics-port 8009 \
     --suggested-fee-recipient=<0x_CHANGE_THIS_TO_MY_ETH_FEE_RECIPIENT_ADDRESS>
   
   [Install]
   WantedBy=multi-user.target
```
Replace <0x_CHANGE_THIS_TO_MY_ETH_FEE_RECIPIENT_ADDRESS> with your own Ethereum address. Tips are sent to this address and are immediately spendable. You should also change the graffiti, be careful to not send any information that may indentify you.

Every time you change a systemd file, you have to run the next command:
```bash
   sudo systemctl daemon-reload
```

Enable auto-start at boot time, and start your client:
```bash
   sudo systemctl enable validator
   sudo systemctl start validator
```

Consensus command cheatsheet:
```bash
   # View logs, my favourite one :)
   sudo journalctl -fu validator | ccze
   # Start
   sudo systemctl start validator
   # Stop
   sudo systemctl stop validator
   # Restart
   sudo systemctl restart validator
   # View status
   sudo systemctl status validator
```

Wait for a few minutes and check the validator logs, if everything went ok you should get:

INFO Connected to beacon node(s)  


## Step 6: Installing Grafana and Prometheus
Prometheus is a monitoring platform that collects metrics from monitored targets and Grafana is a dashboard used to visualize the collected data.

Install Prometheus and Node Explorer:
```bash
   sudo apt-get install -y prometheus prometheus-node-exporter
```

Install Grafana:
```bash
   sudo apt-get install -y apt-transport-https
   sudo apt-get install -y software-properties-common wget
   sudo wget -q -O /usr/share/keyrings/grafana.key https://apt.grafana.com/gpg.key
   echo "deb [signed-by=/usr/share/keyrings/grafana.key] https://apt.grafana.com stable main" | sudo tee -a /etc/apt/sources.list.d/grafana.list
   sudo apt-get update && sudo apt-get install -y grafana
```

Enable services so they start automatically:
```bash
   sudo systemctl enable grafana-server prometheus prometheus-node-exporter
```

Create the prometheus.yml config file:
```bash
   # Remove the default prometheus.yml configuration file.
   sudo rm /etc/prometheus/prometheus.yml
   # Edit a new one
   sudo nano /etc/prometheus/prometheus.yml
```

For a Nethermind + Lighthouse configuration, you have to paste the following:
```bash
   global:
  scrape_interval:     15s # By default, scrape targets every 15 seconds.

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
    monitor: 'codelab-monitor'

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
   - job_name: 'node_exporter'
     static_configs:
       - targets: ['localhost:9100']
   - job_name: 'lighthouse'
     metrics_path: /metrics
     static_configs:
       - targets: ['localhost:8008']
   - job_name: 'lighthouse_validator'
     metrics_path: /metrics
     static_configs:
       - targets: ['localhost:8009']
   - job_name: 'nethermind'
     static_configs:
       - targets: ['localhost:6060']
```

Follow the next steps:
```bash
   # Update file permissions
   sudo chmod 644 /etc/prometheus/prometheus.yml

   # Restart the services
   sudo systemctl restart grafana-server prometheus prometheus-node-exporter

   # Verify that the services are running
   sudo systemctl status grafana-server prometheus prometheus-node-exporter
```

To access Grafana, you have to create a SSH Tunnel (only if you're using a VPS, if not just type localhost:3000 in yor browser).
1. If you are using Linux or MacOS:
```bash
   ssh -N -v <user>@<staking.node.ip.address> -L 3000:localhost:3000

   #Full Example
   ssh -N -v ethereum@192.168.1.69 -L 3000:localhost:3000
```
2. If you're using Windows:

   If you use Putty. Navigate to Connection > SSH > Tunnels > Enter Source Port 3000 > Enter Destination localhost:3000 > Click Add.

   ![image](https://github.com/Not-Elias/staking_walkthrough/assets/58786035/8d35afde-d6b8-40f2-a699-819f775b30e6)

   Save the configuration. Navigate to Session > Enter a session name > Save. Then open the connection.

   If you use Mobaxterm. Navigate to Tunneling > New SSH tunnel

   ![image](https://github.com/Not-Elias/staking_walkthrough/assets/58786035/33fb4117-225b-4985-955e-f00ab4cf2310)

   Introduce your own VPS IP.
   Save and start the tunnel.

Now you can access Grafana on your local machine by pointing a web browser to http://localhost:3000

Setup the Grafana Dashboards:
1. Open http://localhost:3000
2. Login with admin / admin
3. Change password
4. Click the configuration gear icon, then Add data Source
5. Select Prometheus
6. Set Name to "Prometheus"
7. Set URL to http://localhost:9090
8. Click Save & Test
9. Download and save your Lighthouse configuration file: https://raw.githubusercontent.com/Yoldark34/lighthouse-staking-dashboard/main/Yoldark_ETH_staking_dashboard.json

To download it windows, open PowerShell and do the following:
```PowerShell
   $url = "<lighhouse_config_file>"

   $outputFile = "C:\downloads\myfile.txt" # For example

   Invoke-WebRequest -Uri $url -OutFile $outputFile
```
10. Do the same with Nethermind: https://raw.githubusercontent.com/NethermindEth/metrics-infrastructure/master/grafana/provisioning/dashboards/nethermind.json
11. Download the node exporter dashboard for general system monitoring: https://grafana.com/api/dashboards/11074/revisions/9/download
12. Click Create + icon > Import
13. Add the consensus client dashboard via Upload JSON file
14. If needed, select Prometheus as Data Source.
15. Click the Import button.
16. Repeat steps 12-15 for the execution client dashboard.
17. Repeat steps 12-15 for the node-exporter dashboard.

These steps may slightly vary with new updates, please search for the latest Grafana configuration walkthroughs.















