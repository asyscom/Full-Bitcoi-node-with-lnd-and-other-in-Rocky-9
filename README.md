
---

# Detailed Guide for Installing a Full Bitcoin Node with Lightning Network, Electrs, Thunderhub, Tor, and LNbits on Rocky Linux 9

### Minimum System Requirements

- **CPU**: 2 vCPUs (4 recommended)
- **RAM**: 8 GB (16 GB recommended)
- **Storage**: **Minimum 1 TB SSD** (recommended 2 TB)
- **Network Connection**: Broadband connection, preferably with a static IP
- **Operating System**: Rocky Linux 9 x64

---

### Step 1: Create Admin User (`admin`)

1. **Create the `admin` user**:

```bash
sudo useradd -m admin
sudo passwd admin
```

2. **Add `admin` to the sudo group**:

```bash
sudo usermod -aG wheel admin
```

---

### Step 2: Update the System

Log in as `admin` and update the system:

```bash
sudo dnf update -y
```

---

### Step 3: Install General Dependencies

Install the necessary dependencies for all the software we will install:

```bash
sudo dnf install -y git wget curl tar build-essential autoconf libtool pkg-config libssl-dev libboost-all-dev libsqlite3-dev python3 python3-pip jq
```

---

### Step 4: Create the `/data` Directory

Create the `/data` directory to host the data for all applications:

```bash
sudo mkdir /data
sudo chown admin:admin /data
```

---

### Step 5: Create Application Users and Symbolic Links

Create users for each application and add a symbolic link (symlink) in their home directory pointing to the respective directory in `/data`.

1. **Create a user for Bitcoin Core**:

```bash
sudo useradd -m -d /home/bitcoin -s /bin/bash bitcoin
sudo mkdir -p /data/bitcoin
sudo ln -s /data/bitcoin /home/bitcoin/bitcoin_data
```

2. **Create a user for LND**:

```bash
sudo useradd -m -d /home/lnd -s /bin/bash lnd
sudo mkdir -p /data/lnd
sudo ln -s /data/lnd /home/lnd/lnd_data
```

3. **Create a user for Electrs**:

```bash
sudo useradd -m -d /home/electrs -s /bin/bash electrs
sudo mkdir -p /data/electrs
sudo ln -s /data/electrs /home/electrs/electrs_data
```

4. **Create a user for Thunderhub**:

```bash
sudo useradd -m -d /home/thunderhub -s /bin/bash thunderhub
sudo mkdir -p /data/thunderhub
sudo ln -s /data/thunderhub /home/thunderhub/thunderhub_data
```

5. **Create a user for LNbits**:

```bash
sudo useradd -m -d /home/lnbits -s /bin/bash lnbits
sudo mkdir -p /data/lnbits
sudo ln -s /data/lnbits /home/lnbits/lnbits_data
```

---

### Step 6: Access Permissions for `admin`

Grant the `admin` user access permissions to the files of each application:

```bash
sudo usermod -aG bitcoin,lnd,electrs,thunderhub,lnbits admin
```

---

### Step 7: Install Bitcoin Core

#### 1. Download and Install Bitcoin Core

Log in as the `bitcoin` user:

```bash
su - bitcoin
cd /tmp
VERSION=27.1
```

Download and verify the binaries:

```bash
wget https://bitcoincore.org/bin/bitcoin-core-$VERSION/bitcoin-$VERSION-x86_64-linux-gnu.tar.gz
wget https://bitcoincore.org/bin/bitcoin-core-$VERSION/SHA256SUMS
wget https://bitcoincore.org/bin/bitcoin-core-$VERSION/SHA256SUMS.asc
sha256sum --check SHA256SUMS --ignore-missing
```

Extract and install the binaries:

```bash
tar -xvf bitcoin-$VERSION-x86_64-linux-gnu.tar.gz
sudo install -m 0755 -o root -g root -t /usr/local/bin bitcoin-$VERSION/bin/bitcoind bitcoin-$VERSION/bin/bitcoin-cli
```

Verify the installation:

```bash
bitcoind --version
```

#### 2. Configure Bitcoin Core

Edit the `bitcoin.conf` configuration file:

```bash
nano /home/bitcoin/.bitcoin/bitcoin.conf
```

Add the following configurations:

```ini
server=1
daemon=1
txindex=1
prune=0
listen=1
proxy=127.0.0.1:9050  # For Tor
bind=127.0.0.1
datadir=/data/bitcoin
```

Start Bitcoin Core:

```bash
bitcoind -daemon
```

Monitor synchronization status:

```bash
bitcoin-cli getblockchaininfo
```

---

### Step 8: Configure Bitcoin Core as a `systemd` Service

1. Create the service file:

```bash
sudo nano /etc/systemd/system/bitcoind.service
```

2. Insert the following content:

```ini
[Unit]
Description=Bitcoin Daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/bitcoind -conf=/home/bitcoin/.bitcoin/bitcoin.conf -datadir=/data/bitcoin
User=bitcoin
Group=bitcoin
Type=forking
PIDFile=/data/bitcoin/bitcoind.pid
Restart=on-failure
PrivateTmp=true

[Install]
WantedBy=multi-user.target
```

3. Enable and start the service:

```bash
sudo systemctl enable bitcoind
sudo systemctl start bitcoind
```

4. Verify the service status:

```bash
sudo systemctl status bitcoind
```

---

### Step 9: Install Lightning Network (LND)

#### 1. Download and Install LND

Log in as the `lnd` user:

```bash
su - lnd
cd /tmp
LND_VERSION=0.18.3-beta
```

Download the LND binaries:

```bash
wget https://github.com/lightningnetwork/lnd/releases/download/v$LND_VERSION/lnd-linux-amd64-v$LND_VERSION.tar.gz
wget https://github.com/lightningnetwork/lnd/releases/download/v$LND_VERSION/manifest-v$LND_VERSION.txt
wget https://github.com/lightningnetwork/lnd/releases/download/v$LND_VERSION/manifest-v$LND_VERSION.txt.sig
sha256sum --check manifest-v$LND_VERSION.txt
```

Extract and install the binaries:

```bash
tar -xvf lnd-linux-amd64-v$LND_VERSION.tar.gz
sudo install -m 0755 -o root -g root -t /usr/local/bin lnd-linux-amd64/lnd lnd-linux-amd64/lncli
```

Verify the installation:

```bash
lnd --version
```

#### 2. Configure LND

Edit the configuration file:

```bash
nano /home/lnd/.lnd/lnd.conf
```

Insert the following configuration:

```ini
[Application Options]
alias=your_lightning_node
color=#FF0000
listen=127.0.0.1:9735
rpclisten=127.0.0.1:10009
tlsextraip=127.0.0.1
tlsautorefresh=1
tlsdisableautofill=1

[Bitcoin]
bitcoin.active=1
bitcoin.mainnet=1
bitcoin.node=bitcoind

[Bitcoind]
bitcoind.rpchost=127.0.0.1
bitcoind.zmqpubrawblock=tcp://127.0.0.1:28332
bitcoind.zmqpubrawtx=tcp://127.0.0.1:28333
```

Start LND:

```bash
lnd
```

---

### Step 10: Configure LND as a `systemd` Service

1. Create the service file:

```bash
sudo nano /etc/systemd/system/lnd.service
```

2. Insert the following content:

```ini
[Unit]
Description=LND Lightning Daemon
After=network.target

[Service]
ExecStart=/usr/local/bin/lnd --configfile=/home/lnd/.lnd/lnd.conf
User=lnd
Group=lnd
Type=simple
Restart=always

LimitNOFILE=128000

[Install]
WantedBy=multi-user.target
```

3. Enable and start the service:

```bash
sudo systemctl enable lnd
sudo systemctl start lnd
```

4. Verify the service status:

```bash
sudo systemctl status lnd
```

---

### Step 11: Install Electrs

#### 1. Download and Install Electrs

Log in as the `electrs` user:

```bash
su - electrs
cd /data
git clone https://github.com/romanz/electrs.git
cd electrs
cargo build --release
```

Verify the installation:

```bash
./target/release/electrs --version
```

#### 2. Configure Electrs

Create the configuration file:

```bash
nano ~/.electrs/config.toml
```

Insert the following configuration:

```ini
network = "mainnet"
db_dir = "/data/elect



rs/db"
rpc_addr = "127.0.0.1:8080"
daemon_dir = "/data/bitcoin/.bitcoin"
```

Start Electrs:

```bash
./target/release/electrs
```

---

### Step 12: Configure Electrs as a `systemd` Service

1. Create the service file:

```bash
sudo nano /etc/systemd/system/electrs.service
```

2. Insert the following content:

```ini
[Unit]
Description=Electrs Server
After=network.target

[Service]
ExecStart=/data/electrs/target/release/electrs
User=electrs
Group=electrs
Type=simple
Restart=always

[Install]
WantedBy=multi-user.target
```

3. Enable and start the service:

```bash
sudo systemctl enable electrs
sudo systemctl start electrs
```

4. Verify the service status:

```bash
sudo systemctl status electrs
```

---

### Step 13: Install Thunderhub

Log in as the `thunderhub` user:

```bash
su - thunderhub
cd /data/thunderhub
```

Install Node.js and npm if not already installed:

```bash
sudo dnf install -y nodejs npm
```

Clone the Thunderhub repository and install dependencies:

```bash
git clone https://github.com/boltzmann/ThunderHub.git .
npm install
```

Start Thunderhub:

```bash
npm start
```

---

### Step 14: Install LNbits

Log in as the `lnbits` user:

```bash
su - lnbits
cd /data/lnbits
```

Clone the LNbits repository and install dependencies:

```bash
git clone https://github.com/lnbits/lnbits.git .
pip install -r requirements.txt
```

Start LNbits:

```bash
python3 -m lnbits
```

---

### Step 15: Install and Configure Tor

1. Install Tor:

```bash
sudo dnf install -y tor
```

2. Configure Tor to run as a service:

```bash
sudo nano /etc/tor/torrc
```

Insert the following lines to configure your onion service:

```ini
HiddenServiceDir /var/lib/tor/hidden_service/
HiddenServicePort 80 127.0.0.1:8080  # Modify the port according to your setup
```

3. Restart Tor:

```bash
sudo systemctl restart tor
```

4. Get your onion address:

```bash
sudo cat /var/lib/tor/hidden_service/hostname
```

---

### Step 16: Final Steps

- Make sure all services are running smoothly and monitor their logs.
- Secure your services and configure firewalls as necessary.

---

This guide is intended to give you a comprehensive understanding of how to set up a full Bitcoin node with Lightning Network on Rocky Linux 9. Always refer to the official documentation of each software for the latest updates and security best practices.

--- 

If you found this guide helpful and would like to support the project, consider making a donation. Your contributions help maintain and improve this resource.

### Donate via Bitcoin
You can send Bitcoin directly to the following address:

**`bc1qy0l39zl7spspzhsuv96c8axnvksypfh8ehvx3e`**

### Donate via Lightning Network
For faster and lower-fee donations, you can use the Lightning Network:

**asyscom@sats.mobi**

Thank you for your support!

## License
This project is licensed under the MIT License.
