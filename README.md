# Running a FLOP Labs Validator (Devnet)

A practical, step-by-step guide to standing up a **FLOP Labs** validator on the
administrator-assisted **devnet**, and getting your machine proven so it's ready on
testnet day one.

> **Unofficial community guide.** Always cross-check the official docs at
> [docs.flop.network](https://docs.flop.network) and
> [ask.flop.finance](https://ask.flop.finance). Nothing here grants admission, and
> network identity / release inputs change, verify them yourself before you act.

---

## What this is

[FLOP Labs](https://flop.network) is a verified-AI-inference network: miners run measured
inference on GPUs, **validators** author and finalize blocks (BABE + AlephBFT) and serve data
availability. A validator is a **CPU node — no GPU or TEE required**, which makes it the
lowest-barrier way to participate.

A few honest facts before you spend time or money:

- **Devnet earns nothing.** It's a throwaway network for proving your setup. No rewards, no
  airdrop weight. The only reason to join is to be proven and live when the **testnet** opens
  (the testnet is what counts toward the genesis allocation).
- **$FLOP is not live.** There is no token to buy, mint, claim, or trade yet. Anyone offering to
  sell you one is a scammer.
- **Admission is gated.** The team funds and activates your validator; running the software does
  not put you on the network by itself.

---

## Prerequisites

- A **dedicated Linux amd64 host** with persistent, encrypted storage. Published planning
  estimates are ~8 cores, 32 GB RAM, 4 TB NVMe (incl. DA custody), reliable 1 Gbps. These are
  targets, not measured devnet minimums, but a validator is judged on uptime, so don't run it on
  a busy, shared box.
- **Docker Engine + Compose**
- **Node.js 24+**
- **pnpm 12.14.0**
- **Tailscale** (installed, logged out — used to receive the network key)
- **age**, **jq**, **curl**

```sh
# examples (adjust to your distro)
corepack enable && corepack prepare pnpm@12.14.0 --activate
sudo apt-get install -y age jq curl
curl -fsSL https://tailscale.com/install.sh | sh   # install only; do NOT log in yet
```

---

## Steps

### 1. Get the package and verify it

Download the current validator package and checksum from the official validator page
([explorer.devnet.flop.network/validator](https://explorer.devnet.flop.network/validator)).
**Never run an operator package you haven't checksum-verified.**

```sh
# after downloading <package>.tar.gz and its .sha256
sha256sum -c <package>.tar.gz.sha256      # must print: OK
tar -xzf <package>.tar.gz
cd <package-dir>
sha256sum -c SHA256SUMS                    # verifies every file in the package
```

### 2. Preflight

Confirms your host can reach the devnet and prints the live genesis hash, finalized height and
peer count. Stop if the genesis hash differs from the published release.

```sh
node preflight.mjs
# -> {"genesis":"0x...","finalized":<n>,"peers":<n>}
#    "<package> validator preflight: verified"
```

### 3. Install dependencies (no install scripts)

`--ignore-scripts` means no package can run arbitrary code during install — a sane default for
any operator tooling.

```sh
pnpm install --frozen-lockfile --ignore-scripts
```

### 4. Prepare your account

Generates your validator identity. It prints **two public values** — your **hostname** and your
**public SS58 address**.

```sh
pnpm onboard prepare
```

> 🔒 **Keep `account.json` and `.env` private and on this host only.** They are your validator's
> keys. Never paste them into chat, issues, or anywhere public. Never share a seed phrase.

### 5. Request admission

Send only the **two public values** (hostname + SS58 address) to your FLOP operations contact.
If you don't have one, request an introduction in
[flop-labs/network issue #10](https://github.com/flop-labs/network/issues/10) — posting there does
not grant admission, and no response deadline is promised.

### 6. Join and go "ready"

Operations sends back an **encrypted network key** (decrypt with `age` / Tailscale as instructed)
and **funding**. Then run the package's documented `join` step and signal **ready**. "Ready" means
ready for operations to review, not already active — the team checks registration and activates
the validator.

---

## Security checklist

- Run on a **dedicated host**, isolated from any wallets, secrets, or production services.
- **Verify every checksum** (package archive and `SHA256SUMS`) and the **genesis hash** before
  signing anything.
- Keep `account.json`, `.env`, and all signing files **private and backed up**; never reuse
  validator keys across machines.
- Treat the devnet as **disposable**. Don't commit real funds; there's nothing to buy.

---

## Useful links

- Docs: <https://docs.flop.network>
- Docs search / MCP: <https://ask.flop.finance> · `run-validator`, `run-operator`
- Validator page: <https://explorer.devnet.flop.network/validator>
- Network repo & onboarding requests: <https://github.com/flop-labs/network>

---

*Community-maintained. Not affiliated with FLOP Labs. Verify everything against the official
sources before running an operator node. $FLOP is not live; this is not financial advice.*
