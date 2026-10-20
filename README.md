# GB Parano1d Solo (`gb-parano1d-solo`)

Solo mining for Parano1d (NOID) on your own node. One Docker container runs
a Parano1d node and a Stratum pool for all your rigs; every block you find
is paid straight to your own address.

- **Your own node, your own blocks.** No pool account, no payout threshold,
  no waiting for payouts. Your wallet keys never go into the container.
- **Back to work right after every block.** The node builds the next template
  much faster than a standard node: your rigs get new work within about 1 s
  of the fastest pool in the network (measured on mainnet over 90 minutes and
  155 blocks on an 8-core desktop). A standard node leaves rigs idle for about
  20 s after every block.
- **All your rigs in one place.** Any number of rigs on Stratum V1 — Linux,
  Windows, HiveOS. Works with gb-parano1d-miner out of the box.
- **See what you get.** Dashboard with hashrate per rig, effort, expected time
  per block and the state of every block you found.

## What you need

- A PC with Docker: Linux (Docker Engine with Compose v2) or Windows 10/11
  (Docker Desktop).
- A fast CPU: the node proves every block template. 8 or more modern cores
  are recommended; AVX-512 helps.
- A few GB of free SSD space.
- Your Parano1d address (`o1…`).
- Recommended: forward TCP port 9600 to this PC in your router, so other
  nodes can connect to yours and new blocks reach you sooner.

## Install on Linux

```sh
mkdir gb-parano1d-solo
cd gb-parano1d-solo
curl -fsSLO https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/docker-compose.yml
curl -fsSL https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/.env.example -o .env
nano .env
```

In `.env`, set `WALLET_ADDRESS=` to your `o1…` address and save. Then:

```sh
docker compose up -d
docker compose logs -f
```

When the log says `node is synchronized and ready for mining` (usually within
a minute or two), connect your rigs.

## Install on Windows

1. Install [Docker Desktop](https://www.docker.com/products/docker-desktop/)
   and start it (it uses WSL 2; the installer sets that up).
2. Open PowerShell and run:

```powershell
mkdir C:\gb-parano1d-solo
cd C:\gb-parano1d-solo
curl.exe -fsSLO https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/docker-compose.yml
curl.exe -fsSL https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/.env.example -o .env
notepad .env
```

3. In Notepad, set `WALLET_ADDRESS=` to your `o1…` address, save and close.
   Then, in the same PowerShell window:

```powershell
docker compose up -d
docker compose logs -f
```

4. If Windows asks whether Docker may accept network connections, allow it —
   otherwise rigs on other machines cannot reach port 3333.

When the log says `node is synchronized and ready for mining`, connect your
rigs. Any folder works instead of `C:\gb-parano1d-solo`.

## Connect your rigs

Point every rig at `stratum+tcp://<ip-of-the-pool-pc>:3333`. The worker name
is just a label; payouts always go to `WALLET_ADDRESS`.

- **gb-parano1d-miner:** enter the pool URL and your address in `start.sh`
  (Linux) or `start.bat` (Windows), or on the command line:

  ```sh
  ./gb-parano1d-miner cuda-mine --rpc stratum+tcp://192.168.1.10:3333 --coinbase o1… --worker rig1 --search-binary ./gb-parano1d-engine
  ```

- **HiveOS:** flight sheet with pool URL `stratum+tcp://192.168.1.10:3333`
  and your `o1…` address as the wallet.
- **Other miners:** any Stratum V1 miner that speaks the parano1d dialect.

Replace `192.168.1.10` with the LAN address of the pool PC.

## Check what is happening

Dashboard: `http://localhost:9640` on the pool PC. To open it to your LAN, set
`STATS_BIND_HOST=0.0.0.0` in `.env` and restart (`docker compose up -d`); then
any device can open `http://<ip-of-the-pool-pc>:9640`.

The tiles show hashrate (10 min and 1 min), connected rigs, effort (work
done since the last block, as a share of the expected work per block), blocks
found, chain height, how long the node takes to prove a block, and how long
your rigs waited for new work after the last network block ("Last template")
and on average ("Template delivery"). Below: every rig with its hashrate and
shares, and every block you found with its state (submitted, accepted, final
or orphaned).

The same from the command line:

```sh
docker exec gb-parano1d-solo gbsolo status    # node, rigs, hashrate, effort, blocks
docker exec gb-parano1d-solo gbsolo workers   # per rig
docker exec gb-parano1d-solo gbsolo blocks    # blocks found
```

## Solo mining means variance

A block is found when one of your rigs hits the network target. With a small
farm that can take days or weeks; the dashboard and `gbsolo status` show the
expected time per block for your hashrate. Nothing is lost while you wait —
every block you find pays the full reward to your address.

## Update

```sh
docker compose pull
docker compose up -d
```

Your data (node database, pool state, found blocks) stays in the
`gbsolo-data` volume. `docker compose down` stops the container and keeps the
data; `docker compose down -v` also deletes the data.

## Without Docker Hub

Every release page also carries the image as a file
(`gb-parano1d-solo-<version>-image.tar.gz` with a `.sha256` checksum). To use
it instead of pulling from Docker Hub:

```sh
docker load -i gb-parano1d-solo-0.1.2-image.tar.gz
```

Then set `GBSOLO_VERSION=0.1.2` in `.env` and run `docker compose up -d`.

## Settings (.env)

| Variable | Default | Meaning |
|---|---|---|
| `WALLET_ADDRESS` | – | Your `o1…` address (required) |
| `STRATUM_PORT` | `3333` | Port your rigs connect to |
| `P2P_PORT` | `9600` | Node P2P port; forward it in your router |
| `PUBLIC_ADDRESS` | – | Your public IP, if `P2P_PORT` is forwarded |
| `P2P_PEERS` | `16` | Outbound peers of the node (8–16) |
| `STATS_BIND_HOST` | `127.0.0.1` | `0.0.0.0` opens the dashboard to your LAN |
| `STATS_PORT_HOST` | `9640` | Port of the dashboard |
| `NODE_THREADS` | all | CPU threads for block proofs |
| `STATUS_INTERVAL` | `60` | Seconds between status lines in the log |
| `GBSOLO_VERSION` | `latest` | Image version to run (e.g. `0.1.2`) |
| `TLS_ENABLED`, `TLS_CERT`, `TLS_KEY` | off | Stratum over TLS for rigs on the internet (needs a real certificate) |

## License

The pool program is proprietary (© gustlborg). The image contains a modified
build of the Parano1d node, licensed under the Apache License 2.0; see
`/usr/share/doc/gb-parano1d-solo/` in the image.
