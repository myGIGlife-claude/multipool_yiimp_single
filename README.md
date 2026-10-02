# multipool_yiimp_single
Installation files for YiiMP single server

#### These files do nothing on their own. Please go to https://github.com/myGIGlife-claude/Multi-Pool-Installer

## Supported systems

- Ubuntu 22.04 LTS (Jammy Jellyfish)
- Ubuntu 24.04 LTS (Noble Numbat)
- Ubuntu 26.04 LTS (Resolute Raccoon)

64-bit (x86_64) only. Ubuntu 16.04, 18.04 and 20.04 are no longer supported.

## Notes

- PHP is installed from the [Ondrej PHP PPA](https://launchpad.net/~ondrej/+archive/ubuntu/php)
  using the `PHP_VERSION` set in `/etc/multipool.conf` (falls back to the Ubuntu PHP packages on
  releases the PPA does not support yet: Ubuntu 26.04 gets its own PHP 8.5). MariaDB, nginx and
  certbot come from Ubuntu.
- The YiiMP source is cloned from https://github.com/myGIGlife-claude/yiimp. To install from a
  fork set `YIIMP_REPO` (and optionally `YIIMP_BRANCH`) in the environment before starting the installer.
- Database user names and passwords are saved in `$STORAGE_ROOT/yiimp/.my.cnf` (readable by the
  installing user only).
- `stratum start|stop|restart <algo>` starts or stops the stratum of one algo. Without dedicated
  coin ports, most Bitcoin-stratum algos, Equihash and yespowerRES start at boot. The KawPoW
  family and randomx must be started by hand.
- `max_cons_per_ip` in the `[STRATUM]` section of an algo's `.conf` limits the connections from
  one IP address (IPv6: per /64). The default `0` means no limit.
- The `verthash` stratum (Vertcoin) needs its 1.2 GB data file. Create it once:
  `cd $STORAGE_ROOT/yiimp/site/stratum && ./verthash_gen verthash.dat`.

## Algos with their own stratum protocols

These algos use a protocol other than the Bitcoin stratum. Each needs a few
extra steps on the coin daemons.

### KawPoW family (kawpow, evrprogpow, meowpow, firopow, sccpow, meraki)

- Coins: RVN, XNA, NEOX, SATOX, EVR, MEWC, FIRO, SCC, TLS.
- Ports: 9501-9506.
- Each stratum keeps the ProgPoW verification caches of the current and next epoch in memory, about 95-140 MB per coin.
- Because of that memory use they are not started at boot. Start the ones you need with `stratum start kawpow` (or firopow, meowpow...).
- Coins whose getblocktemplate has a `!segwit` rule (MEWC 30.x, TLS 3.x) need `usesegwit` enabled on the coin.

### Equihash (equihash, equihash144, equihash192) and yespowerRES

- Coins: ZEC, KMD, ARRR (200,9); BTG, BTCZ, GLINK (144,5); YEC, ZER, ZCL (192,7); RES.
- Ports: 9600-9602 and 9650.
- Daemon setup:
  - zcashd 6.x needs `mineraddress=<pool t-address>`. The coinbase, founders reward and funding streams come from the daemon.
  - BTG needs `usesegwit` enabled on the coin.
  - Resistance needs its Sapling/Sprout parameters in `~/.resistance-params`.
- Every daemon needs at least one peer to answer getblocktemplate, and `blocknotify=/usr/bin/blocknotify 127.0.0.1:<stratum port> <coin id> %s`.
- The Equihash personalization of a coin can be set in the stratum `.conf`: `equihash_personalization`, or an `[EQUIHASH]` section with `SYMBOL = personalization`.

### Decred (decred, BLAKE3)

- Decred's proof of work has been BLAKE3 since block 794,368. The `decred` stratum (port 3252) mines it through dcrd's getwork.
- dcrd must run with `--miningaddr=<pool wallet address>`. The block reward is paid there.
- The coin's RPC can point at dcrwallet (RPC passthrough, not SPV).
- Blocks are confirmed by `blocknotify-dcr` from the YiiMP source. Building it needs Go 1.21 or newer:
  ```
  make -C blocknotify-dcr install
  blocknotify-dcr -stratum 127.0.0.1:3252 -coinid <id> -rpcuser <user> -rpcpass <pass> -rpccert <dcrd rpc.cert>
  ```
- Keep the stratum difficulty at 1 or more.

### RandomX (randomx, Monero)

- Coin: XMR. Port: 9701. Miners use the xmrig protocol (xmrig, XMRig-proxy).
- It is not started at boot. Start it with `stratum start randomx`.
- **Resources:**
  - Memory: up to about 800 MB of RandomX caches.
  - CPU: each share costs about 20-40 ms of one core.
- **monerod:**
  - Run it with `--rpc-login <user>:<pass>`.
  - Also pass `--block-notify '/usr/bin/blocknotify 127.0.0.1:9701 <coin id> %s'`.
- **monero-wallet-rpc:**
  - It runs the pool wallet. That wallet's address must be the coin's master wallet.
  - Payouts are sent from it with `transfer_split`.
- **Coin setup:**
  - Set the coin's *RPC Type* to `XMR`.
  - Add the wallet to `serverconfig.php`: `$configWalletRPC['XMR'] = 'host:port:user:pass';`. Without that line it is expected on the daemon host at RPC port + 1.
- **Testing:** payouts have been tested only once, on a private test network.
- To connect a different stratum program (for algos this stratum can't handle), see [docs/BRIDGE.md](https://github.com/myGIGlife-claude/yiimp/blob/next/docs/BRIDGE.md) in the YiiMP source.
