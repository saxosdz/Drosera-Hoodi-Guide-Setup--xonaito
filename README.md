Drosera Trap + Operator Setup (Hoodi Testnet)

Single Trapper • Single Operator • Docker Compose Only

This guide walks you through deploying one Drosera Trap and running one Drosera Operator on the Hoodi Ethereum testnet, using Docker Compose only.

Prerequisites

Ubuntu / Linux (WSL Ubuntu works)

4 CPU cores, 8 GB RAM (recommended)

Docker + Docker Compose

One Ethereum wallet funded with Hoodi ETH

Open ports: 31313 (P2P) and 31314 (HTTP)

Install Dependencies

	sudo apt update && sudo apt upgrade -y
	sudo apt install -y curl git nano ufw

🔹Install Dependencies
 	
		sudo apt install curl ufw iptables build-essential git wget lz4 jq make gcc nano automake autoconf tmux htop nvme-cli libgbm1 pkg-config libssl-dev libleveldb-dev tar clang bsdmainutils ncdu unzip libleveldb-dev -y

Install Docker

	sudo apt update -y && sudo apt upgrade -y
	# Remove old docker packages if exist
	for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

	sudo apt-get update

	sudo apt-get install ca-certificates curl gnupg -y

	sudo install -m 0755 -d /etc/apt/keyrings

	curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

	sudo chmod a+r /etc/apt/keyrings/docker.gpg

	echo \
 	 "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  	$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
 	 sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

	sudo apt update -y && sudo apt upgrade -y

	sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

	# Test Docker
	sudo docker run hello-world


1️⃣ Trap Setup (Trapper)
Install Drosera + Foundry

	curl -L https://app.drosera.io/install | bash
	source ~/.bashrc
	droseraup

	curl -L https://foundry.paradigm.xyz | bash
	source ~/.bashrc
	foundryup

	# Bun (JavaScript runtime)
	curl -fsSL https://bun.sh/install | bash
	source ~/.bashrc

Initialize Trap Project

	mkdir ~/my-drosera-trap
	cd ~/my-drosera-trap
	
	git config --global user.email "your_github_email@example.com"
	git config --global user.name "your_github_username"

	forge init -t drosera-network/trap-foundry-template

Edit HelloWorldTrap.sol:

	#nano src/HelloWorldTrap.sol

Copy & delete everything & coller 

	// SPDX-License-Identifier: MIT
	pragma solidity ^0.8.20;

	contract DormantDrainTrap {
    event TrapTriggered(
        address indexed target,
        address indexed caller,
        uint256 amount,
        uint256 dormantPeriod
    );

    mapping(address => uint256) public lastActivity;
    uint256 public constant DORMANT_THRESHOLD = 30 days;

    function recordActivity(address target) external {
        lastActivity[target] = block.timestamp;
    }

    function checkDrain(
        address target,
        address caller,
        uint256 amount
    ) external {
        uint256 dormantTime = block.timestamp - lastActivity[target];

        if (dormantTime >= DORMANT_THRESHOLD && amount > 0) {
            emit TrapTriggered(
                target,
                caller,
                amount,
                dormantTime
            );
        }

        lastActivity[target] = block.timestamp;
    }
	}

Build Trap
	forge init -t drosera-network/trap-foundry-template
	forge build

Configure drosera.toml
nano drosera.toml

	ethereum_rpc = "https://ethereum-hoodi-rpc.publicnode.com"
	drosera_rpc  = "https://relay.hoodi.drosera.io"
	eth_chain_id = 560048
	drosera_address = "0x91cB447BaFc6e0EA0F4Fe056F5a9b1F14bb06e5D"

	[traps]

	[traps.helloworld]
	path = "out/HelloWorldTrap.sol/HelloWorldTrap.json"
	response_contract = "0x183D78491555cb69B68d2354F7373cc2632508C7"
	response_function = "helloworld(string)"
	cooldown_period_blocks = 33
	min_number_of_operators = 1
	max_number_of_operators = 1
	block_sample_size = 10
	private_trap = true
	whitelist = ["YOUR_OPERATOR_WALLET_ADDRESS"]
	
	# New Users/Migrate:
	# address = "DELETE THIS LINE WHEN APPLYING" (it will generate address after apply trap config)

	# Existing Users:
	# If you've deployed a trap with your wallet previously (Hoodi not Holesky), add your trap address here:
	# address = "TRAP_ADDRESS"

Deploy the Trap

	DROSERA_PRIVATE_KEY=YOUR_WALLET_PRIVATE_KEY drosera apply


After deployment, note the Trap Config Address.

2️⃣ Operator Setup (Docker Compose)
Create Operator Folder

	mkdir ~/Drosera-Network
	cd ~/Drosera-Network

Create .env

	nano .env

	ETH_PRIVATE_KEY=YOUR_OPERATOR_PRIVATE_KEY
	VPS_IP=YOUR_PUBLIC_IP

Create docker-compose.yaml

	services:
 	 drosera-operator:
    image: ghcr.io/drosera-network/drosera-operator:latest
    container_name: drosera-operator
    command: node
    ports:
      - "31313:31313"
      - "31314:31314"
    environment:
      - DRO__DB_FILE_PATH=/data/drosera.db
      - DRO__DROSERA_ADDRESS=0x91cB447BaFc6e0EA0F4Fe056F5a9b1F14bb06e5D
      - DRO__ETH__CHAIN_ID=560048
      - DRO__ETH__RPC_URL=https://ethereum-hoodi-rpc.publicnode.com
      - DRO__ETH__BACKUP_RPC_URL=https://rpc.hoodi.ethpandaops.io
      - DRO__ETH__PRIVATE_KEY=${ETH_PRIVATE_KEY}
      - DRO__NETWORK__EXTERNAL_P2P_ADDRESS=${VPS_IP}
      - DRO__NETWORK__P2P_PORT=31313
      - DRO__SERVER__PORT=31314
      - DRO__LISTEN_ADDRESS=0.0.0.0
      - DRO__DISABLE_DNR_CONFIRMATION=true
      - RUST_LOG=info
    volumes:
      - drosera_data:/data
    restart: always

	volumes:
	  drosera_data:

Start Operator

	docker compose pull ghcr.io/drosera-network/drosera-operator:latest
	docker compose up -d
	docker compose logs -f

3️⃣ Register Operator (Docker)

	docker run -it --rm ghcr.io/drosera-network/drosera-operator:latest register \
	  --eth-chain-id 560048 \
  	--eth-rpc-url https://ethereum-hoodi-rpc.publicnode.com \
 	 --drosera-address "trap address" \
	  --eth-private-key YOUR_OPERATOR_PRIVATE_KEY

4️⃣ Opt-In Trap (Recommended: Dashboard)

Go to
👉 https://app.drosera.io

Network: Hoodi
Opt-in your operator to your trap.

5️⃣ Firewall
sudo ufw allow ssh
sudo ufw allow 31313/tcp
sudo ufw allow 31314/tcp
sudo ufw enable

✅ Verification

Operator logs show peers + blocks

Trap visible in Drosera dashboard

Node status turns green

🎉 You’re live on Hoodi.

Useful Commands
# Logs
docker compose logs -f

# Restart operator
docker compose restart

# Stop operator
docker compose down

# Re-apply trap changes
DROSERA_PRIVATE_KEY=YOUR_KEY drosera apply

Support

Docs: https://dev.drosera.io

Discord: https://discord.gg/drosera
