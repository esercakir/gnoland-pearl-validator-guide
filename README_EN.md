# Gno.land Pearl Testnet Validator Setup Guide

This guide walks through setting up a full node on the `chain/pearl` branch and registering as a validator candidate, including real issues encountered during setup and how they were resolved. Official source: [VALIDATOR.md](https://github.com/gnolang/gno/blob/chain/pearl/misc/deployments/pearl.gno.land/VALIDATOR.md)

> Tested with `gnoland v1.0.0-rc.0` (chain/pearl branch). Flag names may differ on other versions.

---

## Requirements

- A Linux server (cloud/VPS, on-prem, or data-center) with Go installed
- A public IP with port `26656` (P2P) reachable from the internet
- `git`, `make`, and standard build tools

---

## 1. Build the binaries from source

```bash
git clone https://github.com/gnolang/gno.git gno-pearl
cd gno-pearl
git checkout chain/pearl
make -C gno.land install.gnoland install.gnokey
```

This installs `gnoland` and `gnokey` under `$GOPATH/bin` (usually `/root/go/bin`). Verify:

```bash
which gnoland
which gnokey
```

Alternative: build a Docker image with `docker build --target gnoland -t gnoland:pearl .`, or use prebuilt binaries from the [release page](https://github.com/gnolang/gno/releases/tag/chain%2Fpearl) and the `ghcr.io/gnolang/gno/gnoland` container registry.

---

## 2. Download and verify the genesis file

```bash
wget -O genesis.json https://github.com/gnolang/gno/releases/download/chain/pearl/genesis.json
shasum -a 256 genesis.json
```

The output must **exactly match**:

```
c45fe60c8c8a1f859d9e4d5aad7ce4d100ff0eb78302e71318ba0de481a8dc91  genesis.json
```

If it doesn't match, don't use the file — re-download it.

---

## 3. Node configuration

> **Note:** In this build, `config init` / `secrets init` do **not** support a `--home` flag. Instead, each command takes its own directory flag (`-config-path`, `-data-dir`). `gnoland start` expects a single `-data-dir` root directory containing `config/` and `secrets/` subdirectories.

```bash
mkdir -p ~/pearl-node

gnoland config init -config-path ~/pearl-node/config/config.toml
gnoland secrets init -data-dir ~/pearl-node/secrets
```

### Setting config values

**Network-wide — must be identical across all nodes:**

```bash
gnoland config set p2p.persistent_peers "g1m37xukfq6yl555k93fcyzns83qnmgyax9zm875@seed-1.pearl.testnets.gno.land:26656,g1ngukqd3khekaqjf90k45cglzm0l25wwzl2fkn2@seed-2.pearl.testnets.gno.land:26656" -config-path ~/pearl-node/config/config.toml
gnoland config set application.prune_strategy syncable -config-path ~/pearl-node/config/config.toml
gnoland config set consensus.timeout_commit 3s -config-path ~/pearl-node/config/config.toml
gnoland config set consensus.peer_gossip_sleep_duration 10ms -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.flush_throttle_timeout 10ms -config-path ~/pearl-node/config/config.toml
```

**Node-specific:**

```bash
# Get your public IP
curl -4 ifconfig.me

gnoland config set moniker "your-node-name" -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.external_address "SERVER_PUBLIC_IP:26656" -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.pex true -config-path ~/pearl-node/config/config.toml
```

⚠️ Set `p2p.external_address` to your **actual public IP, not a placeholder** — an invalid/unresolvable value will prevent the node from starting (`invalid p2p external address: unable to look up IP`).

**Recommended:**

```bash
gnoland config set mempool.size 10000 -config-path ~/pearl-node/config/config.toml
gnoland config set p2p.max_num_outbound_peers 40 -config-path ~/pearl-node/config/config.toml
```

For a sentry-node setup, see the [Sentry-node architecture](https://github.com/gnolang/gno/blob/master/gno.land/cmd/gnoland/README.md#sentry-node-architecture) section of the `gnoland` README.

---

## 4. Start the node

```bash
gnoland start \
  -data-dir ~/pearl-node \
  -gnoroot-dir ~/gno-pearl \
  -chainid pearl-1 \
  -genesis genesis.json \
  -skip-genesis-sig-verification
```

`-skip-genesis-sig-verification` is **required**: some genesis transactions intentionally carry invalid/placeholder signatures (e.g. the `names.Enable` call), and the node will panic on startup without this flag.

A successful start shows:

```
Genesis replay report   {"mode": "strict", "total": 90, "ok": 90, "ok_gas_differs": 0, "failed": 0, "skipped_failed": 0}
```

And since the node isn't in the validator set yet:

```
This node is not a validator    {"addr": "g1...", "pubKey": "gpub1..."}
```

**Note down the `pubKey` value from this line** — you'll need it for registration in step 6.

### Tracking sync status

```bash
curl -s localhost:26657/status | jq '.result.sync_info'
```

Once `catching_up: false`, the node has caught up to the chain tip. To watch it continuously:

```bash
watch -n 2 'curl -s localhost:26657/status | jq ".result.sync_info.latest_block_height, .result.sync_info.catching_up"'
```

---

## 5. Running as a systemd service

```bash
sudo tee /etc/systemd/system/gnoland-pearl.service > /dev/null <<'EOF'
[Unit]
Description=Gnoland Pearl Testnet Validator Node
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment=GNOROOT=/root/gno-pearl
WorkingDirectory=/root/gno-pearl
ExecStart=/root/go/bin/gnoland start \
  -data-dir /root/pearl-node \
  -gnoroot-dir /root/gno-pearl \
  -chainid pearl-1 \
  -genesis /root/gno-pearl/genesis.json \
  -skip-genesis-sig-verification
Restart=on-failure
RestartSec=5
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable gnoland-pearl
sudo systemctl start gnoland-pearl
```

⚠️ **The `Environment=GNOROOT=...` line is critical.** When running manually in a terminal, the shell may auto-detect GNOROOT, but systemd runs in a clean environment and won't know this variable, resulting in `panic: gno was unable to determine GNOROOT`. The `-gnoroot-dir` flag alone is not enough — the environment variable is also required.

Check status and logs:

```bash
sudo systemctl status gnoland-pearl
sudo journalctl -u gnoland-pearl -f
```

---

## 6. Registering as a validator candidate

### Getting the consensus public key

> **Known issue:** `gnoland secrets get validator_key -data-dir ~/pearl-node` crashes with a reflect panic in this build (`reflect: call of reflect.Value.Interface on zero Value`), both plain and with `validator_key.pub_key -raw`.
>
> **Workaround:** The `gpub1...` value is already printed in the node's startup logs (`"This node is not a validator" {"pubKey": "gpub1..."}`) — restart the service and grab it from there:
>
> ```bash
> sudo systemctl restart gnoland-pearl
> sudo journalctl -u gnoland-pearl -f | grep -m1 "pubKey"
> ```
>
> Alternatively, you can inspect the raw key file with `cat ~/pearl-node/secrets/priv_validator_key.json`, but the `pub_key.value` field there is raw base64 and needs to be converted to bech32 `gpub1...` format — pulling it from the log line is far more practical.

### Checking the operator account balance

```bash
gnokey query bank/balances/<your-g1-address> --remote https://rpc.pearl.testnets.gno.land
```

If empty, request funds from the faucet: **https://pearl.testnets.gno.land/faucet**

### Register transaction

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func Register \
  --broadcast \
  --chainid pearl-1 \
  --args "<MONIKER>" \
  --args "<short description>" \
  --args "<cloud|on-prem|data-center>" \
  --args "<your operator g1... address>" \
  --args "<gpub1... consensus pubkey>" \
  --gas-fee 1000000ugnot \
  --gas-wanted 100000000 \
  --remote https://rpc.pearl.testnets.gno.land \
  <your-gnokey-account-name>
```

> The `-home` flag wasn't needed here since the default directory (`/root/.config/gno`) already contained the account found by `gnokey list`. If you use a different directory, add `-home <path>`.

### Adding a detailed profile description (optional)

To add a longer, multi-line validator profile, use `UpdateDescription`:

```bash
gnokey maketx call \
  --pkgpath gno.land/r/gnops/valopers \
  --func UpdateDescription \
  --broadcast \
  --chainid pearl-1 \
  --args "<your operator g1... address>" \
  --args "$(cat <<'EOF'
<multi-line profile text here>
EOF
)" \
  --gas-fee 1000000ugnot \
  --gas-wanted 15000000 \
  --remote https://rpc.pearl.testnets.gno.land \
  <your-gnokey-account-name>
```

### Verifying the registration

**https://pearl.testnets.gno.land/r/gnops/valopers**

---

## 7. Joining the active validator set

The Register transaction only lists you as a **candidate**. Getting added to the active validator set (`r/sys/validators/v3`) requires a GovDAO member to submit and pass a proposal. This step is outside your control — it requires waiting on the GovDAO process after registration.

View the current active set at: **https://pearl.testnets.gno.land/r/sys/validators/v3**

---

## Troubleshooting Summary

| Issue | Cause | Fix |
|---|---|---|
| `flag provided but not defined: -home` | This build's `config init`/`secrets init` don't support `--home` | Use `-config-path` / `-data-dir` instead |
| `invalid p2p external address: unable to look up IP` | `p2p.external_address` contains a placeholder/invalid IP | Get your real IP with `curl -4 ifconfig.me` and set it |
| `panic: gno was unable to determine GNOROOT` (only under systemd) | systemd doesn't inherit shell environment variables | Add `Environment=GNOROOT=<repo-path>` to the service file |
| `secrets get validator_key` reflect panic | Known bug in this build | Grab the `gpub1...` value from the node's startup log instead |
| `unable to overwrite secret ... overwrite not enabled` | `secrets init` won't overwrite existing keys | Harmless — existing keys are used; add `-force` to regenerate from scratch |

---

## References

- Official guide: https://github.com/gnolang/gno/blob/chain/pearl/misc/deployments/pearl.gno.land/VALIDATOR.md
- Valoper list: https://pearl.testnets.gno.land/r/gnops/valopers
- Active validator set: https://pearl.testnets.gno.land/r/sys/validators/v3
- Faucet: https://pearl.testnets.gno.land/faucet
