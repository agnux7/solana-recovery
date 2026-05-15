# I Clicked a Phishing Link and Lost My Solana Wallet. Here's How I Got $300 in Meme Tokens Back.

*A step-by-step story of sweeper bots, failed Jito bundles, and the one trick that actually worked.*

---

It happened in seconds.

I clicked a link. Signed what I thought was a routine transaction. By the time I checked my wallet, the SOL was gone. All of it.

But the tokens were still there.

103 meme tokens. About $300 worth. Sitting in a wallet I no longer controlled.

---

## The Sweeper Bot Problem

Here's what most people don't know about compromised Solana wallets: **the attacker doesn't just drain you once. They install a sweeper bot.**

A sweeper bot is a script that monitors your wallet 24/7. The moment any SOL lands — even 0.001 SOL — it's gone in under a second. Before you can blink. Before any transaction you try to send can confirm.

This is why Phantom said "Unable to send." This is why sending $1 of SOL from a friend's wallet did nothing. The bot ate it instantly.

The tokens sat there, mocking me. I could see them. I just couldn't touch them.

---

## What Doesn't Work

Before I tell you what worked, let me save you 2 hours.

**Sending SOL directly:** The sweeper drains it before your transfer transaction can use it for fees. Every time.

**Using Phantom:** It won't even let you sign — "Unable to send" because the wallet has no SOL.

**Jito Bundles (atomic transactions):** I spent hours on this. Bundles were "accepted" but never landed on-chain. Why? Jito simulates transactions before including them in a bundle. The compromised wallet had no SOL at simulation time, so the simulation failed silently. Every single time.

---

## The Solution: Make the Attacker's Bot Irrelevant

Here's the insight that changed everything:

**In Solana, the fee payer and the transaction signer don't have to be the same wallet.**

You can structure a transaction so that:
- A **clean "funder" wallet** pays ALL the fees
- The **compromised wallet** only signs as the token transfer authority — it never receives a single lamport

The sweeper bot has nothing to steal. No SOL ever touches the compromised wallet. The bot just... sits there, watching tokens leave.

```
Funder wallet  ──pays all fees──→  Solana network
                                         ↑
Compromised wallet ──signs transfer──→  tokens go to your new wallet
```

---

## The Technical Setup

The script processes tokens in batches of 4. Each batch:

1. Sets the funder as `feePayer` for the entire transaction
2. Creates destination ATAs (Associated Token Accounts) — paid by the funder
3. Transfers each token — authorized by the compromised wallet signature
4. Both wallets sign. Transaction broadcasts.

```javascript
tx.feePayer = funder.publicKey;  // funder pays everything

// For each token:
createAssociatedTokenAccountInstruction(
  funder.publicKey,  // funder pays ATA creation (~0.002 SOL each)
  destATA, destination, mint, programId
)
createTransferCheckedInstruction(
  sourceATA, mint, destATA,
  compromised.publicKey,  // compromised only authorizes the transfer
  amount, decimals
)

tx.sign(funder, compromised);  // both sign
```

The compromised wallet signs — which it can, because signing doesn't require SOL, only the private key — but it never holds fees.

**The sweeper bot cannot intercept what never arrives.**

---

## What It Cost

Each batch of 4 tokens costs roughly:
- 4 × 0.00204 SOL for ATA creation = ~0.008 SOL
- Priority fee + base fee = ~0.002 SOL
- **Total per batch: ~0.01 SOL**

For 103 tokens (26 batches): about **0.26 SOL** total, or ~$35 at the time.

Not ideal. But better than losing $300.

---

## The Result

Three runs. Two mid-run funder recharges (the funder kept running out of SOL). One very tense afternoon.

```
[1/26] Sending 4 tokens... ✓ 4Sd5AwRsr1GXbaab...
[2/26] Sending 4 tokens... ✓ 5939cJCs7Uwkc6CU...
...
[26/26] Sending 3 tokens... ✓ 2ZKSV1mQmoBgQzVA...

✓ Complete: 103/103 tokens moved.
```

Every single token landed in the new wallet. The sweeper bot watched helplessly as its inventory evaporated.

---

## Lessons I'll Never Forget

**1. One signature can cost you everything.**
Not every transaction asks for token approval. Some grant full wallet control. Read what you're signing. If you don't understand it, reject it.

**2. SPL tokens aren't auto-swept like SOL.**
The attacker's bot only targeted SOL. If you get compromised, move your tokens immediately — you may have a window.

**3. Keep a backup wallet with SOL for emergencies.**
A separate wallet with 0.1–0.5 SOL costs almost nothing. In this situation, it was the difference between recovering $300 and losing it.

**4. The funder-pays-fees pattern is powerful.**
This isn't just a recovery trick. Any time you need to authorize transactions from a wallet with no SOL, this pattern applies.

**5. Jito Bundles are great — but not for this.**
They're atomic and MEV-resistant. But if your transaction depends on a zero-balance wallet, simulation will kill you. Know the tool.

---

## The Full Script

I open-sourced everything, including the complete recovery script with explanations:

**github.com/agnux7/solana-recovery**

The script handles both SPL tokens and Token-2022, batches efficiently, skips already-transferred tokens on re-runs, and gives clear error messages when the funder runs dry.

If this ever happens to you — or someone you know — this is how you get your tokens back.

---

*Stay safe out there. Verify before you sign.*
