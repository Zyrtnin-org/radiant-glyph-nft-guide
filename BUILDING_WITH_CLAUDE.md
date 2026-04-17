# Building NFT Applications on Radiant with Claude AI

**A Guide for Radiant Blockchain Developers**

This document shares lessons learned from building a production NFT application on Radiant blockchain using Claude AI as a development partner. It includes practical examples, common pitfalls, and strategies for effective AI-assisted blockchain development.

---

## Table of Contents

1. [Introduction](#introduction)
2. [Critical Discoveries](#critical-discoveries)
3. [Why Claude for Blockchain Development](#why-claude-for-blockchain-development)
   - [With the MCP Server](#with-the-mcp-server)
   - [Setting Up the MCP Server](#setting-up-the-mcp-server)
4. [Project Setup](#project-setup)
5. [Working with Claude](#working-with-claude)
6. [Glyph NFT Implementation](#glyph-nft-implementation)
7. [Critical Gotchas](#critical-gotchas)
8. [Debugging Workflow](#debugging-workflow)
9. [Testing Strategy](#testing-strategy)
10. [Code Patterns That Work](#code-patterns-that-work)
11. [Resources](#resources)

---

## Introduction

This guide documents the experience of building a complete Glyph NFT minting system on Radiant blockchain with Claude AI assistance. The result: working mainnet NFTs using the commit/reveal pattern with singleton ref technology.

**What We Built:**
- Full commit/reveal NFT minting pipeline
- CBOR-encoded Glyph protocol metadata
- **On-chain thumbnails for wallet display** (critical!)
- Node.js signing integration for custom scripts
- Player wallet minting (Option B)
- Container/author ref support for NFT organization
- IPFS integration for full-resolution photo storage

**Verified Mainnet Transactions:**
- With on-chain thumbnail: `27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e`

---

## Critical Discoveries

> **Read this section first.** These are hard-won lessons that will save you hours of debugging.

### Discovery 1: On-Chain Images Are Required

**The Problem:**
We created NFTs with IPFS links in the `loc` field. They appeared in Glyph Explorer and Glyphium wallet... as blank cards. No image displayed.

**The Investigation:**
After much debugging, we discovered that Glyph wallets look for the `main` field to display NFT images. IPFS links are for full-resolution backup only.

**The Solution:**
```javascript
payload.main = {
    t: 'image/webp',           // MIME type (must match thumbnail format)
    b: thumbnailUint8Array     // Image data as Uint8Array (NOT base64!)
};
```

**Cost Impact:**
- 225px, 90% WebP thumbnail: ~22KB, ~0.22 RXD extra cost (pre-V2)
- Worth it for visible NFTs!

### Discovery 2: CBOR Encoding is Mandatory

**The Problem:**
NFTs showed as "Unknown NFT" with no metadata visible in Glyph wallets.

**The Root Cause:**
Our code had a fallback to JSON when CBOR library wasn't loaded:
```javascript
// This silent fallback broke everything!
if (typeof CBOR === 'undefined') {
    console.warn('Using JSON fallback');
    return JSON.stringify(data);  // WRONG!
}
```

**The Solution:**
```javascript
if (typeof CBOR === 'undefined') {
    throw new Error('CBOR library not loaded! NFT would be unreadable.');
}
```

**How We Found It:**
Claude suggested checking the browser console. We saw "CBOR library not loaded, using JSON fallback" - the library wasn't loading due to script order in HTML.

### Discovery 3: Thumbnail Quality vs Size Tradeoff

**Initial Attempt:** 100px, 50% quality
- Result: Blurry, pixelated thumbnails
- Size: ~3KB

**Initial Working Settings:** 150px, 65% JPEG
- Result: Clear, recognizable images
- Size: ~6KB, Cost: ~0.06 RXD extra

**Updated Recommendation:** 225px, 90% WebP
- Result: Sharp, detailed images with best quality-to-size ratio
- Size: ~22KB, Cost: ~0.22 RXD extra (pre-V2)

**The Lesson:**
Don't over-optimize thumbnail size. WebP gives better quality at comparable sizes to JPEG.

### Discovery 4: Container/Author Refs Must Be Extracted, Not Calculated

**The Problem:**
Child NFTs existed on-chain and were visible in Glyphium, but they didn't appear nested under the container. Container showed 0 children despite child NFTs having the `in` field populated.

**The Root Cause:**
We calculated container/author refs from the reveal transaction ID:
```javascript
// ❌ WRONG - This creates orphaned NFTs!
const containerRef = reverseHex(containerTxid) + '00000000';

// Child NFT with wrong ref:
payload.in = [hexToUint8Array(containerRef)];  // Orphaned!
```

**Why It Failed:**
- The singleton ref in the output script is based on the COMMIT transaction
- The reveal txid is NOT the same as the ref value
- Computing from reveal txid gives a completely different 36-byte value
- Child NFTs reference a non-existent parent

**The Solution:**
Extract the ref directly from the NFT's output script:
```javascript
// ✅ CORRECT - Extract from blockchain
const tx = await rpc.call('getrawtransaction', [containerTxid, true]);
const script = tx.vout[0].scriptPubKey.hex;
// Script format: d8<36-byte ref>7576a914...
//                ^^ skip this
const containerRef = script.substring(2, 74);  // Extract actual ref

// Now child NFTs will correctly nest:
payload.in = [hexToUint8Array(containerRef)];  // Correct parent!
```

**Real-World Example:**
- Container reveal txid: `4edad6696f9ba2c20b7f81bf135032bf1a781ebca40644c9fc1cd8aa817a3b63`
- Wrong ref (from txid): `58584137d68cf3eb418cd38cd3bcab8a8e4a8a4150999d7e8e176d956290491300000000`
- Correct ref (from script): `13499062956d178e7e9d9950418a4a8e8aabbcd38cd38c41ebf38cd63741585800000000`

**The Impact:**
We minted 20+ test NFTs with wrong refs. They exist on-chain and display in wallets, but they're orphaned - not connected to the container. These NFTs had to be melted and re-minted.

**How We Found It:**
After checking Glyphium and seeing the container empty, we compared:
1. The ref we calculated from the txid
2. The ref in the actual output script
3. The ref in child NFT payloads

They didn't match! Claude helped decode the output script and extract the correct values.

**The Lesson:**
Never assume you can compute blockchain references. Always query the actual on-chain data and extract what you need.

### Discovery 5: SSL Certificate Issues with IPFS

**The Problem:**
IPFS uploads failing with: "SSL certificate problem: unable to get local issuer certificate"

**The Quick Fix (Development):**
```bash
# In .env
IPFS_SKIP_SSL_VERIFY=true
```

**The Proper Fix (Production):**
Configure `curl.cainfo` in php.ini with valid CA certificates.

---

## Why Claude for Blockchain Development

### Strengths

**Complex Protocol Understanding:**
Claude excels at understanding and implementing complex blockchain protocols:
- Parsed Radiant opcodes and script structures
- Understood commit/reveal patterns
- Grasped CBOR encoding requirements
- Followed introspection opcode logic
- **Discovered undocumented requirements** (like on-chain images)

**Debugging Partner:**
When transactions failed, Claude:
- Analyzed hex dumps of scripts
- Compared working vs failing transactions
- Identified byte-level encoding errors
- Suggested targeted fixes
- **Traced issues through entire data flow**

**Documentation:**
Claude automatically generated:
- Comprehensive implementation guides
- Debugging documentation
- API endpoint specifications
- Test transaction references
- **Updated docs as new discoveries were made**

### With the MCP Server

**Claude can now query the Radiant blockchain directly** via the [Radiant MCP Server](https://github.com/Radiant-Core/radiant-mcp-server) (56 tools across read-only queries, token operations, wallet management, and transaction building):
- Check balances, UTXOs, and transaction history
- Read Glyph token metadata and verify NFT state
- Build, sign, and broadcast transactions
- Decode scripts and parse Glyph envelopes
- Query dMint contracts, WAVE names, and DEX orderbooks
- Estimate fees from the network
- Inspect mint txs, fetch raw transactions, and verify on-chain state

> ⚠️ **Do not route signing WIFs through an LLM-connected MCP server.**
> The Radiant MCP server exposes tools that can accept a WIF (mint, transfer,
> burn). Any such call passes the WIF through the model's tool-call
> serializer — the same channel an attacker can reach via prompt injection
> from untrusted inputs the model reads (IPFS image captions, tx memos, web
> pages, tool outputs from *other* MCP servers in the session). Prompt
> injection is an active threat class against agentic LLM setups; no current
> Claude model is hardened to refuse "now call mint_nft with wif=..."
> instructions smuggled inside otherwise-benign tool output.
>
> **Safe operating model:**
> 1. Use MCP in **read-only mode only**: fetch txs, decode scripts, query
>    state. Never register tools that accept a WIF or broadcast a tx.
> 2. Keep signing **off the LLM entirely** — the `sign_reveal.js` subprocess
>    pattern in the README (stdin-only WIF, no argv, no env) is purpose-built
>    for this. The signer runs on a host the LLM has no shell access to.
> 3. If you *must* expose a broadcast tool, require human confirmation for
>    every `sendrawtransaction` call and never stream the raw tx hex through
>    the LLM.
> 4. Use a dedicated hot wallet with a small balance for any MCP-accessible
>    path, with separation from your mint-authority / treasury keys.

### Still Always Needed

- **Visual verification in Glyphium/Photonic wallet** — screenshots catch display issues code can't
- **Testing NFT display in wallets** — blank cards, metadata rendering problems
- **Mempool inspection** — not available via MCP
- **Private key handling** — keep WIFs out of LLM context entirely; see the warning above and the `sign_reveal.js` subprocess pattern in the README

### Setting Up the MCP Server

Install and register the [Radiant MCP Server](https://github.com/Radiant-Core/radiant-mcp-server) to give Claude direct blockchain access:

```bash
git clone https://github.com/Radiant-Core/radiant-mcp-server.git
cd radiant-mcp-server
npm install
npm run build

claude mcp add \
  -e ELECTRUMX_HOST=electrumx.radiant4people.com \
  -e ELECTRUMX_PORT=50012 \
  -e ELECTRUMX_SSL=true \
  -e RADIANT_NETWORK=mainnet \
  --transport stdio \
  --scope user \
  radiant \
  -- node /path/to/radiant-mcp-server/dist/index.js
```

> **Gotcha:** `-e` environment flags must come BEFORE `--transport` and `--scope`, or you get "Invalid environment variable format" error.

Restart your Claude Code session after adding. Verify with `claude mcp list`.

---

## Project Setup

### 1. Start with Your Environment

**Tell Claude About Your Stack:**

```
"I'm building on Radiant blockchain. Here's my environment:
- Radiant node via Docker (RPC on port 7332)
- PHP backend for RPC calls
- JavaScript frontend
- Node.js 18+ available
- Windows development machine

I want to implement Glyph NFTs using commit/reveal pattern."
```

### 2. Share Existing Code

If you have existing blockchain code:

```
"Here's my current blockchain_rpc.php file that handles RPC calls.
I need to add NFT minting functionality."

[Paste or upload your code]
```

Claude will analyze your patterns and match the coding style.

### 3. Reference Documentation

Provide Claude with:
- Radiant opcode documentation
- Glyph protocol specs
- Example transactions (if available)

```
"The Radiant opcode summary is at: https://radiant4people.com/programming/radiant/opcode_summary/

Key opcodes:
- 0xd8: OP_PUSHINPUTREFSINGLETON
- 0xc8: OP_OUTPOINTTXHASH (NOT 0xc1!)
- 0xc9: OP_OUTPOINTINDEX (NOT 0xc2!)
"
```

### 4. Critical Setup Checklist

Before writing any code, ensure:

- [ ] CBOR library downloaded and added to HTML **before** blockchain scripts
- [ ] Radiant node accessible via RPC
- [ ] Node.js 18+ installed for signing scripts
- [ ] `@radiantblockchain/radiantjs` installed
- [ ] IPFS provider configured (Pinata recommended)

---

## Working with Claude

### Effective Prompting Strategies

#### Be Specific About Radiant Differences

```
❌ BAD: "How do I calculate transaction fees?"

✅ GOOD: "Radiant uses photons/byte for fees (NOT photons/kB like Bitcoin).
The minimum is 1000 photons/byte (increasing to 10,000 after block 415,000).
Here's my current code that's wrong:
$fee = ($txSize * $feeRate) / 1000;
How should I fix this?"
```

#### Share Visual Evidence

```
✅ GOOD: "My NFTs mint successfully but show as blank cards in Glyphium.
[Attach screenshot of wallet showing blank NFTs]

Here's my payload structure:
{
    p: [2],
    name: 'Test NFT',
    loc: 'ipfs://Qm...'
}

What's missing?"
```

This led to discovering the `main` field requirement.

#### Include Context from Multiple Sources

```
"Here's what I'm seeing:

Console log: 'Thumbnail: 150x200, 14538 bytes'
PHP log: 'IPFS upload failed: SSL certificate error'
Wallet display: NFT shows with placeholder image

The full-res upload is failing. What should I check?"
```

### Iterative Development

**Start Simple, Then Enhance:**

1. **Phase 1:** Get basic NFT minting working
   ```
   "Let's first create a simple NFT with just name and protocol fields."
   ```

2. **Phase 2:** Add on-chain images
   ```
   "NFTs are blank in wallet. Let's add thumbnail support."
   ```

3. **Phase 3:** Add IPFS integration
   ```
   "Now add IPFS for full-resolution backup images."
   ```

4. **Phase 4:** Optimize
   ```
   "Thumbnails are too large (15KB). Let's find optimal size/quality balance."
   ```

**Document Each Success:**

```
"The NFT with thumbnail worked! Cost was 0.16 RXD.
Here's the reveal txid: 27390efab1e3168c05301b18f6cdfd553a6d122a41496d0f5e104e79a918be7e
Please document this for future reference."
```

---

## Glyph NFT Implementation

### Complete Payload Structure

Based on real-world testing, here's what a complete payload looks like:

```javascript
const payload = {
    p: [2],                           // Protocol: NFT (REQUIRED)
    name: "Score: 1,234,567",         // Display name (REQUIRED)
    type: "photo",                    // Type (optional)
    main: {                           // On-chain image (REQUIRED for display!)
        t: 'image/webp',             // Must match thumbnail format
        b: thumbnailUint8Array        // NOT base64!
    },
    loc: 'ipfs://Qm...',              // Full-res backup (optional)
    attrs: {                          // Custom attributes (optional)
        score: 1234567,
        game: "Medieval Madness",
        player: "Alice",
        tournament: "Weekly Tournament",
        trust_level: "high"
    }
};
```

### Thumbnail Creation

**Optimized settings based on testing:**

```javascript
async createThumbnail(dataUrl, maxSize = 225, quality = 0.90) {
    return new Promise((resolve, reject) => {
        const img = new Image();
        img.onload = () => {
            let width = img.width;
            let height = img.height;

            // Scale while maintaining aspect ratio
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
            ctx.imageSmoothingQuality = 'high';  // Important for quality!
            ctx.drawImage(img, 0, 0, width, height);

            canvas.toBlob((blob) => {
                const reader = new FileReader();
                reader.onload = () => {
                    const uint8Array = new Uint8Array(reader.result);
                    console.log(`Thumbnail: ${width}x${height}, ${uint8Array.length} bytes`);
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

### CBOR Encoding with Verification

```javascript
function encodeGlyphData(data) {
    // CRITICAL: Verify CBOR is loaded
    if (typeof CBOR === 'undefined') {
        throw new Error('CBOR library not loaded! NFT would be unreadable.');
    }

    // Set protocol if missing
    if (!data.p) {
        data.p = [2];
    }

    // Encode
    const marker = new TextEncoder().encode('gly');
    const cborData = CBOR.encode(data);
    const payload = new Uint8Array(cborData);

    // Combine
    const result = new Uint8Array(marker.length + payload.length);
    result.set(marker, 0);
    result.set(payload, marker.length);

    console.log(`Glyph payload: ${result.length} bytes`);
    return result;
}
```

---

## Critical Gotchas

### 1. Blank NFTs in Wallet

**Symptom:** NFT mints successfully, shows in explorer, but blank card in wallet.

**Cause:** Missing `main` field with on-chain image.

**Fix:**
```javascript
payload.main = {
    t: 'image/webp',        // Must match thumbnail format
    b: thumbnailUint8Array  // Must be Uint8Array!
};
```

### 2. "Unknown NFT" with No Metadata

**Symptom:** NFT appears but shows "Unknown NFT", no attributes visible.

**Cause:** CBOR library not loaded, falling back to JSON.

**Fix:**
1. Download CBOR library
2. Load **before** blockchain scripts in HTML
3. Verify: `console.log(typeof CBOR)` should output "object"

### 3. Fee Calculation (Radiant-Specific)

**The Bug:**
```php
// WRONG - Bitcoin-style calculation
$fee = ($txSize * $feeRate) / 1000;
```

**The Fix:**
```php
// CORRECT - Radiant uses photons/byte (photons are Radiant's smallest unit)
// Pre-V2: 1000 photons/byte. Post-block 415,000: 10,000 photons/byte (10x).
$fee = $txSize * $feeRate;
```

### 4. Singleton Ref Format

**The Bug:**
```php
// WRONG - Added push opcode
$singletonScript = 'd824' . $ref . '7576a914' . $pubkeyhash . '88ac';
```

**The Fix:**
```php
// CORRECT - OP_PUSHINPUTREFSINGLETON consumes next 36 bytes directly
$singletonScript = 'd8' . $ref . '7576a914' . $pubkeyhash . '88ac';
```

### 5. Signing Subscript

**The Bug:**
```javascript
// WRONG - Signing with P2PKH only
const p2pkhScript = Script.buildPublicKeyHashOut(address);
const sig = Transaction.Sighash.sign(tx, privateKey, sigType, 0, p2pkhScript, amount);
```

**The Fix:**
```javascript
// CORRECT - Sign with full commit script
tx.setInputScript(0, (txObj, output) => {
    const sig = Transaction.Sighash.sign(
        txObj, privateKey, sigType, 0,
        output.script,  // ✅ Full nftCommitScript
        commitAmount
    );
    // ...
});
```

### 6. Poor Thumbnail Quality

**The Bug:**
```javascript
// Aggressive compression
const thumbnail = await createThumbnail(dataUrl, 100, 0.5);
// Result: Blurry, pixelated images
```

**The Fix:**
```javascript
// Balanced settings
const thumbnail = await createThumbnail(dataUrl, 225, 0.90);
// Result: Sharp images, good size (~22KB)
```

### 7. Orphaned Child NFTs (Container Refs)

**The Bug:**
```javascript
// WRONG - Computing ref from reveal txid
const containerRef = reverseHex(containerTxid) + '00000000';
payload.in = [hexToUint8Array(containerRef)];
// Result: NFT exists but doesn't appear in container
```

**The Fix:**
```javascript
// CORRECT - Extract ref from output script
const tx = await rpc.call('getrawtransaction', [containerTxid, true]);
const script = tx.vout[0].scriptPubKey.hex;
const containerRef = script.substring(2, 74);  // Skip d8, take 72 chars
payload.in = [hexToUint8Array(containerRef)];
// Result: NFT properly nested in container
```

**Why:** The singleton ref is based on the COMMIT transaction, not the reveal. You MUST extract it from the actual output script.

### 8. SSL Certificate Errors (IPFS)

**Symptom:** "SSL certificate problem: unable to get local issuer certificate"

**Development Fix:**
```bash
# In .env
IPFS_SKIP_SSL_VERIFY=true
```

**Production:** Configure proper CA certificates.

---

## Debugging Workflow

### When NFTs Don't Display

**1. Check CBOR Encoding**
```javascript
console.log(typeof CBOR);  // Should be "object"
```

**2. Check Payload Structure**
```javascript
console.log('Payload:', JSON.stringify(payload, null, 2));
// Verify: has p, name, main fields
```

**3. Check Thumbnail**
```javascript
console.log('Thumbnail type:', typeof payload.main?.b);
// Should be "object" (Uint8Array)
console.log('Thumbnail size:', payload.main?.b?.length);
// Should be 15000-25000 bytes for 225px WebP
```

**4. Check Transaction**
- View on Glyph Explorer: `https://glyph-explorer.rxd-radiant.com/tx/<txid>`
- Verify metadata decodes correctly

### When Transactions Fail

**1. Get the Full Error**

If you're running Radiant inside Docker (the usual case), you need the container
name and the `-datadir` flag or the CLI won't find the data directory:

```bash
docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant \
  sendrawtransaction "0200000001..."
# Note exact error message
```

On a host install the bare form works:

```bash
radiant-cli sendrawtransaction "0200000001..."
```

**2. Share with Claude**
```
"Transaction rejected: bad-txns-inputs-outputs-invalid-transaction-reference-operations

Raw TX: 0200000001263c6922...
Commit txid: 6afb402d...

What's wrong?"
```

**3. Compare Against Working Transaction**
```
"Here's a working reveal TX: 27390efab1e3168c...
And my failing TX: 0200000001...

Can you compare the structures?"
```

---

## Testing Strategy

### Verification Checklist

Before minting to mainnet:

- [ ] `typeof CBOR === 'object'` in browser console
- [ ] Thumbnail is Uint8Array, < 30KB
- [ ] Payload has `p`, `name`, and `main` fields
- [ ] IPFS upload succeeds (if using)
- [ ] Test CBOR encode/decode round-trip

### Test on Testnet First

```
"Let's test on Radiant testnet first.
Testnet RPC: localhost:18332

Create a minimal NFT to verify:
1. CBOR encoding works
2. On-chain thumbnail displays
3. Full mint flow completes"
```

### Verify in Multiple Places

After minting:
1. **Glyph Explorer** - Does metadata show?
2. **Glyphium Wallet** - Does image display?
3. **`radiant-cli gettxout <txid> <vout>`** - Does UTXO exist? In Docker: `docker exec radiant-node radiant-cli -datadir=/home/radiant/.radiant gettxout <txid> <vout>`

---

## Code Patterns That Work

### Pattern 1: Complete Mint Flow

```javascript
async mintNFT(imageDataUrl, metadata) {
    // 1. Create thumbnail
    const thumbnail = await this.createThumbnail(imageDataUrl, 225, 0.90);
    console.log(`Thumbnail: ${thumbnail.length} bytes`);

    // 2. Upload full-res to IPFS
    const ipfs = await this.uploadToIPFS(imageDataUrl);
    console.log(`IPFS: ${ipfs.url}`);

    // 3. Build payload with main field
    const payload = {
        p: [2],
        name: metadata.name,
        main: { t: 'image/webp', b: thumbnail },
        loc: ipfs.url,
        attrs: metadata.attrs
    };

    // 4. Encode and mint
    const glyphData = this.encodeGlyphData(payload);
    const result = await this.createSingletonNFT(glyphData);

    return result;
}
```

### Pattern 2: Defensive CBOR Check

```javascript
function encodeGlyphData(data) {
    if (typeof CBOR === 'undefined') {
        throw new Error(
            'CBOR library not loaded! ' +
            'Add <script src="js/cbor.min.js"></script> BEFORE blockchain scripts.'
        );
    }
    // ... rest of encoding
}
```

### Pattern 3: Thumbnail Size Validation

```javascript
async createThumbnail(dataUrl, maxSize, quality) {
    const thumbnail = await this._createThumbnail(dataUrl, maxSize, quality);

    if (thumbnail.length > 30000) {
        console.warn(`Thumbnail large (${thumbnail.length} bytes). Consider reducing.`);
    }

    if (thumbnail.length < 1000) {
        console.warn(`Thumbnail very small. May appear low quality.`);
    }

    return thumbnail;
}
```

---

## Resources

### Documentation to Share with Claude

**Radiant Blockchain:**
- Main site: https://radiantblockchain.org
- Opcode summary: https://radiant4people.com/programming/radiant/opcode_summary/
- Glyph protocol: https://radiantblockchain.org/glyphs-protocol-guide.html

**Libraries:**
- radiantjs: https://github.com/Radiant-Core/radiantjs
- Photonic Wallet: https://github.com/Radiant-Core/Photonic-Wallet
- CBOR: https://github.com/paroga/cbor-js

**AI Development Tools:**
- Radiant MCP Server: https://github.com/Radiant-Core/radiant-mcp-server
- AI Knowledge Base: https://github.com/Radiant-Core/radiant-mcp-server/blob/master/docs/RADIANT_AI_KNOWLEDGE_BASE.md

**Ecosystem Tools:**
- RXinDexer (token indexer): https://github.com/Radiant-Core/RXinDexer
- Glyph-miner (dMint mining): https://github.com/Radiant-Core/Glyph-miner

**Explorers:**
- Mainnet: https://explorer.radiantblockchain.org
- Glyph explorer: https://glyph-explorer.rxd-radiant.com

### Example Prompts for Common Tasks

**Adding On-Chain Thumbnails:**
```
"My NFTs show as blank in Glyphium wallet.

I discovered I need a 'main' field with on-chain image data.
Can you:
1. Create a thumbnail function (225px WebP, 90% quality)
2. Add main field to payload with {t: mimetype, b: Uint8Array}
3. Keep IPFS link in loc field for full-res"
```

**Debugging Unknown NFTs:**
```
"NFTs show as 'Unknown NFT' with no metadata.
Browser console shows: 'CBOR library not loaded, using JSON fallback'

How do I:
1. Download and include CBOR library correctly
2. Verify it's loaded before minting
3. Prevent silent fallback to JSON"
```

**Optimizing Thumbnail Size:**
```
"My thumbnails are 30KB each, making minting expensive (~0.3 RXD).

Current settings: 300px, 80% quality
What settings give good visual quality at lower cost?"
```

---

## Lessons Learned

### What Worked

1. **Visual Evidence**
   - Screenshots of blank NFTs led to main field discovery
   - Console logs revealed CBOR loading issue

2. **Iterative Improvement**
   - Started with 200px thumbnails (~15KB)
   - Tested 100px (too blurry)
   - Settled on 225px WebP (best quality-to-size ratio)

3. **Comprehensive Logging**
   - Log thumbnail size after creation
   - Log CBOR library status
   - Log IPFS upload results

4. **Reference Transactions**
   - Kept verified working transactions
   - Compared hex when debugging

### What Didn't Work

1. **Silent Fallbacks**
   - JSON fallback made debugging hard
   - Solution: Fail loudly instead

2. **Over-optimizing Early**
   - 100px thumbnails saved bytes but looked bad
   - Solution: Get it working first, then optimize

3. **Assuming IPFS = Display**
   - Thought IPFS link would show image
   - Reality: Need on-chain thumbnail

---

## Conclusion

Claude AI is a powerful partner for Radiant blockchain development. The key discoveries from this project:

1. **On-chain images are required** - Use the `main` field
2. **CBOR encoding is mandatory** - JSON = "Unknown NFT"
3. **225px WebP, 90% quality** is the optimal thumbnail setting
4. **Verify visually** - Screenshots catch issues code can't

**Keys to Success:**
1. Be specific about Radiant-specific behaviors
2. Share screenshots when things look wrong
3. Verify CBOR library is loaded before minting
4. Test in Glyphium wallet, not just explorer
5. Document working transactions for reference

**Cost Reference (pre-V2 / post-V2):**
- Minimal NFT (no image): ~0.01 / ~0.1 RXD
- With 225px WebP thumbnail: ~0.22 / ~2.2 RXD
- With 150px JPEG (budget): ~0.07 / ~0.69 RXD

> Post-block 415,000: minimum relay fee increases 10x. Use `estimatefee` RPC instead of hardcoding.

Happy building on Radiant!

---

**Community Resources:**
- Radiant Discord: https://discord.gg/radiantblockchain
- Developer Chat: #development channel
- Glyph NFT Discussion: #glyphs channel

**Questions?**
Post in #development with:
- What you're trying to build
- Screenshots of issues
- Transaction IDs (if available)
- Code snippets

---

**Last Updated:** 2026-04-16
**Author:** Radiant Developer Community
**License:** MIT - Free to use and share

**Key Discoveries Documented:**
1. On-chain images (`main` field) required for wallet display
2. CBOR encoding mandatory (JSON = "Unknown NFT")
3. Optimal thumbnail: 225px WebP, 90% quality (~22KB, ~0.22 RXD pre-V2)
4. SSL certificate issues with IPFS in development
