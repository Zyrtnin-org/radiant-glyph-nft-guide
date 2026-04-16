# Radiant Blockchain Glyph NFT Implementation Guide

**Complete Technical Documentation for Building NFT Applications on Radiant**

This guide provides everything you need to implement Glyph NFTs on the Radiant blockchain, updated for **V2** (block 410,000+). It includes critical discoveries from real-world implementation, all 11 Glyph protocol types, V2 opcode reference, and updated fee calculations for the post-V2 fee increase.

Designed to be used as context for AI coding agents (Claude, Cursor, etc.) — paste the README into your session and start building. See [BUILDING_WITH_CLAUDE.md](BUILDING_WITH_CLAUDE.md) for MCP server setup and AI-assisted workflow tips.

> **FOR AI AGENTS — Start Here:**
> - **First NFT mint?** → Read sections 2 (Critical Requirements), 4 (On-Chain Images), 6 (CBOR Payload), then 7-9 (Commit/Reveal/Signing)
> - **Debugging a failed mint?** → Jump to section 12 (Common Errors) and the Appendix (opcodes, hex values)
> - **Upgrading to V2?** → Read section 15 (What's New in V2) and the Fee Calculations section for updated costs
> - **Hardware wallet (Ledger) support?** → See [`radiant-ledger-guide`](https://github.com/Zyrtnin-org/radiant-ledger-guide). Minting still requires software signing; receiving + spending Glyph UTXOs works with the community Ledger app
> - **Using Claude with MCP?** → See [BUILDING_WITH_CLAUDE.md](BUILDING_WITH_CLAUDE.md) for MCP setup and workflow tips

---

## Table of Contents

1. [Overview](#overview)
2. [Critical Requirements (Read First!)](#critical-requirements-read-first)
3. [Prerequisites](#prerequisites)
4. [Infrastructure Setup](#infrastructure-setup)
   - [Getting a Working Radiant Node](#getting-a-working-radiant-node)
   - [Networking the Node](#networking-the-node)
   - [Calling the Signer from PHP](#calling-the-signer-from-php)
5. [On-Chain Images: The `main` Field](#on-chain-images-the-main-field)
6. [Architecture](#architecture)
7. [CBOR Payload Format](#cbor-payload-format)
8. [Commit Transaction](#commit-transaction)
9. [Reveal Transaction](#reveal-transaction)
10. [Signing Challenge](#signing-challenge)
11. [Fee Calculations & Cost Analysis](#fee-calculations--cost-analysis)
12. [IPFS Integration](#ipfs-integration)
13. [Common Errors & Solutions](#common-errors--solutions)
14. [Complete Implementation Example](#complete-implementation-example)
15. [Testing & Verification](#testing--verification)
16. [Security Best Practices](#security-best-practices)
17. [What's New in V2](#whats-new-in-v2)
18. [Appendix: Quick Reference](#appendix-quick-reference)

---

## Overview

### What are Glyph NFTs?

Glyph NFTs are a protocol for creating non-fungible tokens on the Radiant blockchain. They support:

- **True NFTs** - Transferable, unique, visible in explorers and wallets
- **Singleton Ref Technology** - Uses `OP_PUSHINPUTREFSINGLETON` for uniqueness
- **CBOR Encoding** - Binary encoding for metadata (REQUIRED - see below)
- **On-Chain Images** - Embedded thumbnails for wallet display (REQUIRED - see below)
- **Container/Author Refs** - Organize NFTs into collections with provenance

### Two-Transaction Pattern

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Funding    │      │   Commit     │      │   Reveal     │
│    UTXO      │ ───► │     TX       │ ───► │     TX       │
│  (P2PKH)     │      │(nftCommit)   │      │(singleton)   │
└──────────────┘      └──────────────┘      └──────────────┘
                            │                      │
                            ▼                      ▼
                      Custom script          Singleton ref
                      validates glyph        NFT output
```

**Why Two Transactions?**

1. **Commit TX** - Creates an output with custom script that validates the glyph data
2. **Reveal TX** - Spends the commit output, embedding glyph data in scriptSig and creating the final NFT

This pattern ensures the NFT data is validated before the NFT is created.

### Container Organization: Commit vs Reveal Addresses

**IMPORTANT:** The commit address determines which container your NFTs belong to, while the reveal destination address determines who owns the NFT.

```
┌─────────────────────────────────────────────────────────┐
│ COMMIT TRANSACTION                                       │
│ - Address: Admin/Platform wallet                        │
│ - Purpose: Groups all NFTs into platform container      │
│ - Result: UTXOs and change stay with platform          │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│ REVEAL TRANSACTION                                       │
│ - destAddress: Player wallet OR admin wallet            │
│ - Purpose: Determines who owns the NFT                  │
│ - Result: NFT singleton output to owner's address       │
└─────────────────────────────────────────────────────────┘
```

**Best Practice for Platform NFTs:**
- **Commit address:** Always use your platform's wallet (keeps all NFTs in one container)
- **Reveal destAddress:** Use player's wallet if available, otherwise your platform wallet

This design ensures:
1. All NFTs are organized in your platform's container (easy to browse in explorers)
2. Players own their NFTs (can transfer and manage them)
3. Clean fund management (UTXOs stay with platform wallet)

### Glossary

Terms you'll see throughout this guide. If you've worked with Bitcoin-style
chains before, most will be familiar; a few are Radiant/Glyph-specific.

- **Photon** — Radiant's smallest unit. 1 RXD = 100,000,000 photons (same
  ratio as BTC : satoshi). Fee rates in this guide are in photons/byte.
- **Commit TX** — First of the two-transaction mint. Creates an output locked
  by a custom `nftCommitScript` that validates the glyph payload hash.
- **Reveal TX** — Second transaction. Spends the commit output, embeds the
  glyph payload in the scriptSig, and produces the final singleton-ref NFT
  output.
- **Singleton ref** — A 36-byte reference (commit_txid_reversed + commit_vout_LE)
  pushed by `OP_PUSHINPUTREFSINGLETON` (opcode `0xd8`). Enforces uniqueness —
  the same ref can never appear in two different outputs.
- **`nftCommitScript`** — The custom locking script on the commit output.
  Encodes the hash of the CBOR payload; the reveal tx must present a payload
  that hashes to the same value.
- **`"gly"` marker** — The three ASCII bytes `676c79` that prefix every glyph
  payload in a scriptSig. Used by wallets and explorers to detect Glyph NFTs.
- **Container / Author ref** — Optional 36-byte refs in the payload's `in` /
  `by` fields that group NFTs into collections and attribute authorship. Must
  be **extracted from the singleton output script** of the referenced NFT, not
  derived from its txid. See the Container and Author Refs section.
- **CBOR** — Concise Binary Object Representation (RFC 8949). The binary
  encoding used for Glyph payloads. Every wallet expects CBOR; JSON won't parse.
- **`main` field** — The on-chain thumbnail embedded in the CBOR payload.
  Wallet displays come from here, not from IPFS. Without it, your NFT is
  invisible in Glyphium.
- **`destAddress`** — The address the reveal tx sends the NFT output to. May
  differ from the commit address — typical pattern is commit = platform
  wallet, destAddress = player wallet.
- **Photonic Wallet / radiantjs** — The canonical browser wallet for Radiant
  and its underlying JS library. This guide uses radiantjs (via
  `require('@radiantblockchain/radiantjs')`) as the server-side signer.
- **Glyphium / Glyph Explorer** — Community wallets and block explorers that
  render Glyph-protocol NFTs. `https://glyph-explorer.rxd-radiant.com` is the
  usual explorer URL.
- **Block heights that matter.** V2 activated at **410,000**; the grace period
  for the new fee floor ended at **415,000**. Mainnet has been past both since
  early 2026.

---

## Critical Requirements (Read First!)

> **These three requirements are essential. Missing any of them will result in broken or invisible NFTs.**

### 1. CBOR Encoding is MANDATORY

**Problem:** If you encode NFT metadata with JSON instead of CBOR, your NFT will appear as "Unknown NFT" in Glyph wallets with no metadata visible.

**Solution:** Always use CBOR encoding. Verify the library is loaded before minting:

```javascript
if (typeof CBOR === 'undefined') {
    throw new Error('CBOR library not loaded! NFT would be unreadable.');
}
```

### 2. On-Chain Thumbnail is MANDATORY

**Problem:** If you only include an IPFS link (`loc` field), your NFT will display as a blank card in Glyph Explorer and Glyphium wallet.

**Solution:** Embed a thumbnail image in the `main` field:

```javascript
payload.main = {
    t: 'image/webp',    // MIME type (must match thumbnail format)
    b: thumbnailBytes   // Uint8Array of image data
};
```

### 3. Correct Script Construction

**Problem:** The singleton output script must NOT include a push opcode before the 36-byte ref.

```
CORRECT: d8<ref>7576a914...
WRONG:   d824<ref>7576a914...  // The 24 breaks it
```

---

## Prerequisites

### Software Requirements

1. **Radiant Node** (v2.3.0 or newer, wallet-enabled build).

   The wallet-capability story is not obvious from release titles:
   - **v2.1.2**: prebuilt linux-x64 tarballs on GitHub releases are actually ARM64
     binaries mislabeled as x64. On an Intel/AMD host you'll hit "Exec format error".
     Build from source or upgrade.
   - **v2.2.0**: prebuilt linux-x64 runs on x86_64 but **ships without wallet support**
     compiled in. `listwallets`, `listunspent`, `dumpprivkey`, and
     `signrawtransactionwithwallet` all return `-32601 Method not found`. Every
     minting flow in this guide depends on wallet RPCs, so v2.2.0 is unusable here.
   - **v2.3.0+**: prebuilt linux-x64, wallet-enabled. This is the first release
     the guide's examples have been verified against.

   Verify your binary actually has the wallet before proceeding:
   ```bash
   # Expected: [""] (default wallet loaded) or ["<wallet-name>", ...].
   # If you get error code -32601 "Method not found", you're on a
   # node-only build — upgrade.
   radiant-cli -datadir=/home/radiant/.radiant listwallets
   ```

   Docker: avoid `:latest`, pin the version, and rebuild with `--no-cache` on
   version bumps (Docker layer caching will happily serve an old binary otherwise).
   ```bash
   # Example — rebuild explicitly from a v2.3.0 Dockerfile against the official tarball
   docker build --no-cache -t radiant-core:2.3.0 -f Dockerfile.mainnet.v2 .
   docker run -d --name radiant-node \
     -p 127.0.0.1:7332:7332 \
     -p 7333:7333 \
     -v radiant-data:/home/radiant/.radiant \
     radiant-core:2.3.0
   ```

   **Bind RPC to `127.0.0.1:` on the host.** The P2P port 7333 is the only port
   that should face the public internet. See "Networking the Node" below for the
   matching `radiant.conf` settings when the node and your PHP backend run in
   separate Docker containers.

2. **Node.js** - v20+ for signing scripts (v18 EOL, v24 works for our needs).
   ```bash
   node --version  # Should be 20.x or higher
   ```

3. **radiantjs Library** - For transaction signing.

   `@radiantblockchain/radiantjs` is **not published on the npm registry.**
   Install the `chainbow` fork from GitHub — ideally pinned to a commit SHA
   you've reviewed, since this library sits directly in your signing path:
   ```bash
   # Pin to a specific commit for reproducibility + supply-chain safety
   npm install github:chainbow/radiantjs#<commit-sha>
   # Or, less safely, follow master
   npm install github:chainbow/radiantjs
   ```
   Commit a `package-lock.json` alongside so the exact resolved tarball is
   frozen across machines.

   The signing script does `require('@radiantblockchain/radiantjs')` because the
   package's own `package.json` declares the scoped name. **npm's resolved
   install path depends on npm version:** recent npm (10+) often places the tree
   at `node_modules/radiantjs/` (following the dependency key you used), not
   `node_modules/@radiantblockchain/radiantjs/`. If the scoped require path
   doesn't resolve after `npm install`, bridge them with a symlink:

   ```bash
   mkdir -p node_modules/@radiantblockchain
   ln -sfn ../radiantjs node_modules/@radiantblockchain/radiantjs
   ```

   The Dockerfile pattern in "Calling the Signer from PHP" below does this
   automatically — you don't need to do it by hand if you're installing into
   `/opt/signing-deps/`.

   In a `package.json`:
   ```json
   "dependencies": {
     "radiantjs": "github:chainbow/radiantjs#<commit-sha>"
   }
   ```

4. **CBOR Library** - For metadata encoding

   **For Node.js/backend:**
   ```bash
   npm install cbor
   ```

   **For browser/frontend - Download and include in HTML:**
   ```bash
   curl -o js/cbor.min.js "https://raw.githubusercontent.com/paroga/cbor-js/master/cbor.js"
   ```

   > ⚠️  **Pin a version you've reviewed.** Fetching from `master` at build time
   > is a supply-chain hazard — a repo takeover or network MITM can swap the CBOR
   > library and silently corrupt every glyph payload you mint. Either:
   > - Vendor the file into your repo and commit a SHA256-pinned copy, or
   > - Use an `<script integrity="sha384-...">` subresource-integrity tag when
   >   loading from a CDN.
   >
   > Be aware also of **Cloudflare / CDN caching** on unversioned JS/CSS. If your
   > nginx sends `Cache-Control: public, immutable` with a 7-day expiry, CDNs
   > will hold the stale file for a week — code fixes won't reach users. For
   > unversioned JS/CSS, use short TTL + `must-revalidate`, or embed a version
   > query string / content hash in the filename. Reserve `immutable` for
   > content-addressed assets.

   **Load in HTML (BEFORE blockchain scripts):**
   ```html
   <!-- CBOR must load FIRST -->
   <script src="js/cbor.min.js"></script>
   <!-- Then blockchain code -->
   <script src="js/radiant_chain.js"></script>
   <script src="js/glyph_minter.js"></script>
   ```

   **Verify it's loaded:**
   ```javascript
   console.log(typeof CBOR);  // Must output "object", NOT "undefined"
   ```

### Knowledge Requirements

- Basic understanding of Bitcoin-style transactions
- Familiarity with hex encoding and byte manipulation
- JavaScript or PHP programming
- Understanding of public/private key cryptography

---

## Infrastructure Setup

Most of the real pain in getting a Glyph NFT to mint comes from infrastructure,
not protocol — missing wallet RPCs on the node, the web container not being able
to reach the node, Node.js not being installed where your signing script runs,
deploy pipelines that drop `node_modules` on the floor. These three sections
walk through the setup that the example code later in this guide assumes.

### Getting a Working Radiant Node

**1. Version & wallet verification.** See the Prerequisites note on the v2.1.2 /
v2.2.0 / v2.3.0 landmines. After your container starts, run:

```bash
docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant listwallets
# Expected: [""] (default wallet loaded) or ["<wallet-name>", ...].
# If you get error code -32601 "Method not found", your binary is node-only.
# Rebuild from a v2.3.0 wallet-enabled source.
```

If you built the image from source, confirm the wallet compiled in:

```bash
docker run --rm --entrypoint=sh <image> -c 'strings /usr/local/bin/radiantd | grep -iE "^listunspent$"'
# Should print: listunspent
```

**2. Protect `wallet.dat` at upgrade time.** The wallet lives in your node's
datadir volume — e.g. `/home/radiant/.radiant/wallet.dat` inside the container.
Persisting the volume is necessary but **not sufficient** across version jumps:

- A node-only binary leaves `wallet.dat` untouched (no wallet code = nothing
  that would touch the file).
- A wallet-enabled binary loaded against an older or unexpected wallet format
  can auto-initialize a fresh `wallet.dat` at startup, silently hiding the
  original behind a blank keypool. The symptom is `getaddressinfo <addr>`
  returning `ismine: false` for an address you know you funded.

**Always copy `wallet.dat` out before upgrading or rebuilding the image.**

```bash
# Before the upgrade
docker cp radiant-node:/home/radiant/.radiant/wallet.dat ./wallet.dat.backup.$(date +%Y%m%d-%H%M%S)
sha256sum ./wallet.dat.backup.*

# If the post-upgrade wallet looks blank, stop the container, swap the backup
# into the volume, and restart:
docker stop radiant-node
cp ./wallet.dat.backup.<timestamp> \
   /var/lib/docker/volumes/<your-volume>/_data/wallet.dat
chmod 600 /var/lib/docker/volumes/<your-volume>/_data/wallet.dat
docker start radiant-node
```

After the upgrade, verify the expected address is still yours:

```bash
docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant \
  getaddressinfo <your-hot-wallet-address>
# Expect: "ismine": true, "iswatchonly": false

docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant \
  listunspent 0 9999999 '["<your-hot-wallet-address>"]' | head -20
```

**3. Rebuild with `--no-cache` on version bumps.** Docker's layer cache will
happily reuse an old binary layer even when the Dockerfile now points to a new
release. A rebuild that appears to succeed can silently keep you on the
wallet-less v2.2.0 binary. Always:

```bash
docker build --no-cache -f Dockerfile.mainnet.v2 -t radiant-core:2.3.0 .
```

…and re-check `radiant-cli --version` inside the running container to confirm.

### Networking the Node

If the node and your PHP backend run in the **same** container (unusual), you
can leave `rpcbind=127.0.0.1`. Everywhere else — separate containers, or
containers on separate Docker networks — that default breaks RPC access silently.

**In `radiant.conf`:**

```ini
# Bind to all interfaces so containers on the same Docker network can reach us.
rpcbind=0.0.0.0

# Restrict who's allowed to talk to RPC. Tighten this as much as you can —
# find your app network's actual CIDR with:
#   docker network inspect <app-network> --format '{{(index .IPAM.Config 0).Subnet}}'
# and replace the broad 172.16.0.0/12 fallback below with that CIDR.
rpcallowip=127.0.0.1
rpcallowip=172.20.0.0/16           # example: tighten to your app network
# rpcallowip=172.16.0.0/12         # broad Docker-default fallback — AVOID on
#                                  # multi-tenant hosts; any other container in
#                                  # the 172.16-31.x range can reach your RPC
#                                  # with only the rpcpassword for auth.

# Standard RPC auth — override these in an environment file, not in-repo.
rpcuser=flipperchain
rpcpassword=CHANGE_ME_IN_YOUR_ENV_FILE
```

**On the host, never bind RPC to `0.0.0.0:7332`.** An exposed RPC port + weak
password = drained wallet:

```yaml
# docker-compose.yml — correct
ports:
  - "127.0.0.1:7332:7332"   # RPC — host-local only
  - "7333:7333"              # P2P — public is fine
  - "127.0.0.1:9100:9100"    # Prometheus metrics — host-local
```

**Connecting from another container.** If your PHP backend lives in a separate
compose file / network, attach the Radiant container to that network with an
alias so app code can use a stable hostname:

```bash
docker network connect --alias radiant-rpc <app-network> radiant-node
```

Then in your app's env: `RADIANT_RPC_HOST=radiant-rpc`. DNS resolves to the
container's IP on the shared network; `rpcallowip=172.16.0.0/12` covers both
sides of the bridge.

### Calling the Signer from PHP

The Signing Challenge section later in this guide shows a Node.js script
(`scripts/sign_reveal.js`) that builds and signs the reveal transaction. It has
to be a subprocess because PHP's wallet RPCs can't sign the non-standard
`nftCommitScript`. Getting PHP to actually *invoke* it reliably in a Dockerized
deployment has a few sharp edges.

**1. Node.js has to be inside the web container.** The default
`php:8.2-fpm-alpine` image does not include Node. `sh: node: not found` is the
symptom. In your Dockerfile:

```dockerfile
RUN apk add --no-cache nodejs npm git
```

**2. Install signing deps OUTSIDE the volume mount.** If your `scripts/`
directory is bind-mounted from the host (common for rapid iteration), any
`node_modules` you install at `/var/www/html/your-app/scripts/node_modules`
gets *hidden* by the mount the moment the container starts. Install at an
image-local path and expose via `NODE_PATH`:

```dockerfile
RUN mkdir -p /opt/signing-deps && cd /opt/signing-deps && \
    echo '{"name":"signing","version":"1.0.0","dependencies":{"radiantjs":"github:chainbow/radiantjs"}}' > package.json && \
    npm install --omit=dev && \
    mkdir -p node_modules/@radiantblockchain && \
    ln -sfn ../radiantjs node_modules/@radiantblockchain/radiantjs

ENV NODE_PATH=/opt/signing-deps/node_modules
```

The symlink exists because npm installs the package using its declared name
(`radiantjs` top-level, despite the package's own `package.json` declaring
`@radiantblockchain/radiantjs`). The signing script requires the scoped path;
the symlink makes both resolve.

Also check your deploy pipeline: `rsync --exclude node_modules/` matches
*every* `node_modules` directory, so scripts-level deps never reach the server
that way. Either allow-list the specific path or install inside the image at
build time as above.

**3. `proc_close()` can lie about exit codes.** When PHP shells out to Node and
polls `proc_get_status()` in a loop to detect completion, `proc_get_status()`
reaps the process and consumes the exit code. The subsequent `proc_close()`
then returns `-1` forever, so code that does `success = ($returnCode === 0)`
treats every call as failed — even when Node exited 0 with a valid signed
transaction in stdout.

**Correct pattern**: capture the exit code the moment the status transitions
to `running=false`, and fall back to that if `proc_close()` returns `-1`.
Additionally, trust the parsed stdout JSON when it's well-formed — the signed
transaction itself is the artifact you care about, and the blockchain will
reject it if it's malformed regardless of any process exit code:

```php
function signRevealViaNode(array $params, string $scriptPath): array {
    $descriptors = [0 => ['pipe', 'r'], 1 => ['pipe', 'w'], 2 => ['pipe', 'w']];
    $proc = proc_open(['node', $scriptPath], $descriptors, $pipes);
    if (!is_resource($proc)) {
        throw new RuntimeException('failed to spawn node');
    }

    fwrite($pipes[0], json_encode($params));
    fclose($pipes[0]);

    $stdout = '';
    $stderr = '';
    $capturedExitCode = null;
    stream_set_blocking($pipes[1], false);
    stream_set_blocking($pipes[2], false);

    $start = time();
    while (time() - $start < 60) {
        $status = proc_get_status($proc);
        $stdout .= (string) stream_get_contents($pipes[1]);
        $stderr .= (string) stream_get_contents($pipes[2]);
        if (!$status['running']) {
            $capturedExitCode = $status['exitcode']; // capture BEFORE proc_close
            break;
        }
        usleep(50_000);
    }
    $stdout .= (string) stream_get_contents($pipes[1]);
    $stderr .= (string) stream_get_contents($pipes[2]);
    fclose($pipes[1]);
    fclose($pipes[2]);

    $closeCode = proc_close($proc);
    $exit = ($closeCode === -1 && $capturedExitCode !== null)
        ? $capturedExitCode
        : $closeCode;

    // Trust parsed JSON over exit code — the signed tx is the binding artifact.
    $parsed = json_decode($stdout, true);
    if (is_array($parsed) && !empty($parsed['success']) && !empty($parsed['signedTx'])) {
        return $parsed;
    }

    throw new RuntimeException("sign failed (exit=$exit): $stderr");
}
```

**4. Pass WIF via stdin JSON, never argv or env.** argv is visible in
`/proc/<pid>/cmdline`, env is visible in `/proc/<pid>/environ`. The stdin pipe
is the only channel that doesn't leave the key in the process table.

**5. Verify the signed tx before broadcasting.** The stdout-JSON-trust pattern
above accepts any well-formed `{success:true, signedTx}` and broadcasts it.
That's convenient but not enough on its own: a compromised signing script (or
a future supply-chain swap of `chainbow/radiantjs`) could emit a structurally
valid tx that *pays an attacker address*. The blockchain won't reject a valid
tx just because it wasn't the one you expected — it only rejects malformed
ones.

Before `sendrawtransaction`, decode the returned raw tx and assert the outputs
match what you asked for:

```php
// After $parsed['signedTx'] comes back:
$decoded = $rpc->call('decoderawtransaction', [$parsed['signedTx']]);

// Output 0 must be the singleton script with the expected destination pubkeyhash
$out0 = $decoded['vout'][0];
if ($out0['value'] * 100_000_000 !== $expectedNftOutputSats) {
    throw new RuntimeException('signed tx output value mismatch');
}
$expectedScript = 'd8' . $ref . '7576a914' . $expectedDestPubkeyhash . '88ac';
if (strtolower($out0['scriptPubKey']['hex']) !== $expectedScript) {
    throw new RuntimeException('signed tx output script mismatch — refusing to broadcast');
}
// Optional: also check no unexpected outputs (only the singleton + any change)
```

Without this gate, "trust the stdout JSON" is a soft trust boundary that only
the cryptographic correctness of the tx (not its semantics) defends.

**6. RPC creds never touch the browser.** Wrap every blockchain call behind a
PHP endpoint that:
- Authenticates the caller (session cookie + CSRF token for state-changing calls)
- Rate-limits by IP and by session
- Allow-lists *which* RPC methods are callable. `dumpprivkey`, `dumpwallet`,
  `walletpassphrase`, `sendtoaddress`, and related privileged methods must
  never be reachable from a browser-initiated request, ever. A single
  mis-routed `dumpprivkey` call drains the wallet in one request.

---

## On-Chain Images: The `main` Field

> **This is the most commonly missed requirement.** Without on-chain image data, your NFT will be invisible in wallets.

### Why On-Chain Images Are Required

Glyph wallets (Glyphium, Glyph Explorer) look for the `main` field to display NFT images. If this field is missing or empty:
- The NFT appears as a blank card
- No image preview is shown
- Only the NFT name (if readable) appears

IPFS links in the `loc` field are for **full-resolution backup**, not wallet display.

### The `main` Field Structure

```javascript
{
    p: [2],
    name: "My NFT",
    main: {
        t: 'image/webp',           // MIME type (must match thumbnail format)
        b: thumbnailUint8Array     // Image data as Uint8Array
    },
    loc: 'ipfs://Qm...'            // Full-res backup (optional)
}
```

### Thumbnail Size vs Cost Tradeoffs

| Format | Dimensions | Quality | Approx. Bytes | Pre-V2 Cost | Post-V2 Cost (10x) |
|--------|------------|---------|---------------|-------------|---------------------|
| JPEG   | 150px      | 65%     | 5-8 KB        | ~0.07 RXD   | ~0.7 RXD            |
| JPEG   | 200px      | 75%     | 14-18 KB      | ~0.15 RXD   | ~1.5 RXD            |
| WebP   | 200px      | 85%     | 15-17 KB      | ~0.17 RXD   | ~1.7 RXD            |
| **WebP** | **225px** | **90%** | **20-25 KB** | **~0.22 RXD** | **~2.2 RXD**      |
| PNG    | 200px      | lossless| 70-85 KB      | ~0.85 RXD   | ~8.5 RXD            |

**Recommendation:** Use **WebP format** at 225px max dimension with 90% quality. This provides:
- Sharp, clear images with excellent detail retention
- Best quality-to-size ratio of any format
- Good balance between quality and transaction costs
- Full compatibility with Glyphium wallet and Glyph Explorer

**Why WebP over JPEG?**
- WebP produces fewer compression artifacts at equivalent file sizes
- Single lossy compression step (capture as PNG, compress to WebP) vs double (JPEG capture → JPEG thumbnail)
- Supported by all modern browsers and Glyph wallets
- At 90% quality, produces noticeably sharper images than JPEG at similar file sizes

### JavaScript Thumbnail Creation

```javascript
async createThumbnail(dataUrl, maxSize = 225, quality = 0.90) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => {
            // Calculate dimensions maintaining aspect ratio
            let width = img.width;
            let height = img.height;

            if (width > height) {
                if (width > maxSize) {
                    height = Math.round((height * maxSize) / width);
                    width = maxSize;
                }
            } else {
                if (height > maxSize) {
                    width = Math.round((width * maxSize) / height);
                    height = maxSize;
                }
            }

            // Create canvas and draw scaled image
            const canvas = document.createElement('canvas');
            canvas.width = width;
            canvas.height = height;
            const ctx = canvas.getContext('2d');
            ctx.imageSmoothingEnabled = true;
            ctx.imageSmoothingQuality = 'high';
            ctx.drawImage(img, 0, 0, width, height);

            // Convert to WebP for best quality/size ratio
            canvas.toBlob((blob) => {
                const reader = new FileReader();
                reader.onload = () => {
                    const uint8Array = new Uint8Array(reader.result);
                    console.log(`Thumbnail: ${width}x${height}, ${uint8Array.length} bytes (WebP)`);
                    resolve(uint8Array);
                };
                reader.onerror = reject;
                reader.readAsArrayBuffer(blob);
            }, 'image/webp', quality);
        };
        img.onerror = reject;
        img.src = dataUrl;
    });
}
```

### Adding `main` Field to Payload

```javascript
function createGlyphPayload(photoData, metadata) {
    const payload = {
        p: [2],  // NFT protocol
        name: metadata.name,
        type: metadata.type || 'photo',
        attrs: metadata.attrs || {}
    };

    // CRITICAL: Add thumbnail for wallet display
    if (photoData.thumbnailData instanceof Uint8Array) {
        payload.main = {
            t: 'image/webp',  // WebP for best quality/size
            b: photoData.thumbnailData
        };
    }

    // Add IPFS for full resolution (optional)
    if (photoData.ipfsUrl) {
        payload.loc = photoData.ipfsUrl;
    }

    return payload;
}
```

> ⚠️  **`paroga/cbor-js` Uint8Array trap.** The `b` field above works only if
> `photoData.thumbnailData` is a real `Uint8Array`. If it's a plain `Array` of
> numbers (e.g. `Array.from(uint8)`), `paroga/cbor-js` encodes it as a CBOR
> **array of integers (major type 4)**, not a **byte string (major type 2)**.
> The bytes land on chain, but Glyph wallets look for major type 2 and render
> your NFT as a blank card. Two checks save you:
>
> 1. **At payload-build time**, assert the type: `if (!(thumbnailData instanceof Uint8Array)) throw new Error('thumbnail must be Uint8Array');`
> 2. **After encoding**, round-trip the CBOR and confirm `main.b` comes back
>    as bytes. The `in` and `by` ref fields are a good positive control —
>    they're 36-byte buffers and render correctly on existing NFTs; if
>    `main.b` decodes differently than they do, you have the trap.
>
> Common ways to end up with a plain Array instead of Uint8Array: JSON
> round-trips (`JSON.parse(JSON.stringify(payload))` converts `Uint8Array` to
> a numeric `Array`), shallow copies via `{ ...obj }` into a new plain object,
> and anything that crosses a `postMessage` / `structuredClone` boundary
> configured incorrectly. Handle `Uint8Array` as the last step before CBOR
> encoding.

---

## Architecture

### Transaction Components

#### Commit Transaction
```
Inputs:  [Funding UTXO(s)]
Outputs: [nftCommitScript Output, Change Output (optional)]

nftCommitScript validates:
  1. Glyph payload hash (double SHA256 of CBOR)
  2. "gly" marker presence
  3. Singleton ref in reveal transaction outputs
  4. P2PKH signature
```

#### Reveal Transaction
```
Inputs:  [Commit Output]
         ScriptSig: <signature> <pubkey> <"gly"> <CBOR payload>

Outputs: [Singleton NFT Output]
         Script: OP_PUSHINPUTREFSINGLETON <36-byte ref> OP_DROP <P2PKH>
```

### Key Data Structures

**36-Byte Ref Format:**
```
ref = reversed(commitTxid) + littleEndian(commitVout)
    = 32 bytes              + 4 bytes
```

**Example:**
```javascript
Commit TXID: 6afb402d085b2214b44853dad42499b5a02b823153e2789bb2bfb0e522693c26
Reversed:    263c6922e5b0bfb29b78e25331823ba0b59924d4da5348b414225b082d40fb6a
Vout: 0
Vout LE:     00000000
Ref:         263c6922e5b0bfb29b78e25331823ba0b59924d4da5348b414225b082d40fb6a00000000
```

---

## CBOR Payload Format

### Glyph Protocol Structure

```javascript
{
    p: [2],                           // Protocol: 2 = NFT (REQUIRED)
    name: "My NFT #12345",            // Token name (REQUIRED)
    type: "photo",                    // Type (optional)
    main: {                           // On-chain image (REQUIRED for display!)
        t: 'image/webp',              // MIME type (must match thumbnail format)
        b: thumbnailUint8Array        // Image bytes
    },
    in: [containerRefBytes],          // Container ref (optional)
    by: [authorRefBytes],             // Author ref (optional)
    attrs: {                          // Custom attributes (optional)
        score: 1000000,
        game: "Game Name",
        player: "Player Name"
    },
    loc: "ipfs://..."                 // IPFS full-res location (optional)
}
```

### Protocol Identifiers

| Value | Meaning |
|-------|---------|
| 1 | Fungible Token (FT) |
| 2 | Non-Fungible Token (NFT) |
| 3 | Data Storage (DAT) |
| 4 | Decentralized Mint (dMint) |
| 5 | Mutable (MUT) |
| 6 | Explicit Burn |
| 7 | Container / Collection (parent-only) |
| 8 | Encrypted Content |
| 9 | Timelocked Reveal |
| 10 | Issuer Authority |
| 11 | WAVE Naming System |

### Container and Author Refs

**CRITICAL:** Container and author refs MUST be extracted from the NFT's singleton output script, NOT calculated from the transaction ID.

#### The Problem

Many implementations calculate refs like this (WRONG):

```javascript
// ❌ WRONG - This will NOT work for container/author refs!
const txidReversed = Buffer.from(containerTxid, 'hex').reverse();
const voutLE = Buffer.alloc(4);
voutLE.writeUInt32LE(0);
const wrongRef = txidReversed.toString('hex') + voutLE.toString('hex');
```

**Why this fails:**
- The singleton ref in the output script is based on the COMMIT transaction, not the reveal transaction
- Computing from reveal txid gives you a completely different 36-byte value
- Child NFTs will reference a non-existent parent, breaking the container hierarchy

#### The Solution: Extract from Output Script

**Correct approach:**

```javascript
// ✅ CORRECT - Extract ref from the singleton output script
// 1. Get the reveal transaction
const tx = await rpc.call('getrawtransaction', [revealTxid, true]);

// 2. Get output 0's scriptPubKey
const script = tx.vout[0].scriptPubKey.hex;

// 3. Extract the 36-byte ref (positions 2-74 in hex string)
// Script format: d8<36-byte ref>7576a914...
//                ^^ skip this
const containerRef = script.substring(2, 74);  // 72 hex chars = 36 bytes

// 4. Use this ref in child NFT payloads
const payload = {
    p: [2],
    name: "Child NFT",
    in: [hexToUint8Array(containerRef)],  // Now correctly references parent
    by: [hexToUint8Array(authorRef)]      // Also extracted from output script
};
```

#### Real-World Example

**Container NFT:**
- Reveal txid: `4edad6696f9ba2c20b7f81bf135032bf1a781ebca40644c9fc1cd8aa817a3b63`
- Output script: `d813499062956d178e7e9d9950418a4a8e8aabbcd38cd38c41ebf38cd63741585800000000757...`
- **Correct ref**: `13499062956d178e7e9d9950418a4a8e8aabbcd38cd38c41ebf38cd63741585800000000` (from script)
- **Wrong ref**: `58584137d68cf3eb418cd38cd3bcab8a8e4a8a4150999d7e8e176d956290491300000000` (from txid)

Using the wrong ref means:
- Child NFTs won't appear in Glyphium under the container
- The hierarchy breaks
- NFTs exist but are orphaned

#### Implementation

```javascript
// Helper to extract singleton ref from a transaction
async function getSingletonRef(txid, vout = 0) {
    const tx = await rpc.call('getrawtransaction', [txid, true]);
    const script = tx.vout[vout].scriptPubKey.hex;

    // Verify it's a singleton script (starts with d8)
    if (!script.startsWith('d8')) {
        throw new Error('Not a singleton output');
    }

    // Extract 36-byte ref (72 hex chars starting at position 2)
    return script.substring(2, 74);
}

// Use when creating child NFTs
const containerRef = await getSingletonRef(containerTxid, 0);
const authorRef = await getSingletonRef(authorTxid, 0);

function hexToUint8Array(hex) {
    const bytes = new Uint8Array(hex.length / 2);
    for (let i = 0; i < hex.length; i += 2) {
        bytes[i / 2] = parseInt(hex.substr(i, 2), 16);
    }
    return bytes;
}

const payload = {
    p: [2],
    name: "My NFT",
    in: [hexToUint8Array(containerRef)],  // 72 hex chars from OUTPUT SCRIPT
    by: [hexToUint8Array(authorRef)]      // 72 hex chars from OUTPUT SCRIPT
};
```

#### Why This Matters

Container refs organize NFTs into collections:
- All NFTs with the same `in` ref appear under that container in explorers
- This creates browsable collections (e.g., "My App Verified Photos")
- Essential for platform branding and user experience

**Testing container refs:**
1. Mint a container NFT (no `in` field)
2. Extract its ref from the output script
3. Mint child NFTs using that ref in their `in` field
4. Check Glyphium - children should appear nested under the container
5. If children are orphaned, your ref extraction is wrong

### JavaScript CBOR Encoding

```javascript
function encodeGlyphData(data) {
    // Verify CBOR library is loaded
    if (typeof CBOR === 'undefined') {
        throw new Error('CBOR library not loaded! NFT would be unreadable.');
    }

    // Ensure protocol is set
    if (!data.p) {
        data.p = [2];
    }

    // Encode to CBOR
    const marker = new TextEncoder().encode('gly');
    const cborData = CBOR.encode(data);

    // CBOR.encode returns ArrayBuffer, convert to Uint8Array
    const payload = new Uint8Array(cborData);

    // Combine marker and payload
    const result = new Uint8Array(marker.length + payload.length);
    result.set(marker, 0);
    result.set(payload, marker.length);

    return result;
}
```

### Glyph Data Format

```
Glyph = "gly" marker (3 bytes) + CBOR payload
      = 676c79 + <CBOR bytes>
```

---

## Commit Transaction

### Purpose

The commit transaction creates an output with a custom script (nftCommitScript) that:
1. Validates the glyph data hash
2. Checks for "gly" marker
3. Verifies singleton ref exists in reveal outputs
4. Performs standard P2PKH signature validation

### nftCommitScript Structure

```
OP_HASH256 <32-byte-payload-hash> OP_EQUALVERIFY   // Verify CBOR hash
<3-byte "gly"> OP_EQUALVERIFY                       // Check "gly" marker
OP_INPUTINDEX OP_OUTPOINTTXHASH                    // Push input txid
OP_INPUTINDEX OP_OUTPOINTINDEX                     // Push input vout
OP_4 OP_NUM2BIN OP_CAT                             // Build 36-byte ref
OP_REFTYPE_OUTPUT OP_2 OP_NUMEQUALVERIFY           // Verify singleton in output
OP_DUP OP_HASH160 <20-byte-pubkeyhash> OP_EQUALVERIFY OP_CHECKSIG  // P2PKH
```

### Hex Breakdown

| Hex | Opcode | Description |
|-----|--------|-------------|
| `aa` | OP_HASH256 | Hash top stack item (double SHA256) |
| `20` | Push 32 bytes | Payload hash follows |
| `88` | OP_EQUALVERIFY | Check hash matches |
| `03` | Push 3 bytes | "gly" marker follows |
| `676c79` | "gly" | Literal bytes (ASCII: g=67, l=6c, y=79) |
| `c0` | OP_INPUTINDEX | Push current input index |
| `c8` | OP_OUTPOINTTXHASH | Push input's txid (BCH introspection) |
| `c9` | OP_OUTPOINTINDEX | Push input's vout (BCH introspection) |
| `54` | OP_4 | Push number 4 |
| `80` | OP_NUM2BIN | Convert vout to 4-byte LE |
| `7e` | OP_CAT | Concatenate txid+vout = 36-byte ref |
| `da` | OP_REFTYPE_OUTPUT | Check ref type in outputs |
| `52` | OP_2 | Push 2 (singleton type) |
| `9d` | OP_NUMEQUALVERIFY | Verify ref is singleton |
| `76a914...88ac` | P2PKH | Standard signature verification |

### PHP Implementation

```php
/**
 * Build the nftCommitScript for a Glyph NFT commit output.
 *
 * INVARIANTS (caller's responsibility — this function does NOT validate):
 *   - $pubkeyhash MUST be exactly 40 hex chars (20-byte P2PKH pubkeyhash)
 *   - $payloadHash MUST be exactly 64 hex chars (32-byte SHA256(CBOR payload))
 *
 * Passing shorter or attacker-influenced hex here produces a malformed
 * script whose length bytes disagree with the actual content — in the
 * worst case it can shift the pubkeyhash portion and direct the NFT to
 * an attacker-chosen address. Validate upstream before calling.
 */
private function buildNftCommitScript($pubkeyhash, $payloadHash) {
    assert(strlen($pubkeyhash) === 40 && ctype_xdigit($pubkeyhash));
    assert(strlen($payloadHash) === 64 && ctype_xdigit($payloadHash));

    $OP_HASH256 = 'aa';
    $OP_EQUALVERIFY = '88';
    $OP_DUP = '76';
    $OP_HASH160 = 'a9';
    $OP_CHECKSIG = 'ac';
    $OP_INPUTINDEX = 'c0';
    $OP_OUTPOINTTXHASH = 'c8';  // BCH introspection
    $OP_OUTPOINTINDEX = 'c9';   // BCH introspection
    $OP_4 = '54';
    $OP_NUM2BIN = '80';
    $OP_CAT = '7e';
    $OP_REFTYPE_OUTPUT = 'da';
    $OP_2 = '52';
    $OP_NUMEQUALVERIFY = '9d';

    $glyphMarker = '676c79';  // "gly" in hex

    $script = $OP_HASH256;
    $script .= '20' . $payloadHash;
    $script .= $OP_EQUALVERIFY;
    $script .= '03' . $glyphMarker;
    $script .= $OP_EQUALVERIFY;
    $script .= $OP_INPUTINDEX . $OP_OUTPOINTTXHASH;
    $script .= $OP_INPUTINDEX . $OP_OUTPOINTINDEX;
    $script .= $OP_4 . $OP_NUM2BIN . $OP_CAT;
    $script .= $OP_REFTYPE_OUTPUT . $OP_2 . $OP_NUMEQUALVERIFY;
    $script .= $OP_DUP . $OP_HASH160;
    $script .= '14' . $pubkeyhash;
    $script .= $OP_EQUALVERIFY . $OP_CHECKSIG;

    return $script;
}
```

### Payload Hash Calculation

```php
// $glyphHex = "676c79" + <CBOR hex>
$cborHex = substr($glyphHex, 6);  // Skip first 6 hex chars ("gly")
$cborBinary = hex2bin($cborHex);
$hash1 = hash('sha256', $cborBinary, true);   // First SHA256 (binary)
$hash2 = hash('sha256', $hash1, false);        // Second SHA256 (hex)
$payloadHash = $hash2;  // This goes in the nftCommitScript
```

---

## Reveal Transaction

### Purpose

The reveal transaction spends the commit output and creates the final NFT with:
1. Glyph data embedded in scriptSig
2. Singleton ref output that references the commit transaction
3. NFT owned by specified pubkeyhash

### ScriptSig Structure (CRITICAL)

The scriptSig must be in this **exact order**:

```
<signature> <pubkey> <"gly" marker> <CBOR payload>
```

### Singleton Output Script

```
d8 <36-byte-ref> 75 76 a9 14 <20-byte-pubkeyhash> 88 ac
```

| Hex | Opcode | Description |
|-----|--------|-------------|
| `d8` | OP_PUSHINPUTREFSINGLETON | Consumes next 36 bytes as ref |
| (36 bytes) | ref | txid (reversed) + vout (LE) |
| `75` | OP_DROP | Drop the ref from stack |
| `76a914...88ac` | P2PKH | Standard P2PKH script |

**CRITICAL:** `d8` directly consumes the next 36 bytes. Do NOT add a push opcode:

- **Correct:** `d8<ref>7576a914...`
- **WRONG:** `d824<ref>7576a914...`

### JavaScript Implementation

```javascript
function buildSingletonScript(commitTxid, commitVout, ownerPubkeyhash) {
    const txidReversed = Buffer.from(commitTxid, 'hex').reverse().toString('hex');
    const voutLE = Buffer.alloc(4);
    voutLE.writeUInt32LE(commitVout);
    const ref = txidReversed + voutLE.toString('hex');

    // d8 + ref + 75 (OP_DROP) + 76a914 + pubkeyhash + 88ac
    const script = 'd8' + ref + '7576a914' + ownerPubkeyhash + '88ac';

    return { script, ref };
}
```

---

## Signing Challenge

### The Problem

The wallet RPC (`signrawtransactionwithwallet`) cannot sign transactions spending nftCommitScript because:
1. It's a custom script, not recognized P2PKH
2. The wallet doesn't know how to construct the correct scriptSig with glyph data

**Error:** "Unable to sign input, invalid stack size"

### The Solution: External Signing with Node.js

Use radiantjs library to sign the reveal transaction.

### Node.js Signing Script

> **Private Key Security:**
> - **Never** pass WIF keys as command-line arguments (visible in `ps`, shell history)
> - **Never** hardcode WIF keys in source code or commit to git
> - Load keys from files or environment at runtime: `fs.readFileSync('/path/to/key.wif', 'utf8').trim()`
> - For development, use a wallet with limited funds only
> - For production, use dedicated signing services or hardware wallets

```javascript
#!/usr/bin/env node
const { Script, Transaction, PrivateKey, crypto } = require('@radiantblockchain/radiantjs');

async function signReveal(params) {
    const { commitTxid, commitVout, wif, glyphHex, outputSats,
            commitScript, commitAmount, destPubkeyhash } = params;

    const privateKey = PrivateKey.fromWIF(wif);
    const publicKey = privateKey.toPublicKey();
    const pubkeyhash = destPubkeyhash ||
        crypto.Hash.sha256ripemd160(publicKey.toBuffer()).toString('hex');

    // Create 36-byte ref
    const txidReversed = Buffer.from(commitTxid, 'hex').reverse().toString('hex');
    const voutLE = Buffer.alloc(4);
    voutLE.writeUInt32LE(commitVout);
    const ref = txidReversed + voutLE.toString('hex');

    // Build singleton output script
    const singletonScript = 'd8' + ref + '7576a914' + pubkeyhash + '88ac';

    // Build transaction
    const tx = new Transaction();

    tx.addInput(new Transaction.Input({
        prevTxId: commitTxid,
        outputIndex: commitVout,
        script: new Script(),
        output: new Transaction.Output({
            script: Script.fromHex(commitScript),
            satoshis: commitAmount, // radiantjs uses 'satoshis' — these are photons on Radiant
        }),
    }));

    tx.addOutput(new Transaction.Output({
        script: Script.fromHex(singletonScript),
        satoshis: outputSats // photons
    }));

    // Build glyph scriptSig
    const glyMarker = Buffer.from('gly', 'utf8');
    const cborHex = glyphHex.substring(6);
    const cborBuffer = Buffer.from(cborHex, 'hex');

    // CRITICAL: Sign with FULL commit script, NOT P2PKH subscript
    tx.setInputScript(0, (txObj, output) => {
        const sigType = crypto.Signature.SIGHASH_ALL | crypto.Signature.SIGHASH_FORKID;

        const sig = Transaction.Sighash.sign(
            txObj,
            privateKey,
            sigType,
            0,
            output.script,  // MUST be full nftCommitScript
            new crypto.BN(String(commitAmount))
        );

        const sigBuffer = Buffer.concat([sig.toBuffer(), Buffer.from([sigType])]);

        // Build scriptSig: <sig> <pubkey> <"gly"> <CBOR>
        return Script.empty()
            .add(sigBuffer)
            .add(publicKey.toBuffer())
            .add(glyMarker)
            .add(cborBuffer)
            .toString();
    });

    tx.seal();

    return {
        success: true,
        signedTx: tx.toString(),
        txid: tx.id,
        ref: ref
    };
}
```

### Hardware Wallet Support

Glyph **minting** cannot currently be done from a hardware wallet: the reveal transaction's scriptSig (`<glyph_push> <signature> <pubkey>`) is non-standard, and no mainstream hardware wallet supports signing arbitrary script structures. Minting requires software signing via Node.js as shown above.

Glyph **receiving and spending**, however, does work with the community-built Radiant Ledger Nano S Plus app. You can:

- Mint a Glyph with software signing and send the output to a Ledger-derived address (`m/44'/512'/0'/0/x`)
- Later spend that Glyph UTXO with a Ledger-signed transaction (the unlocking side is standard P2PKH)

See [`radiant-ledger-guide`](https://github.com/Zyrtnin-org/radiant-ledger-guide) for installation, wallet pairing, and the direct-APDU harness needed for spending Glyph UTXOs (Electron Radiant's GUI doesn't yet recognize Glyph-prefixed P2PKH as spendable — see section 6 of that guide).

First Ledger-signed Glyph UTXO spend confirmed on mainnet: [`22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3`](https://explorer.radiantblockchain.org/tx/22d4e0e07200437791b48651125a636b994593b215152241aef7113b24b71da3).

---

## Fee Calculations & Cost Analysis

> **V2 Fee Change (Block 415,000+):** Minimum relay fee increases 10x (1,000 to
> 10,000 photons/byte) after a 5,000-block grace period. Use `estimatefee` RPC
> instead of hardcoding fee rates.
>
> **Note:** `estimatefee` returns a dynamic network estimate that may be lower
> than the policy minimum. Always enforce the minimum relay fee as a floor:
> `max(estimatefee_result, minimum_relay_fee)`.

### Radiant Fee Structure

**Radiant uses photons/byte** (NOT photons/kB like Bitcoin):
- Minimum relay fee: **1000 photons/byte** (0.01 RXD/kB)
- Post-block 415,000: **10,000 photons/byte** (0.1 RXD/kB)
- Always add 50% safety margin

> **Terminology:** Photons are Radiant's smallest unit (like satoshis in Bitcoin).
> Code samples use `$feeSats` / `satoshis` variable names because radiantjs and
> most Radiant tooling inherited Bitcoin-style naming. The unit is the same — 1
> RXD = 100,000,000 photons.

### Fee Calculation Pattern

```php
function calculateFee($rpc, $txSize) {
    $blockHeight = $rpc->call('getblockcount');
    $minRate = ($blockHeight >= 415000) ? 10000 : 1000; // photons/byte

    // Use estimatefee as a signal, but never go below the minimum
    $estimate = $rpc->call('estimatefee', [6]); // RXD/kB
    $estimateRate = ($estimate > 0) ? $estimate * 100000000 / 1000 : $minRate; // → photons/byte

    $feeRate = max($minRate, $estimateRate);
    return $txSize * $feeRate * 1.5; // 50% safety margin, in photons
}
```

> ⚠️  **Don't hardcode `feeRate` in client JS.** The most common way to break
> users on a consensus-parameter change is to ship a JS bundle with a static
> fee rate (e.g. `feeRate = 1000`), let the CDN cache it for days, and never
> update when the network minimum moves. The CDN-cache trap in the
> troubleshooting section below compounds this: an `immutable` response can
> keep stale fees in browsers for a week after you think you've deployed the
> fix, producing "min relay fee not met" errors that *only* the users see.
>
> Compute `feeRate` server-side on every mint (as `calculateFee` above does),
> either by returning the current rate from your backend endpoint before the
> client builds a tx, or by letting the server build and sign the tx end-to-end
> (the pattern used in the Signing Challenge section). Either way, the network
> minimum should never live as a literal in long-lived cached JS.

### Common Fee Bug

```php
// WRONG - treats feeRate as photons/kB
$feeSats = ($txSize * $feeRate) / 1000;

// CORRECT - feeRate is already photons/byte
$feeSats = $txSize * $feeRate;
```

### Real-World Cost Examples

Observed on mainnet April 2026 minting a singleton-ref NFT with a small photo
payload through the commit/reveal path documented in this guide:

| Component | Size | Pre-V2 Cost | Post-V2 Cost (10x) |
|-----------|------|-------------|---------------------|
| Commit TX (observed) | ~276 bytes | ~0.003 RXD | ~0.028 RXD |
| Reveal TX (observed, ~640-byte glyph payload) | ~1,038 bytes | ~0.010 RXD | ~0.104 RXD |
| Thumbnail (150px, 65%) | ~6,000 bytes | ~0.06 RXD | ~0.6 RXD |
| Thumbnail (200px, 70%) | ~15,000 bytes | ~0.15 RXD | ~1.5 RXD |
| CBOR metadata | ~200-500 bytes | ~0.002-0.005 RXD | ~0.02-0.05 RXD |

A typical reveal is ~1 KB even with a small glyph — scriptSig carries signature +
pubkey + `"gly"` marker + CBOR body. Larger thumbnails push reveal size past 15 KB
quickly; plan for 0.5–2 RXD per NFT in real minting costs at post-V2 rates.

**Commit-amount sizing.** The commit transaction's output value must cover the
reveal transaction's fee **plus** the NFT output value (typically 10,000 photons
for the singleton dust). Observed: commit amount ≈ 10,400,000 photons to cover a
reveal at ~10,380,000 photons plus 10,000 photons NFT output.

**Total Cost Formula:**
```
Total = Commit Fee + Reveal Fee + (Thumbnail Size × fee_per_byte)
// Pre-V2:  fee_per_byte = 0.00001 RXD/byte (1000 photons/byte)
// Post-V2: fee_per_byte = 0.0001 RXD/byte  (10,000 photons/byte)
```

**Example with 150px thumbnail (pre-V2 / post-V2):**
- Commit: 0.003 / 0.03 RXD
- Reveal base: 0.0025 / 0.025 RXD
- Thumbnail (6KB): 0.06 / 0.6 RXD
- Metadata (300 bytes): 0.003 / 0.03 RXD
- **Total: ~0.07 / ~0.69 RXD**

**Example with 200px thumbnail (pre-V2 / post-V2):**
- Commit: 0.003 / 0.03 RXD
- Reveal base: 0.0025 / 0.025 RXD
- Thumbnail (15KB): 0.15 / 1.5 RXD
- Metadata (300 bytes): 0.003 / 0.03 RXD
- **Total: ~0.16 / ~1.59 RXD**

---

## IPFS Integration

### Purpose

Use IPFS for full-resolution images while keeping on-chain thumbnails small:
- **On-chain (`main` field):** Small thumbnail for wallet display
- **IPFS (`loc` field):** Full-resolution original

### Server-Side IPFS Upload (Pinata)

> ⚠️  **Secrets hygiene.** A Pinata JWT grants full account control (pin,
> unpin, billing). Treat it like a password. Keep real secrets in a local
> `.env` file, **add `.env` to `.gitignore`**, and commit a `.env.example`
> with placeholders so contributors know which keys to set. This applies to
> the Pinata JWT, your Radiant RPC password, any AI provider keys, and
> anything else `getenv()` reads. Public-repo secret leaks are the
> single most common way people burn themselves on projects built from
> guides like this one.

```php
function uploadFileToPinata($fileData, $filename, $mimeType = 'image/jpeg') {
    $jwt = getenv('PINATA_JWT');

    if (empty($jwt)) {
        return ['success' => false, 'error' => 'Pinata JWT not configured'];
    }

    // Handle base64 data URL
    if (strpos($fileData, 'data:') === 0) {
        $parts = explode(',', $fileData, 2);
        if (count($parts) === 2) {
            $fileData = base64_decode($parts[1]);
        }
    }

    $tmpFile = tempnam(sys_get_temp_dir(), 'pinata_');
    file_put_contents($tmpFile, $fileData);

    $cfile = new CURLFile($tmpFile, $mimeType, $filename);

    $ch = curl_init('https://api.pinata.cloud/pinning/pinFileToIPFS');
    curl_setopt_array($ch, [
        CURLOPT_RETURNTRANSFER => true,
        CURLOPT_POST => true,
        CURLOPT_HTTPHEADER => ['Authorization: Bearer ' . $jwt],
        CURLOPT_POSTFIELDS => ['file' => $cfile],
        CURLOPT_TIMEOUT => 60
    ]);

    $response = curl_exec($ch);
    curl_close($ch);
    unlink($tmpFile);

    $result = json_decode($response, true);
    $gateway = getenv('PINATA_GATEWAY') ?: 'gateway.pinata.cloud';

    return [
        'success' => true,
        'cid' => $result['IpfsHash'],
        'url' => 'ipfs://' . $result['IpfsHash'],
        'gatewayUrl' => "https://{$gateway}/ipfs/" . $result['IpfsHash']
    ];
}
```

### SSL Certificate Issues

**Problem:** "SSL certificate problem: unable to get local issuer certificate"

**Development-only workaround** (never ship this):
```bash
# In .env — DEVELOPMENT ONLY. Disables TLS verification for the Pinata curl handle.
IPFS_SKIP_SSL_VERIFY=true
```

> ⚠️  **Never set this in production.** Disabling TLS verification lets an
> active network attacker (hotel/cafe wifi, corporate MITM proxy, compromised
> upstream) substitute the IPFS CID in Pinata's response. You then mint an NFT
> pointing at the attacker's content instead of yours — permanent and visible
> on chain. Fix the CA bundle instead.

**Production:** Configure proper CA bundle path in `curl.cainfo` php.ini
setting, or install `ca-certificates` in your container image so system roots
are present.

---

## Common Errors & Solutions

### Environment & Infrastructure

These are the errors you hit *before* any protocol-level problem, and they
account for the majority of time lost setting up a minting pipeline for the
first time.

#### `sh: node: not found` (from PHP)

**Cause:** The web container doesn't have Node.js installed, but your PHP code
is trying to shell out to `node scripts/sign_reveal.js`.

**Fix:** Install Node in the Dockerfile. See Infrastructure Setup → Calling the
Signer from PHP.

#### `Method not found` (code -32601) for `listunspent` / `dumpprivkey` / `signrawtransactionwithwallet`

**Cause:** Your Radiant daemon was built without wallet support. v2.2.0 prebuilt
tarballs ship node-only; v2.1.2 tarballs are mislabeled ARM64.

**Fix:** Upgrade to a wallet-enabled build (v2.3.0+). Verify with
`docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant listwallets`.
Remember `--no-cache` on the rebuild, and **back up `wallet.dat` first**.

#### `Cannot find module '@radiantblockchain/radiantjs'` (from Node)

**Cause:** radiantjs isn't installed, or is installed at the wrong path (npm put
it at `node_modules/radiantjs/` because that's the dep key you used).

**Fix:** Either install with `npm install github:chainbow/radiantjs` and accept
npm's placement, or add a symlink:

```bash
mkdir -p node_modules/@radiantblockchain
ln -sfn ../radiantjs node_modules/@radiantblockchain/radiantjs
```

If your deps live at `/opt/signing-deps/node_modules`, set
`NODE_PATH=/opt/signing-deps/node_modules` so the lookup walks there.

#### "Node.js signing failed" with a valid `{"success":true,"signedTx":…}` in the message

**Cause:** PHP's `proc_close()` returns -1 because `proc_get_status()` already
reaped the child. The signed transaction in stdout is *correct*; the exit code
is bogus.

**Fix:** Capture the exit code from `proc_get_status()['exitcode']` at the
moment `running=false`, and parse stdout JSON before falling back to the exit
code. See Infrastructure Setup → Calling the Signer from PHP for a reference
implementation.

#### Web container can't reach the Radiant node ("Connection refused" / DNS fails)

**Cause:** Containers are on separate Docker networks, or `rpcbind=127.0.0.1`
in `radiant.conf` restricts the daemon to in-container loopback.

**Fix:** Set `rpcbind=0.0.0.0` + `rpcallowip=172.16.0.0/12` in `radiant.conf`,
bind host ports to `127.0.0.1:`, and attach both containers to the same Docker
network. See Infrastructure Setup → Networking the Node.

#### Schema drift: three variants of "missing column"

The database part of minting has three distinct failure modes that look
superficially similar but need different diagnostics. If you hit any of
them, check **all three** locations where a new column lives: the schema
definition, the API whitelist that filters incoming writes, and the
actual INSERT/UPDATE column list.

| Symptom | Cause |
|---|---|
| `ERROR: must be owner of table` on `ALTER TABLE` | Running migration as the app user instead of `postgres` superuser. Use `-U postgres`. |
| Every write returns 500 with `column "X" does not exist` | New code deployed before migration ran. Always migrate the DB *first*. |
| Write "succeeds" in the UI but state resets on reload | Column exists in schema but API whitelist strips it, or upsert omits it. Save path silently drops the field; reload repopulates from DB with the missing value. |
| Write succeeds but silently no-ops on one field | ORM silently dropping unknown columns. Check your DB client's strict/loose mode and the UPDATE column list. |

The "looks saved in JS but resets on reload" version is the sneakiest — the
failing field looks like a frontend bug because the UI appears to accept the
change. It's almost always a desync between schema / API / upsert: the three
places a new column has to be threaded through.

#### Stale JS served to users after deploy

**Cause:** CDN (usually Cloudflare) is holding the unversioned JS/CSS file
because your nginx sent `Cache-Control: public, immutable; expires 7d`.

**Fix:** Change the nginx rule for JS/CSS to a short TTL with revalidation, and
purge the CDN cache to evict the already-poisoned entries:

```nginx
# Short-cache unversioned JS/CSS — lets deploys reach users within minutes.
location ~* \.(css|js)$ {
    expires 5m;
    add_header Cache-Control "public, max-age=300, must-revalidate";
}
# Keep `immutable` for content-addressed assets only.
location ~* \.(svg|png|jpg|jpeg|gif|ico|woff|woff2|ttf|eot)$ {
    expires 7d;
    add_header Cache-Control "public, immutable";
}
```

#### CORS-blocked image fetch from an IPFS/R2 gateway

**Cause:** CDN cache was populated by a non-browser fetch with no `Origin`
header, so the cached response has no `Access-Control-Allow-Origin`. All
subsequent browser `fetch()` calls hit the same poisoned entry.

**Fix:** Add a per-request cachebuster on the authoritative fetch
(`?_cb=<timestamp>`), then purge the CDN cache once to evict the poisoned
response. Your origin (R2, Pinata, etc.) should have CORS configured correctly;
the problem is almost always the intermediate CDN layer.

### Protocol Errors

#### "NFT shows as blank card in wallet"

**Cause:** Missing `main` field with on-chain image data.

**Fix:** Add thumbnail to payload:
```javascript
payload.main = {
    t: 'image/webp',        // Must match thumbnail format
    b: thumbnailUint8Array  // Must be Uint8Array, not base64
};
```

#### "NFT shows as 'Unknown NFT' with no metadata"

**Cause:** NFT was encoded with JSON instead of CBOR.

**Symptoms:**
- NFT appears but shows as "Unknown NFT"
- No attributes visible
- Browser console shows: "CBOR library not loaded, using JSON fallback"

**Fix:**
1. Download CBOR library: `curl -o js/cbor.min.js "https://raw.githubusercontent.com/paroga/cbor-js/master/cbor.js"`
2. Load BEFORE blockchain scripts in HTML
3. Verify: `console.log(typeof CBOR)` should output "object"
4. Mint new NFT (old one cannot be fixed)

#### "Extra items left on stack after execution"

**Cause:** Using P2PKH commit instead of nftCommitScript.

**Fix:** Pass `glyphHex` to commit transaction:
```javascript
const glyphData = encodeGlyphData(payload);
const glyphHex = Array.from(glyphData).map(b => b.toString(16).padStart(2, '0')).join('');
const commitResult = await createCommitTransaction(feeRate, glyphHex);
```

#### "Unable to sign input, invalid stack size"

**Cause:** Wallet RPC cannot sign custom nftCommitScript.

**Fix:** Use external signing with Node.js and radiantjs (see Signing Challenge section).

#### "min relay fee not met (code 66)"

**Cause:** Transaction fee is below Radiant's enforced minimum relay fee.

**Fix (post-V2 mainnet, block ≥ 415,000):**
```php
// Radiant Core 2 enforces a minimum relay fee of 0.1 RXD/kB = 10,000 photons/byte.
// This is fully in effect from block 415,000 onward; between 410,000 and 415,000
// miners ran in a grace window that capped policy at the legacy 0.01 RXD/kB floor.
$minRate = 10000;                       // photons/byte (post-V2 minimum)
$feeSats = $txSize * $minRate * 1.5;    // photons, with safety margin
```

If you're building against a chain before block 415,000 (e.g. regtest or a custom network that hasn't grace-graduated), you can use `getblockcount` to pick the right floor:

```php
$h = $rpc->call('getblockcount');
$minRate = ($h < 415000) ? 1000 : 10000; // legacy floor before grace ends
```

Mainnet has been past 415,000 since early 2026; production code should default to 10,000.

#### "reference-operations" error

**Cause:** Incorrect ref format or extra push opcode.

**Fix:**
- Verify ref = reversed(txid) + LE(vout), exactly 36 bytes
- Use `d8<ref>75...` NOT `d824<ref>75...`

#### "Signature must be zero"

**Cause:** Signing with P2PKH subscript instead of full commit script.

**Fix:** In signing script, use `output.script` (full nftCommitScript).

#### "SSL certificate problem" (IPFS upload)

See the full discussion in "IPFS Integration → SSL Certificate Issues" above —
including the warning about why you must not ship with `IPFS_SKIP_SSL_VERIFY=true`
enabled.

#### "Child NFTs don't appear in container" (Glyphium/Explorers)

**Cause:** Using txid-derived ref instead of extracting from output script.

**Symptoms:**
- Child NFTs exist and are visible in explorers
- BUT they don't appear nested under the container
- Container shows 0 children even though child NFTs have `in` field set

**Fix:**
```javascript
// ❌ WRONG - Don't calculate from txid
const wrongRef = reverseHex(containerTxid) + '00000000';

// ✅ CORRECT - Extract from output script
const tx = await rpc.call('getrawtransaction', [containerTxid, true]);
const script = tx.vout[0].scriptPubKey.hex;
const correctRef = script.substring(2, 74);  // Skip 'd8', take next 72 chars
```

**Why it happens:**
- The singleton ref is based on the COMMIT transaction, not the reveal
- Reveal txid ≠ the ref value in the output script
- You must query the blockchain and extract the ref from the actual output

**Verification:**
1. Check your container ref matches the value in the output script (starts at position 2)
2. Child NFTs should use this exact ref in their `in` field
3. In Glyphium, click the container - children should be listed

---

## Complete Implementation Example

### Full Minting Flow with Thumbnail

```javascript
class GlyphNFTMinter {
    async mintNFT(imageDataUrl, metadata, ownerAddress) {
        // Step 1: Create thumbnail for on-chain storage (225px @ 90% WebP)
        const thumbnail = await this.createThumbnail(imageDataUrl, 225, 0.90);
        console.log(`Thumbnail: ${thumbnail.length} bytes`);

        // Step 2: Upload full-res to IPFS (optional)
        const ipfsResult = await this.uploadToIPFS(imageDataUrl);
        console.log(`IPFS: ${ipfsResult.url}`);

        // Step 3: Build payload with main field
        const payload = {
            p: [2],
            name: metadata.name,
            type: metadata.type || 'photo',
            main: {
                t: 'image/webp',  // WebP for best quality/size ratio
                b: thumbnail
            },
            loc: ipfsResult.url,
            attrs: metadata.attrs || {}
        };

        // Step 4: Encode to CBOR
        const glyphData = this.encodeGlyphData(payload);
        const glyphHex = Array.from(glyphData)
            .map(b => b.toString(16).padStart(2, '0'))
            .join('');

        // Step 5: Create commit transaction
        const commitResult = await this.createCommitTransaction(glyphHex);
        console.log('Commit TX:', commitResult.txid);

        // Step 6: Wait for confirmation
        await this.waitForConfirmation(commitResult.txid);

        // Step 7: Create and sign reveal transaction
        const revealResult = await this.createRevealTransaction(
            commitResult, glyphHex, ownerAddress
        );
        console.log('Reveal TX:', revealResult.txid);

        return {
            commitTxid: commitResult.txid,
            revealTxid: revealResult.txid,
            glyphId: `${revealResult.txid}:0`,
            ref: revealResult.ref,
            thumbnailSize: thumbnail.length,
            ipfsUrl: ipfsResult.url
        };
    }

    async createThumbnail(dataUrl, maxSize = 225, quality = 0.90) {
        return new Promise((resolve, reject) => {
            const img = new Image();
            img.onload = () => {
                let width = img.width;
                let height = img.height;

                if (width > height) {
                    if (width > maxSize) {
                        height = Math.round((height * maxSize) / width);
                        width = maxSize;
                    }
                } else {
                    if (height > maxSize) {
                        width = Math.round((width * maxSize) / height);
                        height = maxSize;
                    }
                }

                const canvas = document.createElement('canvas');
                canvas.width = width;
                canvas.height = height;
                const ctx = canvas.getContext('2d');
                ctx.imageSmoothingEnabled = true;
                ctx.imageSmoothingQuality = 'high';
                ctx.drawImage(img, 0, 0, width, height);

                // Use WebP for best quality/size ratio
                canvas.toBlob((blob) => {
                    const reader = new FileReader();
                    reader.onload = () => resolve(new Uint8Array(reader.result));
                    reader.onerror = reject;
                    reader.readAsArrayBuffer(blob);
                }, 'image/webp', quality);
            };
            img.onerror = reject;
            img.src = dataUrl;
        });
    }

    encodeGlyphData(data) {
        if (typeof CBOR === 'undefined') {
            throw new Error('CBOR library not loaded!');
        }
        if (!data.p) data.p = [2];

        const marker = new TextEncoder().encode('gly');
        const cborData = CBOR.encode(data);
        const payload = new Uint8Array(cborData);

        const result = new Uint8Array(marker.length + payload.length);
        result.set(marker, 0);
        result.set(payload, marker.length);
        return result;
    }
}
```

---

## Testing & Verification

### Verify CBOR Encoding

```javascript
// Before minting, verify CBOR is working
const testPayload = { p: [2], name: "Test" };
const encoded = CBOR.encode(testPayload);
const decoded = CBOR.decode(encoded);
console.log('CBOR test:', decoded.name === "Test" ? 'PASS' : 'FAIL');
```

### Verify Thumbnail

```javascript
// Check thumbnail size before minting
const thumbnail = await createThumbnail(imageDataUrl, 225, 0.90);
console.log(`Thumbnail size: ${thumbnail.length} bytes`);
if (thumbnail.length > 30000) {
    console.warn('Thumbnail large - consider reducing quality or dimensions');
}
```

### Verify on Glyph Explorer

Visit: `https://glyph-explorer.rxd-radiant.com/tx/<reveal_txid>`

You should see:
- NFT image displayed (from `main` field)
- NFT metadata
- Container/author refs (if used)
- Attributes

### Verify in Glyphium Wallet

Import your wallet and check:
- NFT appears in collection
- Thumbnail is visible and clear
- Attributes are readable

### Verified Working Transactions (January 2026)

**With on-chain thumbnail:**
- Reveal: `27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e`
- Thumbnail: 150x200, ~14KB, displays correctly in Glyphium

---

## Security Best Practices

### Input Validation for Blockchain Operations

When building applications that interact with the Radiant blockchain, **always validate inputs** before passing them to RPC calls or transaction signing scripts. Malformed data can cause errors, vulnerabilities, or unexpected behavior.

#### Transaction ID Validation

```javascript
// Validate txid format (64 hex characters)
function isValidTxid(txid) {
    return /^[a-f0-9]{64}$/i.test(txid);
}

// Example usage
const commitTxid = userInput.trim();
if (!isValidTxid(commitTxid)) {
    throw new Error('Invalid transaction ID format');
}
```

```php
// PHP version
if (!preg_match('/^[a-f0-9]{64}$/i', $commitTxid)) {
    throw new Exception('Invalid commit transaction ID format');
}
```

#### Glyph Hex Validation

Glyph payloads must start with the "gly" marker (`676c79` in hex). Validate both format and size:

```javascript
// Validate glyph hex format
function isValidGlyphHex(glyphHex) {
    // Must start with "gly" marker (676c79)
    if (!glyphHex.toLowerCase().startsWith('676c79')) {
        return false;
    }

    // Must be valid hex
    if (!/^[a-f0-9]+$/i.test(glyphHex)) {
        return false;
    }

    // Reasonable size limit (e.g., 100KB = 200,000 hex chars)
    if (glyphHex.length > 200000) {
        return false;
    }

    return true;
}
```

```php
// PHP version with detailed validation
function validateGlyphHex($glyphHex) {
    // Validate format (must start with "gly" marker)
    if (!preg_match('/^676c79[a-f0-9]*$/i', $glyphHex)) {
        throw new Exception('Invalid glyph hex format (must start with "gly" marker)');
    }

    // Validate length (prevent excessive data)
    if (strlen($glyphHex) > 200000) { // 100KB hex = 200,000 chars
        throw new Exception('Glyph hex data too large (max 100KB)');
    }

    return true;
}
```

#### Output Index Validation

Validate that output indices (vout) are within reasonable bounds:

```javascript
function isValidVout(vout) {
    const index = parseInt(vout, 10);
    return !isNaN(index) && index >= 0 && index < 1000;
}
```

```php
// Validate commitVout is a non-negative integer in a sane range.
// NOTE: use is_int + explicit cast, NOT just "< 0 || > 1000" — PHP's
// type coercion means the string "abc" compares false to both bounds
// and slips through. Do the type check first.
if (!is_int($commitVout)) {
    // Most $_POST/$_GET/JSON values arrive as strings; cast deliberately.
    if (!is_numeric($commitVout) || (int)$commitVout != $commitVout) {
        throw new Exception('Invalid commit output index (not an integer)');
    }
    $commitVout = (int)$commitVout;
}
if ($commitVout < 0 || $commitVout > 1000) {
    throw new Exception('Invalid commit output index');
}
```

#### Address Validation

Always validate Radiant addresses using the RPC:

```javascript
async function validateAddress(address) {
    const validation = await rpc.call('validateaddress', [address]);
    if (!validation.isvalid) {
        throw new Error('Invalid Radiant address');
    }
    return validation;
}
```

```php
// PHP version
if ($destAddress) {
    $validation = $this->rpc->call('validateaddress', [$destAddress]);
    if (!$validation['isvalid']) {
        throw new Exception('Invalid destination address');
    }
}
```

#### Complete Example: Secure Reveal Transaction Creation

```php
function createRevealTransaction($commitTxid, $commitVout, $glyphHex, $destAddress = null) {
    // 1. Validate commit txid format
    if (!preg_match('/^[a-f0-9]{64}$/i', $commitTxid)) {
        throw new Exception('Invalid commit transaction ID format');
    }

    // 2. Validate commit vout is a non-negative integer.
    //    Cast first — PHP type coercion lets strings like "abc" pass bounds checks.
    if (!is_numeric($commitVout) || (int)$commitVout != $commitVout) {
        throw new Exception('Invalid commit output index (not an integer)');
    }
    $commitVout = (int)$commitVout;
    if ($commitVout < 0 || $commitVout > 1000) {
        throw new Exception('Invalid commit output index');
    }

    // 3. Validate glyph hex format and content
    if (!preg_match('/^676c79[a-f0-9]*$/i', $glyphHex)) {
        throw new Exception('Invalid glyph hex format (must start with "gly" marker)');
    }

    // 4. Validate glyph hex length (prevent excessive data)
    if (strlen($glyphHex) > 200000) { // 100KB hex = 200,000 chars
        throw new Exception('Glyph hex data too large (max 100KB)');
    }

    // 5. Validate destination address if provided
    if ($destAddress) {
        $validation = $this->rpc->call('validateaddress', [$destAddress]);
        if (!$validation['isvalid']) {
            throw new Exception('Invalid destination address');
        }
    }

    // All inputs validated - proceed with transaction creation
    return $this->executeRevealTransaction($commitTxid, $commitVout, $glyphHex, $destAddress);
}
```

#### Why This Matters

- **Prevents Script Errors**: Malformed hex or txids can cause Node.js signing scripts to crash
- **Avoids Lost Funds**: Invalid addresses or indices can result in unspendable outputs
- **Security**: Validates data before passing to shell commands or RPC calls
- **Better UX**: Provides clear error messages before attempting blockchain operations

#### Validation Checklist

Before any blockchain operation:

- [ ] Transaction IDs are 64 hex characters
- [ ] Output indices are non-negative integers < 1000
- [ ] Glyph hex starts with `676c79` ("gly" marker)
- [ ] Glyph hex is valid hexadecimal only
- [ ] Glyph hex size is reasonable (< 100KB recommended)
- [ ] Radiant addresses validated via RPC `validateaddress`
- [ ] All user inputs sanitized before passing to shell commands

---

## What's New in V2

> V2 is live on mainnet. This section has been updated against a running Radiant
> Core v2.3.0 node (tip beyond 420,000) and verified against the consensus
> parameters in `src/chainparams.cpp`, `src/policy/policy.h`, and
> `src/script/script.h`.

### V2 Activation (Block 410,000)

At block 410,000, three things activated simultaneously:

- **ASERT difficulty adjustment** — the half-life dropped from 2 days to 12 hours,
  so block-time variance is tighter. Expect faster recovery from hashrate spikes
  and drops.
- **Six new opcodes** — `OP_BLAKE3` (`0xee`), `OP_K12` (`0xef`), `OP_LSHIFT` (`0x98`),
  `OP_RSHIFT` (`0x99`), `OP_2MUL` (`0x8d`), `OP_2DIV` (`0x8e`). These enable dMint
  mining validation and advanced script contracts. See Appendix opcode table.
- **New fee framework** — defined at 410,000 but enforced on a delay (see below).

### Fee Increase (Block 415,000, after grace)

The minimum relay fee rose 10x from 0.01 RXD/kB (legacy) to **0.1 RXD/kB**
(10,000 photons/byte), with a maximum block min-fee cap of **0.5 RXD/kB**. Between
blocks 410,000 and 415,000, miners ran under a 5,000-block (~1 week) grace window
that kept the effective floor at the legacy rate. From block 415,000 onward, the
0.1 RXD/kB floor is fully enforced — every transaction you build today must meet
it or receive `{"code":-26,"message":"min relay fee not met (code 66)"}`.

See Fee Calculations & Cost Analysis for the post-V2 cost tables.

### New Protocols: dMint and WAVE

- **dMint** — Mineable token distribution via PoW. Protocol combination `[1, 4]`. Three active mining algorithms: SHA256D (0), BLAKE3 (1), K12 (2).
- **WAVE** — On-chain naming system. Protocol 11. Provides human-readable addresses and DNS-like records.

See the [Radiant AI Knowledge Base](https://github.com/Radiant-Core/radiant-mcp-server/blob/master/docs/RADIANT_AI_KNOWLEDGE_BASE.md) for implementation details.

---

## Appendix: Quick Reference

### Opcodes

| Hex | Name | Notes |
|-----|------|-------|
| `aa` | OP_HASH256 | Double SHA256 |
| `88` | OP_EQUALVERIFY | Check equal and remove |
| `c0` | OP_INPUTINDEX | BCH introspection |
| `c8` | OP_OUTPOINTTXHASH | BCH introspection - input txid |
| `c9` | OP_OUTPOINTINDEX | BCH introspection - input vout |
| `da` | OP_REFTYPE_OUTPUT | Check ref type in outputs |
| `d8` | OP_PUSHINPUTREFSINGLETON | Create singleton ref |
| `75` | OP_DROP | Remove top stack item |
| `76` | OP_DUP | Duplicate top stack item |
| `a9` | OP_HASH160 | RIPEMD160(SHA256(x)) |
| `ac` | OP_CHECKSIG | Verify signature |

#### V2 Opcodes (available after block 410,000)

| Hex | Name | Notes |
|-----|------|-------|
| `ee` | OP_BLAKE3 | BLAKE3 hash (max 1024-byte input) |
| `ef` | OP_K12 | KangarooTwelve hash |
| `98` | OP_LSHIFT | Bitwise left shift |
| `99` | OP_RSHIFT | Bitwise right shift |
| `8d` | OP_2MUL | Multiply by 2 |
| `8e` | OP_2DIV | Divide by 2 |

### Key Hex Values

| Purpose | Hex |
|---------|-----|
| "gly" marker | `676c79` |
| Push 3 bytes | `03` |
| Push 20 bytes | `14` |
| Push 32 bytes | `20` |
| OP_PUSHDATA1 | `4c` |

### Checklist Before Minting

- [ ] CBOR library loaded (`typeof CBOR === 'object'`)
- [ ] Thumbnail created (Uint8Array, < 30KB recommended)
- [ ] `main` field added to payload with `t` and `b` properties
- [ ] Protocol set (`p: [2]`)
- [ ] Name set
- [ ] Sufficient RXD balance for fees

### Cost Quick Reference

See [Thumbnail Size vs Cost Tradeoffs](#thumbnail-size-vs-cost-tradeoffs) and [Fee Calculations & Cost Analysis](#fee-calculations--cost-analysis) for detailed cost tables with pre-V2 and post-V2 pricing.

---

**Last Updated:** 2026-04-16 — updated against Radiant Core v2.3.0 after a real end-to-end mainnet mint. Added Infrastructure Setup, environment troubleshooting, security hygiene, and glossary sections. Verified fee-activation heights against consensus params in `src/chainparams.cpp` and `src/policy/policy.h`.
**Based on Verified Mainnet Transactions:**
- With thumbnail: `27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e`

**Key Highlights:**
1. On-chain images (`main` field) required for wallet display
2. CBOR encoding mandatory (JSON = "Unknown NFT")
3. Optimal thumbnail: 225px WebP @ 90% quality (~22KB, ~0.22 RXD pre-V2)
4. V2-ready fee calculations with pre/post cost tables
5. All 11 Glyph protocol types and 6 new V2 opcodes documented
6. MCP server integration for AI-assisted development (see [BUILDING_WITH_CLAUDE.md](BUILDING_WITH_CLAUDE.md))

**License:** MIT
**Radiant Blockchain:** https://radiantblockchain.org
