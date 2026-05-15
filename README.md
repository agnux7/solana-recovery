# Solana Token Recovery — Complete Guide

Recover SPL/Token-2022 tokens from a compromised wallet with an active sweeper bot, without ever sending SOL to the compromised wallet.

---

## The Problem

### Initial situation
- Compromised wallet: `<COMPROMISED_WALLET>`
- Cause: clicked a phishing link → signed a malicious transaction
- The attacker installed a **sweeper bot**: a script that monitors the compromised wallet 24/7 and drains any incoming SOL within seconds, before it can be used to pay fees
- Assets to recover: 103 meme tokens (~$300 USD)
- Destination: new clean wallet `<DESTINATION_WALLET>`

### Why the obvious solutions didn't work

| Attempt | Result | Reason |
|---|---|---|
| Phantom → "Send" | "Unable to send" | No SOL for fees |
| Send 0.01 SOL from another wallet | Bot drains it before it can be used | Sweeper bot reacts in <1s |
| Jito Bundles (atomic) | Accepted but never landed | Simulation failed: compromised wallet had no SOL at simulation time |

---

## The Solution: Funder Pays Everything Pattern

### Core concept

In Solana, `feePayer` and the instruction signer can be **different wallets**. This allows:

1. A **funder** wallet (clean, with SOL) acts as `feePayer` for the entire transaction
2. The **compromised** wallet only signs as the token transfer authority (needs no SOL)
3. The sweeper bot never receives SOL to steal → it can't interfere

```
Funder  ──pays fees──→  Solana network
                              ↑
Compromised ──signs transfer→ tokens go to Destination
```

### Transaction structure per batch

```
tx.feePayer = funder.publicKey          // funder pays EVERYTHING

// Priority fee to confirm fast
ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 200_000 })
ComputeBudgetProgram.setComputeUnitLimit({ units: 400_000 })

// For each token in the batch (up to 4):
createAssociatedTokenAccountInstruction(
  funder.publicKey,  // funder pays ATA creation (~0.00204 SOL each)
  destATA, dest, mint, programId
)
createTransferCheckedInstruction(
  sourceATA, mint, destATA,
  compromised.publicKey,  // compromised only authorizes
  amount, decimals, [], programId
)

// Both wallets sign
tx.sign(funder, compromised)
```

---

## Technical Details

### Cost per recovered token
- Destination ATA creation: ~0.00204 SOL
- Transaction fee + priority fee: ~0.001 SOL per tx
- With 4 tokens per tx: ~0.009–0.01 SOL per batch
- **103 tokens total: ~0.26 SOL**

### Batching (4 tokens per transaction)
Solana has a transaction size limit (~1232 bytes). With 4 tokens per tx you avoid:
- Exceeding the size limit
- Exhausting the compute budget
- Spending too much SOL in a single attempt

### Token-2022 support
The script detects and handles both token programs:
- `TOKEN_PROGRAM_ID` — classic SPL tokens
- `TOKEN_2022_PROGRAM_ID` — tokens with extensions (transfer fees, etc.)

`programId` is passed explicitly to `createAssociatedTokenAccountInstruction` and `createTransferCheckedInstruction`.

### Efficient ATA verification
Instead of 103 individual RPC calls:
```js
// 1–2 calls instead of 103
const ataInfos = [];
for (let i = 0; i < prepared.length; i += 100) {
  const infos = await conn.getMultipleAccountsInfo(
    prepared.slice(i, i + 100).map(t => t.destATA)
  );
  ataInfos.push(...infos);
}
```

### Safe re-runs (idempotency)
The script filters tokens with `balance > 0n`. Re-running after a partial failure only processes tokens that haven't been transferred yet.

---

## Errors and Fixes

### `rpc-websockets` module not found (Node 25+)
```
Error: Cannot find module 'rpc-websockets'
```
**Fix:** Pin the compatible version in `package.json`:
```json
"rpc-websockets": "7.6.1"
```

### "bad secret key size" (38 bytes)
The private key was copied truncated. Clear error message:
```
Invalid private key size: 38 bytes (expected 64 or 32)
```
**Fix:** Re-copy the full private key from the wallet (~88 base58 characters).

### "custom program error: 0x1" at Instruction N
The funder ran out of SOL mid-run.
```
Error: custom program error: 0x1
```
**Fix:** Recharge the funder with 0.05+ SOL and re-run the script. Already-transferred tokens are automatically skipped.

### Jito bundles accepted but not landing
Bundles were accepted (`bundle accepted`) but never confirmed. Likely cause: Jito simulates bundle transactions independently; the compromised wallet with no SOL fails simulation.
**Fix:** Abandon Jito. The standard funder-as-feePayer pattern works without atomic bundles because the sweeper bot can't steal what it never receives.

---

## The Script

```javascript
// recover.js — Solana token recovery
// The funder pays all fees directly: no SOL is ever sent to the compromised wallet,
// so the sweeper bot has nothing to steal.

const {
  Connection, Keypair, PublicKey, Transaction,
  LAMPORTS_PER_SOL, ComputeBudgetProgram,
} = require('@solana/web3.js');
const {
  getAssociatedTokenAddressSync,
  createAssociatedTokenAccountInstruction,
  createTransferCheckedInstruction,
  TOKEN_PROGRAM_ID,
  TOKEN_2022_PROGRAM_ID,
} = require('@solana/spl-token');
const bs58    = require('bs58');
const bip39   = require('bip39');
const { derivePath } = require('ed25519-hd-key');

async function keypairFromMnemonic(mnemonic, path = "m/44'/501'/0'/0'") {
  const seed = await bip39.mnemonicToSeed(mnemonic.trim());
  const { key } = derivePath(path, seed.toString('hex'));
  return Keypair.fromSeed(key);
}

function keypairFromPrivateKey(privateKeyBase58) {
  const decoded = bs58.decode(privateKeyBase58);
  if (decoded.length === 64) return Keypair.fromSecretKey(decoded);
  if (decoded.length === 32) return Keypair.fromSeed(decoded);
  throw new Error(`Invalid private key size: ${decoded.length} bytes (expected 64 or 32).`);
}

const CONFIG = {
  // Private key of your COMPROMISED wallet (base58, ~88 characters)
  compromisedKey: 'YOUR_COMPROMISED_PRIVATE_KEY_HERE',

  // Private key of your FUNDER wallet (clean wallet with at least 0.05 SOL)
  funderKey: 'YOUR_FUNDER_PRIVATE_KEY_HERE',

  // Public address of your NEW destination wallet
  destination: 'YOUR_DESTINATION_WALLET_HERE',

  // Tokens to process per transaction (don't go above 5)
  tokensPerTx: 4,

  // true = simulation only, false = execute for real
  dryRun: true,
};

const RPC = 'https://api.mainnet-beta.solana.com';

async function main() {
  const conn        = new Connection(RPC, 'confirmed');
  const compromised = keypairFromPrivateKey(CONFIG.compromisedKey);
  const funder      = keypairFromPrivateKey(CONFIG.funderKey);
  const dest        = new PublicKey(CONFIG.destination);

  console.log('\n=== SOLANA WALLET RECOVERY ===');
  console.log('Compromised wallet  :', compromised.publicKey.toBase58());
  console.log('Funder (pays fees)  :', funder.publicKey.toBase58());
  console.log('Destination         :', CONFIG.destination);
  console.log('Mode                :', CONFIG.dryRun ? 'DRY RUN (simulation)' : 'REAL');

  const funderBal = await conn.getBalance(funder.publicKey);
  console.log(`Funder balance: ${(funderBal / LAMPORTS_PER_SOL).toFixed(5)} SOL`);
  if (funderBal < 0.01 * LAMPORTS_PER_SOL) {
    throw new Error('Funder needs at least 0.01 SOL.');
  }

  console.log('Searching for tokens...');
  const [splRes, t22Res] = await Promise.all([
    conn.getParsedTokenAccountsByOwner(compromised.publicKey, { programId: TOKEN_PROGRAM_ID }),
    conn.getParsedTokenAccountsByOwner(compromised.publicKey, { programId: TOKEN_2022_PROGRAM_ID }),
  ]);

  const tokens = [
    ...splRes.value.map(a => ({ ...a, programId: TOKEN_PROGRAM_ID })),
    ...t22Res.value.map(a => ({ ...a, programId: TOKEN_2022_PROGRAM_ID })),
  ].filter(a => BigInt(a.account.data.parsed.info.tokenAmount.amount) > 0n);

  if (tokens.length === 0) {
    console.log('No tokens with balance > 0.');
    return;
  }

  console.log(`Tokens found: ${tokens.length}`);
  for (const t of tokens) {
    const i = t.account.data.parsed.info;
    console.log(`  ${i.mint} | ${i.tokenAmount.uiAmountString}`);
  }

  if (CONFIG.dryRun) {
    console.log('\n✓ DRY RUN complete. Set dryRun to false to execute for real.');
    return;
  }

  console.log('\nChecking destination ATAs...');
  const prepared = tokens.map(t => {
    const info    = t.account.data.parsed.info;
    const mint    = new PublicKey(info.mint);
    const destATA = getAssociatedTokenAddressSync(mint, dest, false, t.programId);
    return { sourceATA: new PublicKey(t.pubkey), mint, amount: BigInt(info.tokenAmount.amount), decimals: info.tokenAmount.decimals, programId: t.programId, destATA };
  });

  const ataInfos = [];
  for (let i = 0; i < prepared.length; i += 100) {
    const infos = await conn.getMultipleAccountsInfo(prepared.slice(i, i + 100).map(t => t.destATA));
    ataInfos.push(...infos);
  }

  const tokenData = prepared.map((t, i) => ({ ...t, needsCreate: !ataInfos[i] }));
  const newATAs = tokenData.filter(t => t.needsCreate).length;
  console.log(`  Existing ATAs: ${tokenData.length - newATAs} | To create: ${newATAs}`);

  let totalMoved = 0;
  for (let i = 0; i < tokenData.length; i += CONFIG.tokensPerTx) {
    const batch = tokenData.slice(i, i + CONFIG.tokensPerTx);
    const batchNum = Math.floor(i / CONFIG.tokensPerTx) + 1;
    const totalBatches = Math.ceil(tokenData.length / CONFIG.tokensPerTx);

    const { blockhash } = await conn.getLatestBlockhash('finalized');
    const tx = new Transaction();
    tx.feePayer = funder.publicKey;
    tx.recentBlockhash = blockhash;

    tx.add(ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 200000 }));
    tx.add(ComputeBudgetProgram.setComputeUnitLimit({ units: 400000 }));

    for (const td of batch) {
      if (td.needsCreate) {
        tx.add(createAssociatedTokenAccountInstruction(
          funder.publicKey,
          td.destATA, dest, td.mint, td.programId
        ));
      }
      tx.add(createTransferCheckedInstruction(
        td.sourceATA, td.mint, td.destATA,
        compromised.publicKey,
        td.amount, td.decimals, [], td.programId
      ));
    }

    tx.sign(funder, compromised);

    try {
      process.stdout.write(`  [${batchNum}/${totalBatches}] Sending ${batch.length} tokens... `);
      const sig = await conn.sendRawTransaction(tx.serialize(), { skipPreflight: false, maxRetries: 3 });
      await conn.confirmTransaction(sig, 'confirmed');
      totalMoved += batch.length;
      console.log(`✓ ${sig.slice(0, 16)}...`);
    } catch (err) {
      console.log(`✗ Error: ${err.message}`);
    }

    if (i + CONFIG.tokensPerTx < tokenData.length) {
      await new Promise(r => setTimeout(r, 1000));
    }
  }

  console.log(`\n✓ Complete: ${totalMoved}/${tokenData.length} tokens moved.`);
  if (totalMoved < tokenData.length) {
    console.log('  Some batches failed. Re-run the script to retry the remaining ones.');
  }
  console.log('  ⚠️  Delete this file when done (it contains your private keys).');
}

main().catch(err => {
  console.error('\n✗ Fatal error:', err.message);
  process.exit(1);
});
```

---

## Usage

### 1. Install dependencies
```bash
npm install
```

### 2. Configure `recover.js`
Edit the `CONFIG` section:
```js
const CONFIG = {
  compromisedKey: 'YOUR_COMPROMISED_PRIVATE_KEY',  // base58, ~88 chars
  funderKey:      'YOUR_FUNDER_PRIVATE_KEY',        // clean wallet with SOL
  destination:    'YOUR_DESTINATION_ADDRESS',       // public address
  tokensPerTx:    4,
  dryRun:         true,  // start with true to verify
};
```

### 3. Dry run first
```bash
node recover.js
# Verify the wallet addresses are correct
```

### 4. Execute for real
Set `dryRun: false` and run again:
```bash
node recover.js
```

### 5. If the funder runs out of SOL
The script reports which batches failed. Recharge the funder and re-run — already-transferred tokens are skipped automatically.

### 6. When done
```bash
rm recover.js  # contains your private keys
```

---

## How much SOL does the funder need?

| Tokens to recover | Estimated SOL |
|---|---|
| 10 | ~0.03 SOL |
| 25 | ~0.07 SOL |
| 50 | ~0.13 SOL |
| 100 | ~0.26 SOL |

Always keep a 20% buffer for priority fee variance.

---

## Final Result

| Metric | Value |
|---|---|
| Tokens recovered | 103/103 |
| SOL spent (fees + ATAs) | ~0.40 SOL total |
| Total time | ~3 hours (including debugging) |
| Runs needed | 3 (funder ran out of SOL twice) |

---

## Lessons

1. **Never sign transactions from unknown sites.** A single signature can hand over full wallet control.
2. **The sweeper bot can't steal SOL that never arrives.** The funder-pays pattern is the correct solution.
3. **Move your tokens before the attacker drains them.** SPL tokens aren't auto-swept like SOL — there's a time window.
4. **Keep a separate funder wallet with some SOL** for emergencies like this.
5. **Jito Bundles aren't necessary** if your transaction design never gives SOL to the sweeper bot.

---

## Dependencies

```json
{
  "@solana/spl-token": "0.4.8",
  "@solana/web3.js": "1.91.8",
  "bip39": "3.1.0",
  "bs58": "4.0.1",
  "ed25519-hd-key": "1.3.0",
  "rpc-websockets": "7.6.1"
}
```

> `rpc-websockets` must be pinned to `7.6.1` — newer versions have an incompatible format with `@solana/web3.js` 1.91.8 on Node 20+.
