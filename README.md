# 0l-Operations

Tooling for operations of https://0l.network

## Requirements

- [Docker 20+](https://docs.docker.com/get-docker/)
- [Task](https://taskfile.dev/#/installation)
- Internet connection
- Dedicated IP with opened ports (check each service)

## How to

- Clone this repo
- Create `.env` file (`cp example.env .env`) and set relevant variables. This depends on what you want to do:
  - Manage Docker images
  - Operate a fullnode or a validator setup
  - Run a miner only

### Manage docker images

> There exists a list of pre-built images that you may directly use here: https://hub.docker.com/r/nourspace/0l/tags
> These images were built using the [Dockerfile](./Dockerfile) in this repo and come without any guaranties.

- Set these env variables
  - `OL_ENV`: The name of the environment since these tools can manage multiple environments
  - `OL_REPO`: Repository to use. Defaults to `https://github.com/0LNetworkCommunity/libra-framework.git`. This is useful when building forks.
  - `OL_BRANCH`: Release tag or branch to check out and build
  - `OL_IMAGE`: Docker image (`username/image:tag`), used to build/push images, also to run services

- Build and push docker images
  ```shell
  task docker:build
  task docker:push
  ```

- [Optional] Build and push source docker images

  > `source` image is a stage in Dockerfile that has 0L source and Rust toolchain

  ```shell
  task docker:build:source
  task docker:push:source
  ```

- [Optional] Build and push builder docker images

  > `builder` image is a stage in Dockerfile that has 0L source, built binaries, and Rust toolchain

  ```shell
  task docker:build:builder
  task docker:push:builder
  ```

### Operate 0L node (Fullnode or Validator/VFN)

- Set these env variables
  > Create these directories if they not yet exist
  - `OL_DATA_DIR`: points to host path where 0L data directory is located
  - `OL_SOURCE_DIR`: points to host path where 0L repo is checked out
  - `OL_RECOVERY_DIR`: points to host path where 0L recovery is located

  - `OL_NODE_MODE`: determines node mode `fullnode`, `validator`, `vfn`

  > Warning! Only tested with `validator` in v7

- Use docker compose to manage services: `node`, `tower`, `monitor`

  > Warning! Only tested with `node` in v7

  ```shell
  # Start service(s) in background
  docker compose up -d node tower monitor 

  # Tail logs
  docker compose logs -f --tail 50
  ```

### docker compose commands

This is a list of some useful commands.

```shell
########## Main Services #########

# Start service(s)
docker compose up tower 

# You can pass one or more services
docker compose up node tower monitor

# Start service(s) in background
docker compose up -d node tower monitor 

# Display all logs
docker compose logs tower

# Tail logs
docker compose logs -f --tail 50 node

# Force recreate and start service(s)
docker compose up -d --force-recreate node 

# Stop service(s)
docker compose stop node

# Stops containers and removes containers, networks, volumes, and images
# Data directory on the host specified with `$OL_DATA_DIR` won't be deleted. Only the Docker volume will be unmounted
docker compose down

########## Utility Services #########

# Start a shell service with all 0L binaries available
# Useful to run `txs` commands
docker compose run --rm shell bash

# Start a builder service with all 0L source and binaries available
# Useful to run `make` commands
docker compose run --rm builder bash

# Start a builder service and exec into it to keep the container and its files after exiting
docker compose up -d builder
docker compose exec builder bash

# Use Task to start utility services
task shell
task source
task builder

# Clean all utility containers with their data
task clean-utils
```

---

## Main Services

This are the main services commonly run by node operators.

### `node` service

> Ports - operations: 6180 (public consensus), 6181 (private VFN), 6182 (public Fullnode), 8080 (public rpc)

> Ports - monitoring: 9101 (0l node)

This service can run as either of: `fullnode`, `validator`, `vfn`. That is determined by the `OL_NODE_MODE` env
variable.

### `tower` service (not tested in v7)

Tower service can be launched in multiple modes. That depends on whether the accompanying node is fully synced or not.
Validators have operator accounts that can be used to start tower without providing mnemonic phrase for owner account.

- mode 1: initial start uses upstream as fullnode/validator are not yet synced
  > `entrypoint: [ "tower", "--use-upstream-url", "--verbose", "start" ]`

- mode 2: fullnode/validator are synced: run as non-operator (requires MNEM)
  > `entrypoint: [ "tower", "--verbose", "start" ]`
  
  This mode is also meant for miners only as they will be using an upstream fullnode that is synced

- mode 3: fullnode/validator are synced: run as operator
  > `entrypoint: [ "tower", "--is-operator", "--verbose", "start" ]`


### `monitor` service  (not tested in v7)

> Port: 3030

Start web monitor.

---

## Utility Services

These services are mostly for debugging and developing inside containers.

### `shell` service

Run different ol and other commands inside an isolated container. This uses the production image

### `source` service

Run make commands inside an isolated container with access to 0L source under `/root/libra` and Rust toolchain.

### `builder` service

Run different ol and other make commands inside an isolated container with access to 0L source under `/root/libra` and Rust toolchain.

---

## Restarting Nodes with Cron

In some cases when the network is having syncing issues we resort to restarting nodes on the top of the hour. Since
not every node operator might be available, setting a cronjob to restart nodes seemed like a good work around.

Add the content of [restart.cron](./restart.cron) to `crontab -e`. Make sure to update paths as necessary. It is meant to stop node at :58 and start it again on :00.

The cron output will be logged to `./cron-runs.log

---

## Notes

### General

- Some services like `monitor` expect other services like `node` and `tower` to be running first.


### Miners

- Make sure to set `default_node` and `upstream_nodes` in `~/.0L/0L.toml` with IPs of fullnodes.
  https://github.com/0LNetworkCommunity/seed-peers/blob/main/fullnode_seed_playlist.json

- `TEST` variable should not be included in any services other than `tower`
  Pending PR https://github.com/0LNetworkCommunity/libra/pull/979

---

### ~~Run a Tower (0L Miner) only~~ OUTDATED V5.2

> This is useful if you have a unix server, and you only want to mine 0L coins and identity without operating any nodes.

- Set these env variables
  - `OL_DATA_DIR`: points to host path where 0L data directory is located
  > Create this directory if it does not yet exist
  - `OL_TOWER_OPERATOR=''`: this is only for validators that own an operator account
  - `OL_TOWER_USE_FIRST_UPSTREAM='--use-first-url'`: this makes sure the tower is using the upstream urls
  - `OL_TOWER_TEST=y`: allows reading mnemonic seed phrase from env
  > There is a [PR](https://github.com/0LNetworkCommunity/libra/pull/979) that allows reading `MNEM` without setting `TEST=y`
  - `MNEM`: the mnemonic seed phrase generated during account creation

- [Check if an account needs to be created](https://github.com/0LNetworkCommunity/libra/blob/main/ol/documentation/node-ops/tower_mining_VDF_proofs.md)
  This step makes sure we have an account and the data directory `OL_DATA_DIR` contains the required files:
  - `./vdf_proofs/proof_0.json` (and potentially other proofs)
  - `./key_store.json`
  - `./0L.toml`

- To create a new account
  ```shell
  # Generate mnemonic seed phrase
  onboard keygen
  
  # Let someone onboard it using authKey
  
  # Create 0L.toml file
  cd $HOME/.0L
  
  # Initialise configs (requires ol version >= 5.0.12)
  ol init --app --rpc-playlist https://raw.githubusercontent.com/0LNetworkCommunity/seed-peers/main/fullnode_seed_playlist.json

  # Create the first proof and `account.json`, the following will take about 30-40 minutes
  onboard user
  ```

- Use docker compose to manage `tower` service

  ```shell
  # Start Tower service in background
  docker compose up -d tower

  # Tail logs
  docker compose logs -f --tail 50 tower
  ```

---

## Todos

- [ ] More docs updates for v7
- [ ] Bring back VFN, Fullnode, Tower, and other services

### Docker

- [ ] Use smaller image
- [ ] Security
  - [ ] use non-root user
  - [ ] what else?

### Deployment

- [ ] Create CI to auto build image
