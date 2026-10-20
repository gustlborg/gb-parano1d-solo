# GB Parano1d Solo (`gb-parano1d-solo`)

Solo mining for ParanO(1)d (NOID) on your own node: one Docker container runs a
ParanO(1)d node and a Stratum pool for all your rigs. Your blocks are paid
straight to your own address.

- **Your own node, your own blocks.** No pool account, no payout threshold, no
  waiting for pool payouts. Your blocks pay your wallet; you never put your
  wallet keys into the container.
- **Back to work right after every block.** The node builds the next
  template much faster than a standard node: your rigs get new work within
  about 1 s of the fastest pool in the network — measured on mainnet over
  90 minutes and 155 blocks on an 8-core desktop. A standard node leaves rigs
  idle for about 20 s after every block; over 20 minutes rigs had work 92 % of
  the time, versus 44 % with a standard node.
- **All your rigs, one place.** Any number of rigs on Stratum V1 (Linux,
  Windows, HiveOS). Works with gb-parano1d-miner out of the box and with other
  Stratum V1 miners that speak the parano1d dialect.
- **See what you get.** Hashrate per rig, effort, expected time per block and
  the state of every block that pays you (accepted, final, orphaned).

## Requirements

- Linux with Docker (Compose v2).
- A fast CPU: the node proves every block template. More cores and AVX-512
  shorten the proof; 8 or more modern cores are recommended.
- 8 GB RAM (the node uses about 2 GB), a few GB of SSD space.
- Recommended: TCP port 9600 forwarded to this machine in your router, and
  `PUBLIC_ADDRESS` set to your public IP.

## Quick start

```sh
mkdir gb-parano1d-solo && cd gb-parano1d-solo
curl -fsSLO https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/docker-compose.yml
curl -fsSL https://raw.githubusercontent.com/gustlborg/gb-parano1d-solo/main/.env.example -o .env
nano .env                 # set WALLET_ADDRESS=o1…
docker compose up -d
docker compose logs -f
```

The node synchronizes within a minute or two. Once the log says
`node is synchronized and ready for mining`, start your rigs.

## Connect your rigs

Point every rig at `stratum+tcp://<ip-of-this-machine>:3333`. The user name
is only a label (your address or anything else) and does not change where
payouts go.

gb-parano1d-miner, Linux:

```sh
./gb-parano1d-miner cuda-mine --rpc stratum+tcp://192.168.1.10:3333 --coinbase o1… --worker rig1 --search-binary ./gb-parano1d-engine
```

HiveOS flight sheet: pool URL `stratum+tcp://192.168.1.10:3333`, wallet `o1…`.

## Status

Dashboard: open `http://<this-machine>:9640` in a browser (only on the machine
itself unless you set `STATS_BIND_HOST=0.0.0.0` in `.env`). The "Last template"
and "Template delivery" tiles show how long your rigs waited for new work after
each network block, and the node's own share of that time.

```sh
docker exec gb-parano1d-solo gbsolo status    # node, rigs, hashrate, effort, blocks
docker exec gb-parano1d-solo gbsolo workers   # per rig
docker exec gb-parano1d-solo gbsolo blocks    # blocks found
curl -s 127.0.0.1:9640/stats                  # everything as JSON
```

## Solo mining means variance

A block is found when one of your rigs hits the network target. With a small
farm that can take days; `gbsolo status` shows the expected time per block for
your hashrate.

## Update

```sh
docker compose pull && docker compose up -d
```

Data (node database, pool state, found blocks) stays in the `gbsolo-data`
volume.

## Settings (.env)

| Variable | Default | Meaning |
|---|---|---|
| `WALLET_ADDRESS` | – | Your o1… address (required) |
| `STRATUM_PORT` | `3333` | Port for your rigs |
| `P2P_PORT` | `9600` | Node P2P port, forward it in your router |
| `PUBLIC_ADDRESS` | – | Your public IP when `P2P_PORT` is forwarded |
| `P2P_PEERS` | `16` | Outbound peers of the node (8–16) |
| `STATS_BIND_HOST` | `127.0.0.1` | `0.0.0.0` shows the dashboard to your LAN |
| `NODE_THREADS` | all | CPU threads for block proofs |
| `STATUS_INTERVAL` | `60` | Seconds between status lines in the log |
| `STATS_PORT_HOST` | `9640` | Host port of the dashboard |
| `GBSOLO_VERSION` | `latest` | Image version to run (e.g. `0.1.2`) |
| `TLS_ENABLED`, `TLS_CERT`, `TLS_KEY` | off | Stratum over TLS (needs a real certificate) |

## Developer fee

The pool takes a developer fee of 4 % of your mining work. It is charged per
round, not per block: on average one round in twenty-five is mined with the
developer's address, all other rounds pay 100 % of the block to your wallet.
Rounds are chosen by the work your rigs have done, so pausing or restarting
never changes the share, and the long-run fee is exactly 4 %. gb-parano1d-miner
pays nothing on top: its own fee connection mines the same template as your
rigs, so with it the total is 4 %.

## License

The pool program is proprietary (© gustlborg). The image contains a modified
build of the ParanO(1)d node, licensed under the Apache License 2.0; see
`/usr/share/doc/gb-parano1d-solo/` in the image.
