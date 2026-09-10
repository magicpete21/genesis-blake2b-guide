# Assistant briefing: build a sovereign BLAKE2b Bitcoin miner

**How to use this file:** paste it into your AI assistant (Claude, ChatGPT, etc.) along with a description of your hardware, and say: *"Walk me through this one step at a time. I'll run each command and paste the output back."* It is the runbook behind the "Genesis" journal, written for a machine to follow with you.

---

## Instructions to the assistant

1. **Work from official sources, not memory.** Before each phase, fetch and read the relevant page: `bitcoin-blake2b.org/getting-started`, `bitcoin-blake2b.org/mining`, `bitcoin-blake2b.org/nodes`, the Knots release notes for `v29.4.1.knots20260508`, and the README of `github.com/CONVOYMining/datum_gateway`. If a version, URL, or config key on those pages differs from this file, **the official page wins**. Say so explicitly.
2. **One step at a time.** Give one command or one small group of commands, wait for the user to paste the output, read it, then continue. Do not dump the whole procedure at once.
3. **Never ask for or accept secrets.** No seed words, no private keys, no passwords, no RPC credentials. If the user pastes one by accident, tell them to rotate it. Public descriptors and `bc1q…` addresses are fine.
4. **The user verifies security-critical steps themselves.** You may explain GPG verification, but the user runs it and reads the result. You may explain address derivation, but the user compares the payout address on the hardware wallet's own screen. Do not tell them to skip these.
5. **Software only from:** bitcoinknots.org, github.com/bitcoinknots/bitcoin, github.com/CONVOYMining/datum_gateway. Anything else claiming to be "the fork client" is presumptively malware.
6. **Adapt to the hardware.** Laptop vs. desktop vs. Pi, dual-boot vs. wipe, internal vs. external drive, 16 GB vs. 32 GB RAM. Ask before assuming.
7. **When something errors, ask for the exact error text** and the last ~30 lines of the relevant log before proposing a fix.

## Target state

```
[HW wallet] ──(public descriptor only)──▶ [Node box: Ubuntu 24.04]
                                             ├─ Bitcoin Knots v29.4.1.knots20260508 (BLAKE2b chain, active from block 961640)
                                             ├─ CONVOY DATUM Gateway, solo mode, Stratum :23334, dashboard :7152
                                             └─ watch-only wallet "fork_mining" holding the payout address
[BLAKE2b ASICs] ──Ethernet──▶ stratum+tcp://<node-ip>:23334
```

## Phase 1: Prepare the machine

Goal: Ubuntu 24.04, SSH enabled, never sleeps, fixed LAN IP.

- If dual-booting with Windows: back up, disable Fast Startup, suspend BitLocker, shrink `C:` from Disk Management (disable pagefile/hibernation if it won't shrink), then "Install Ubuntu alongside Windows" from a Desktop ISO. Unplug any chain drive during install. Set GRUB default to Ubuntu.
- If wiping: Ubuntu Server or Desktop, either is fine.
- After install:
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install -y openssh-server tmux curl gnupg git
  ```
- Disable sleep/hibernate on lid close and on AC. Cap laptop battery charge at 60–80% if firmware allows.
- Give the box a DHCP reservation in the router. Record the IP: `ip addr`.

**Check:** user can SSH in from another machine; `uptime` after closing the lid shows it stayed up.

## Phase 2: Chain storage

Goal: ~1 TB at `/mnt/chain`, auto-mounted at boot.

```bash
lsblk -f                         # identify the drive; confirm with the user before formatting
sudo mkfs.ext4 /dev/sdX          # ONLY if the drive is new/empty
sudo mkdir -p /mnt/chain
# add to /etc/fstab, using the UUID from lsblk -f:
# UUID=<uuid>  /mnt/chain  ext4  defaults,nofail  0  2
sudo mount -a && df -h /mnt/chain
mkdir -p /mnt/chain/bitcoin
```

**Check:** `df -h /mnt/chain` shows the drive; survives a reboot.

## Phase 3: Install and verify Bitcoin Knots

Goal: `bitcoind --version` prints `v29.4.1.knots20260508`, signatures verified.

1. Download the `x86_64-linux-gnu` (or `aarch64`) tarball, `SHA256SUMS`, and `SHA256SUMS.asc` from bitcoinknots.org or the GitHub release.
2. Import signer keys from `github.com/bitcoinknots/guix.sigs` (do not click through TLS warnings to get keys).
3. Verify (user runs, user reads):
   ```bash
   sha256sum --check SHA256SUMS --ignore-missing
   gpg --verify SHA256SUMS.asc SHA256SUMS
   ```
4. Install:
   ```bash
   tar xzf bitcoin-*.tar.gz
   sudo install -m 755 bitcoin-*/bin/* /usr/local/bin/
   bitcoind --version
   ```

## Phase 4: Configure and sync

`~/.bitcoin/bitcoin.conf` (assistant: confirm the headline line against the official getting-started page for this release before the user saves it):

```
datadir=/mnt/chain/bitcoin
prune=0
dbcache=8192            # ~1/4 of RAM is a reasonable ceiling
server=1
blake2b_headline=8-30 NYPost Deride And Conquer
rpcbind=127.0.0.1
rpcallowip=127.0.0.1
rpcuser=<user>
rpcpassword=<long random password; the assistant must not see it>
blocknotify=curl -fsS -o /dev/null http://127.0.0.1:7152/NOTIFY
listenonion=0
```

First run in the foreground inside tmux so the log is visible:
```bash
tmux new -s node
bitcoind -printtoconsole
```
Expected: initial block download (days on a fresh drive), or, if reusing legacy-chain data, a walk-back to block 961640 followed by a short BLAKE2b sync. A large reorg on that one startup is normal.

**Checks:**
```bash
bitcoin-cli getblockchaininfo | grep -e blocks -e verificationprogress
bitcoin-cli getdeploymentinfo | grep -A3 '"blake2b"'   # height 961640, active true
bitcoin-cli getblocktemplate '{"rules":["segwit","blake2b"]}' | grep '!blake2b'
```

## Phase 5: Public node and service

- Forward TCP 8333 on the router to the node's IP. Confirm the router's WAN IP equals the public IP (no double NAT).
- **Check:** `bitcoin-cli getpeerinfo | grep -c '"inbound": true'` becomes nonzero within hours.
- Optional but recommended once stable, run as a service (`/etc/systemd/system/bitcoind.service`):
  ```
  [Unit]
  Description=Bitcoin Knots (BLAKE2b)
  After=network-online.target mnt-chain.mount
  Requires=mnt-chain.mount
  [Service]
  User=<user>
  ExecStart=/usr/local/bin/bitcoind
  Restart=on-failure
  TimeoutStopSec=600
  [Install]
  WantedBy=multi-user.target
  ```
  Stop the tmux instance first; two bitcoinds cannot share a datadir.

## Phase 6: CONVOY DATUM Gateway (solo)

```bash
sudo apt install -y cmake build-essential pkg-config libmicrohttpd-dev libjansson-dev libcurl4-openssl-dev libsodium-dev
git clone https://github.com/CONVOYMining/datum_gateway.git ~/datum_convoy
cd ~/datum_convoy
git log --oneline | grep 56c31f4      # must print a line; otherwise the build makes invalid blocks
cmake . && make                        # or per README
```

`~/datum_gateway_config.json` (solo recipe; CONVOY's shipped example is a pooled config and will not work for solo):
```json
{
  "bitcoind": { "rpcuser": "<user>", "rpcpassword": "<password>", "rpcurl": "http://127.0.0.1:8332" },
  "stratum":  { "listen_addr": "0.0.0.0", "listen_port": 23334 },
  "api":      { "listen_addr": "0.0.0.0", "listen_port": 7152, "admin_password": "<change-me>", "modify_conf": true },
  "mining":   { "pool_address": "<bc1q payout address>", "coinbase_tag_primary": "<tag>", "allow_hasher_time_rolling": false },
  "datum":    { "pool_host": "", "pooled_mining_only": false }
}
```
Run: `./datum_gateway -c ~/datum_gateway_config.json` (in its own tmux window).

**Checks (first second of log):** `NON-POOLED MINING!` banner, `API listening on … 7152`, `Stratum V1 Server Init complete`, then `NEW NETWORK BLOCK` and a template line with a reward and tx count. If it exits immediately, `pooled_mining_only` is probably still true. Dashboard at `http://<node-ip>:7152` → Config → mining shows `pool_address`.

## Phase 7: Payout wallet (hardware keys, node eyes)

1. User creates a **fresh seed** on their hardware wallet, dedicated to this chain, **from their own entropy** (e.g. dice), not the device RNG. User then **verifies the seed on a second independent device** (restore the words elsewhere; master fingerprint and first receive address must match). The assistant does not see the words or the rolls.
2. User exports the **native SegWit descriptor** (public only) to microSD, moves it to the node.
3. Create a watch-only descriptor wallet and import:
   ```bash
   bitcoin-cli createwallet "fork_mining" true true "" false true
   bitcoin-cli -rpcwallet=fork_mining importdescriptors '[
     {"desc":"<receive descriptor with checksum>","timestamp":"now","active":true,"internal":false},
     {"desc":"<change descriptor with checksum>","timestamp":"now","active":true,"internal":true}]'
   bitcoin-cli -rpcwallet=fork_mining getnewaddress "datum_payout" bech32
   ```
   If the checksum is rejected, `bitcoin-cli getdescriptorinfo "<desc>"` returns the correct one.
4. **User compares the address to the hardware wallet's own screen.** Only then put it in `pool_address` and restart DATUM.
5. **Check:** `bitcoin-cli -rpcwallet=fork_mining getaddressinfo <addr>` shows `ismine: true`, `iswatchonly: true`.

A temporary hot wallet (`createwallet "temp_mining"`) is acceptable for testing at zero balance; it must not be the destination once real hashrate is attached.

## Phase 8: Miners

- 240 V, one miner per dedicated circuit, Ethernet only, DHCP reservation per miner.
- Miner pool URL `stratum+tcp://<node-ip>:23334`, worker any name, password `x`.
- **Check:** DATUM dashboard shows the worker, accepted shares, hashrate near nameplate. `BLOCK FOUND` hash equals new chain tip; a second `duplicate` submit line is benign.

## Done criteria

Node at tip on BLAKE2b rules · inbound peers > 0 · DATUM solo with valid template · payout address verified on hardware wallet · miners submitting accepted shares.
