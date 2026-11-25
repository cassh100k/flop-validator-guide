# Running a FLOP Labs Validator on Devnet

Step-by-step guide to the **administrator-assisted devnet**, for package
**validator-cs-be91fe83 0.4.1-devnet.1** (current as of 25 Nov 2026).

> **Unofficial community guide.** Cross-check the official docs at
> [docs.flop.network](https://docs.flop.network) and [ask.flop.finance](https://ask.flop.finance).
> Nothing here grants admission. Release inputs change: verify them yourself before you act.

**Getting ready for testnet?** See the separate **[FLOP Testnet Guide](https://github.com/cassh100k/flop-testnet-guide)**.

---

## Honest facts first

- **Devnet earns nothing.** It proves your setup. No rewards, no airdrop weight, no promised
  testnet seat. Testnet is the network that counts (see the **[testnet guide](https://github.com/cassh100k/flop-testnet-guide)**).
- **$FLOP is not live.** Anyone selling, minting or "airdropping" it to you is a scammer.
- **Admission is gated.** FLOP operations funds (or sponsors) and activates validators. Running the
  software does not put you on the network by itself.
- **A validator is a CPU node.** No GPU or TEE needed.

---

## Prerequisites

- A **dedicated Linux amd64 host** with persistent, encrypted storage. Planning estimates:
  8 dedicated cores, 32 GB RAM, 4 TB NVMe (incl. DA custody), reliable 1 Gbps. These are targets,
  not measured devnet minimums, but validators are judged on uptime: don't use a busy shared box.
- Docker Engine + Compose, **Node.js 24**, **pnpm 12.14.0**, `curl`, `jq`
- **Tailscale** (installed, logged out) and **age** (`sudo apt install age`)

```sh
corepack enable && corepack prepare pnpm@12.14.0 --activate
sudo apt-get install -y age jq curl
curl -fsSL https://tailscale.com/install.sh | sh   # install only, do NOT log in yet
```

---

## 1. Download and verify the package

Get the archive and its `.sha256` from the official validator page:
[explorer.devnet.flop.network/validator](https://explorer.devnet.flop.network/validator)
(files live under `/validator-downloads/`). **Never run an operator package you haven't verified.**

```sh
sha256sum -c validator-cs-be91fe83-0.4.1-devnet.1.tar.gz.sha256   # must print OK
mkdir release && tar -xzf validator-cs-be91fe83-0.4.1-devnet.1.tar.gz -C release
cd release/validator-cs-be91fe83
sha256sum -c SHA256SUMS                                           # every file must be OK
```

## 2. Preflight, install, prepare

```sh
node preflight.mjs                              # prints genesis, finalized height, peers, "verified"
pnpm install --frozen-lockfile --ignore-scripts # no package can run code during install
pnpm onboard prepare
```

`prepare` creates `account.json` and `.env` (mode 600) and prints your **hostname** and **public
SS58 address**. Reruns keep the same files.

> 🔒 `account.json` and `.env` are your validator keys. Back them up privately. Never paste them,
> your seed, the node keystore or a private SSH key anywhere.

**Check the genesis** before you ever sign anything. It must be
`0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631` on the public RPC
(`https://rpc.devnet.flop.network`), on [faucet.flop.network/chain](https://faucet.flop.network/chain)
and on your own node (`chain_getBlockHash([0])`).

## 3. Send three public values

Send these to your FLOP operations contact (if you don't have one, ask for an introduction in
[flop-labs/network#10](https://github.com/flop-labs/network/issues/10); posting there does not grant
admission):

1. the printed **hostname**
2. the printed **public SS58 address**
3. the host's **SSH public key**: `cat /etc/ssh/ssh_host_ed25519_key.pub`

Then confirm that SSH key on a **second channel** (e.g. a call) by reading out its fingerprint:

```sh
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub   # read the SHA256:... value
```

The public host key is only used to encrypt your network key. It does not let anyone log in.

## 4. Receive your network key, funding and peer list

Operations sends back:

- a **single-use Tailscale key** encrypted to your SSH host key, with the exact `tailscale up`
  command. Save it as `tailnet-key.age` and run the command; **it expires 24 hours after issue**.
- **funding** for your account, *or* a **`validator-grant.json`** (sponsored registration).
- **`da-peers.json`**, the approved private DA peer list. Save it beside the README; join refuses to
  start without a valid one. Don't replace it with public gateway URLs.

Keep Tailscale SSH off: `sudo tailscale set --ssh=false`. If you want no inbound services except
P2P/DA, enforce that allowlist on the tailnet interface (incl. IPv6 and Docker forwarding) while
keeping your own management access. Operations verifies access before joining.

## 5. Join and say "ready"

```sh
pnpm onboard join
```

Join fills in your Tailscale IP, starts the pinned node and DA worker, waits for sync and advancing
finality, and registers the account (locking the live minimum stake). With a
`validator-grant.json`, it registers through the sponsor lane instead: **don't register the account
any other way first**, the lane only accepts new accounts. If the lane is full, join stops and
registers nothing.

When it prints **Ready**, tell operations **"ready"**. They read your keys and queue position from
the chain. Don't paste keys, hashes, logs or config into chat.

If join fails, keep all files and volumes and contact operations. The local `.join-attempt` marker
stops a partial submission from being repeated blindly.

## 6. Confirm activation

Activation can take two session boundaries. After operations confirms:

```sh
pnpm onboard verify   # read-only: active consensus keys, peers, advancing finality
pnpm onboard status   # any time: queued / deferred (with reason) / promoted / active
```

A healthy container or a registration record alone does **not** mean you're active.

---

## Network facts

| | |
|---|---|
| Network | FLOP Devnet, raw spec `flop_devnet_ts` |
| Genesis | `0x99c172c095340ace980631f9123544c6bebd4839f3437e463533422d8be2d631` |
| Public RPC | `https://rpc.devnet.flop.network` / `wss://rpc.devnet.flop.network` |
| Raw spec | `explorer.devnet.flop.network/validator-downloads/flop_devnet_ts-99c172c095340ace.raw.json` |
| Raw spec SHA-256 | `3b213c38e7678285bc38fbdfded5dbe3d6acbd16e836b14aaaeef2f6ae1cd16e` |
| Ports | P2P TCP 30333 (Tailscale IP), DA host TCP 7001 → container 7000. RPC 9944 and metrics 9615 stay on loopback |

Image digests and the reserved peer mesh are pinned in `manifest.json` and the bundled Compose
file. Verify both Cosign signatures as described in
[docs.flop.network/docs/run-validator](https://docs.flop.network/docs/run-validator).

---

## Security checklist

- Dedicated host, isolated from wallets, secrets and production services.
- Verify every checksum (archive + `SHA256SUMS`), the raw spec hash and the genesis before signing.
- Back up `account.json`, `.env`, the node keystore **and both Docker volumes**. A database
  snapshot cannot restore keys.
- **Never run two processes with the same session keys.**
- Before activation, `docker compose stop` keeps your data. After activation, arrange withdrawal with
  operations before stopping. **Never `docker compose down -v`** on retained volumes.
- Code or image rollbacks cannot undo registration, stake locks, promotion, slashing or DA duties.

---

*Community-maintained. Not affiliated with FLOP Labs. $FLOP is not live; this is not financial advice.*
