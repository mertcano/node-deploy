# Deployment tools of BSC

## Installation
Before proceeding to the next steps, please ensure that the following packages and softwares are well installed in your local machine: 

> [!TIP]
> **Don't want to install these manually?** You can skip to the [Docker Version](#docker-version-recommended) section at the bottom of this file.

- nodejs: v16.15.0
- npm: 6.14.6
- go: 1.24+
- foundry
- python3 3.12.x
- poetry
- jq


## Quick Start
1. Clone this repository
```bash
git clone [https://github.com/bnb-chain/node-deploy.git](https://github.com/bnb-chain/node-deploy.git)
```

2. For the first time, please execute the following command
```bash
pip3 install -r requirements.txt
```

3. build `create-validator`

```bash
# This tool is used to register the validators into StakeHub.
cd create-validator
go build
```

4. Configure the cluster
```bash
cp .env.example .env
```

Then set deployment-specific values in the local, ignored `.env` file. You can also modify the following files:
Before starting the cluster, rotate any deployment credentials or key material that was previously committed to Git.

- `config.toml`
- `genesis/genesis-template.json`
- `genesis/scripts/init_holders.template`

5. Setup all nodes.
two different ways, choose as you like.
```bash
bash -x ./bsc_cluster.sh reset # will reset the cluster and start
# The 'vidx' parameter is optional. If provided, its value must be in the range [0, ${BSC_CLUSTER_SIZE}). If omitted, it affects all clusters.
bash -x ./bsc_cluster.sh stop [vidx] # Stops the cluster
bash -x ./bsc_cluster.sh start [vidx] # only start the cluster
bash -x ./bsc_cluster.sh restart [vidx] # start the cluster after stopping it
```

6. Setup a full node.
If you want to run a full node to test snap/full syncing, you can run:

> Attention: it relies on the validator cluster, so you should set up validators by `bsc_cluster.sh` firstly.

```bash
# reset a full sync node0
bash +x ./bsc_fullnode.sh reset 0 full
# reset a snap sync node1
bash +x ./bsc_fullnode.sh reset 1 snap
# restart the snap sync node1
bash +x ./bsc_fullnode.sh restart 1 snap
# stop the snap sync node1
bash +x ./bsc_fullnode.sh stop 1 snap
# clean the snap sync node1
bash +x ./bsc_fullnode.sh clean 1 snap
# reset a full sync node as fast node
bash +x ./bsc_fullnode.sh reset 2 full "--tries-verify-mode none"
# reset a snap sync node with prune ancient
bash +x ./bsc_fullnode.sh reset 3 snap "--pruneancient"
```

You can see the logs in `.local/fullnode`.

Generally, you need to wait for the validator to produce a certain amount of blocks before starting the full/snap syncing test, such as 1000 blocks.

## Background transactions
```bash
## normal tx
cd txbot
go build
./air-drops

## blob tx
cd txblob
go build
./txblob
```

## Docker Version (Recommended)

To run a fully containerized, isolated local BSC cluster without installing dependencies on your host machine, use the provided `Makefile` which handles the 3-phase orchestration automatically.

### Architecture Workflow

```mermaid
sequenceDiagram
    participant User
    participant Makefile
    participant Toolbox as Toolbox (Docker)
    participant HostFS as Host FileSystem
    participant Compose as Docker Compose
    participant Docker as Docker Engine

    User->>Makefile: make cluster-up

    Note over Makefile,Toolbox: Phase 1: Initialization (prepare)
    Makefile->>Toolbox: Start disposable 'bsc-toolbox' container and execute script

    Toolbox->>HostFS: Compile and save 'geth' binary
    Toolbox->>HostFS: Generate genesis, keystores, config.toml (.local/)
    Toolbox->>HostFS: Generate .env.cluster (cluster params, node count)
    Toolbox->>HostFS: Generate docker-compose.cluster.yml (based on env)

    Toolbox-->>Makefile: Exit (container removed)

    Note over Makefile,Compose: Phase 2: Start Cluster (up)
    Makefile->>Compose: docker compose -f docker-compose.cluster.yml up -d

    Compose->>HostFS: Read docker-compose.cluster.yml
    Compose->>HostFS: Load .env.cluster (env injection)

    Compose->>Docker: Create & start N bsc-node-X containers

    Docker->>HostFS: Mount volumes (./ -> /node_deploy)

    Docker->>Docker: Run node_entrypoint.sh inside each container
    Docker->>HostFS: Containers read config.toml / genesis / keystore

    Docker-->>User: Cluster running (N nodes)

    Note over Makefile,Toolbox: Phase 3: Validator Registration (register)
    Makefile->>Makefile: Wait for RPC (localhost:8545 ready)
    Makefile->>Toolbox: Start new Toolbox container inside 'bsc_cluster_network'
    Toolbox->>HostFS: Load .env (get RPC_URL)
    Toolbox->>Docker: Send 'Register' Transactions (via RPC to Node 0)
    Toolbox-->>Makefile: Exit (registration tasks submitted)

    Note over User,Docker: Local BSC Cluster active with registered validators
