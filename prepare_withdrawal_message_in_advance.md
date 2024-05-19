# Preparing Voluntary Exits in Advance 

This walkthrough explains the process of generating a voluntary exit message in advance, allowing it to be broadcasted to the network when needed. For additional information, refer to [the official GitHub page of ethdo](https://github.com/wealdtech/ethdo/blob/master/docs/exitingvalidators.md) and [this article](https://mirror.xyz/ladislaus.eth/wmoBbUBes2Wp1_6DvP6slPabkyujSU7MZOFOC3QpErs), both of which are useful resources.

Let's begin the tutorial!

The process involves creating two files: **offline-preparation.json** and **exit-operations.json**. To generate the former, an online computer with a synced beacon node is required, at least when using testnets. If there is no beacon node, the file will be generated for the mainnet.


## Installation

```bash
  # Get ethdo latest version 
  wget https://github.com/wealdtech/ethdo/releases/download/v1.35.2/ethdo-1.35.2-linux-amd64.tar.gz
  
  # Extract the file
  tar xvf ethdo-1.35.2-linux-amd64.tar.gz
  
  # Remove the file
  rm ethdo-1.35.2-linux-amd64.tar.gzls
  
  # Check if the installation has suceeded
  ./ethdo version
```

## Online process

This part requires an online computer (a computer with a WiFi connection and, in the case of testnets, an attached beacon node):

```bash
  ./ethdo validator exit --prepare-offline
```
This command will:
* Obtain information from your consensus node about all currently running validators and various additional details needed to generate the operations.
* Write this information into an offline-preparation.json file.

## Offline process

An offline computer is one that isn't connected to the internet and therefore won't be running a consensus node. It can be used optionally alongside an online computer to enhance the security of your mnemonic or private key. However, it's less convenient because it involves manual transfer of files between the online and offline computers.

Since the mnemonic will be utilized during the generation of exit operations, we'll employ the offline process. If you're using a private key or keystore, the online process should suffice in terms of security.

```bash
  ./ethdo validator exit --offline --mnemonic="<enter-your-mnemonic-:)>"
```

## Conclusions

If all the steps mentioned above were followed correctly, you should now have both files. To broadcast the withdrawal message, simply upload the files to a block explorer like Beaconcha.
* For holesky testnet: https://holesky.beaconcha.in/tools/broadcast
* For the mainnet: https://beaconcha.in/tools/broadcast








