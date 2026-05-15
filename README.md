# Solana Token Recovery — Guía Completa

Recuperación de tokens SPL/Token-2022 desde una wallet comprometida con sweeper bot activo, sin enviar SOL a la wallet comprometida.

---

## El Problema

### Situación inicial
- Wallet comprometida: `<COMPROMISED_WALLET>`
- Causa: clic en enlace de phishing → firma de transacción maliciosa
- El atacante instaló un **sweeper bot**: un script que monitorea la wallet comprometida 24/7 y drena cualquier SOL que llegue en cuestión de segundos, antes de que pueda usarse para pagar fees
- Contenido a rescatar: 103 tokens meme (~$300 USD)
- Destino: nueva wallet limpia `<DESTINATION_WALLET>`

### Por qué no funcionaron las soluciones obvias

| Intento | Resultado | Motivo |
|---|---|---|
| Phantom → "Send" | "Unable to send" | Sin SOL para fees |
| Enviar 0.01 SOL desde otra wallet | Bot lo drena antes de poder usarlo | Sweeper bot reacciona en <1s |
| Jito Bundles (atomic) | Aceptados pero nunca aterrizaban | Simulación fallaba: wallet comprometida sin SOL al momento de simular |

---

## La Solución: Patrón "Funder Paga Todo"

### Concepto clave

En Solana, `feePayer` y el firmante de la instrucción pueden ser **wallets distintas**. Esto permite:

1. Una wallet **funder** (limpia, con SOL) actúa como `feePayer` de la transacción completa
2. La wallet **comprometida** solo firma como autoridad de la transferencia de tokens (no necesita SOL)
3. El sweeper bot nunca recibe SOL para robar → no puede interferir

```
Funder  ──paga fees──→  Red Solana
                              ↑
Comprometida ──firma transfer→ tokens van a Destino
```

### Arquitectura de cada transacción

```
tx.feePayer = funder.publicKey          // funder paga TODO

// Priority fee para confirmar rápido
ComputeBudgetProgram.setComputeUnitPrice({ microLamports: 200_000 })
ComputeBudgetProgram.setComputeUnitLimit({ units: 400_000 })

// Por cada token en el batch (hasta 4):
createAssociatedTokenAccountInstruction(
  funder.publicKey,  // funder paga la creación del ATA (~0.00204 SOL c/u)
  destATA, dest, mint, programId
)
createTransferCheckedInstruction(
  sourceATA, mint, destATA,
  compromised.publicKey,  // comprometida solo autoriza
  amount, decimals, [], programId
)

// Ambas wallets firman
tx.sign(funder, compromised)
```

---

## Detalles Técnicos

### Costo por token recuperado
- Creación de ATA destino: ~0.00204 SOL
- Fee de transacción + priority fee: ~0.001 SOL por tx
- Con 4 tokens por tx: ~0.009–0.01 SOL por batch
- **103 tokens totales: ~0.26 SOL en total**

### Batching (4 tokens por transacción)
Solana tiene un límite de tamaño de transacción (~1232 bytes). Con 4 tokens por tx se evita:
- Exceder el límite de tamaño
- Agotar el compute budget
- Gastar demasiado SOL en un solo intento

### Soporte Token-2022
El script detecta y maneja ambos programas de token:
- `TOKEN_PROGRAM_ID` — tokens SPL clásicos
- `TOKEN_2022_PROGRAM_ID` — tokens con extensiones (transfer fees, etc.)

Se pasa `programId` explícitamente a `createAssociatedTokenAccountInstruction` y `createTransferCheckedInstruction`.

### Verificación eficiente de ATAs
En lugar de 103 llamadas RPC individuales:
```js
// 1–2 llamadas en lugar de 103
const ataInfos = [];
for (let i = 0; i < prepared.length; i += 100) {
  const infos = await conn.getMultipleAccountsInfo(
    prepared.slice(i, i + 100).map(t => t.destATA)
  );
  ataInfos.push(...infos);
}
```

### Re-ejecución segura (idempotencia)
El script filtra tokens con `balance > 0n`. Al re-correrlo después de un fallo parcial, solo procesa los tokens que aún no se han transferido.

---

## Errores Encontrados y Soluciones

### `rpc-websockets` module not found (Node 25+)
```
Error: Cannot find module 'rpc-websockets'
```
**Fix:** Pinear versión compatible en `package.json`:
```json
"rpc-websockets": "7.6.1"
```

### "bad secret key size" (38 bytes)
La private key se copió truncada. El error correcto:
```
Tamaño de private key inválido: 38 bytes (esperado 64 o 32)
```
**Fix:** Volver a copiar la private key completa desde la wallet (~88 caracteres en base58).

### "custom program error: 0x1" en Instrucción N
El funder se quedó sin SOL a mitad de la ejecución.
```
Error: custom program error: 0x1
```
**Fix:** Recargar el funder con 0.05+ SOL y volver a correr el script. Los tokens ya transferidos se saltan automáticamente.

### Jito bundles aceptados pero sin aterrizaje
Los bundles se aceptaban (`bundle accepted`) pero nunca confirmaban. Causa probable: Jito simula las transacciones del bundle de forma independiente; la wallet comprometida sin SOL falla la simulación.
**Fix:** Abandonar Jito. El patrón estándar con funder como feePayer funciona sin necesidad de bundles atómicos porque el sweeper bot no puede robar si nunca recibe SOL.

---

## El Script

```javascript
// recover.js — Recuperación de tokens Solana
// El funder paga todos los fees directamente: no se envía SOL al compromised wallet,
// así el sweeper bot no tiene nada que robar.

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
  throw new Error(`Tamaño de private key inválido: ${decoded.length} bytes (esperado 64 o 32).`);
}

const CONFIG = {
  // Private key de tu wallet COMPROMETIDA (base58, ~88 caracteres)
  compromisedKey: 'TU_PRIVATE_KEY_COMPROMETIDA_AQUI',

  // Private key de tu wallet FONDEADORA (wallet limpia con al menos 0.05 SOL)
  funderKey: 'TU_PRIVATE_KEY_FUNDER_AQUI',

  // Dirección pública de tu NUEVA wallet destino
  destination: 'TU_WALLET_DESTINO_AQUI',

  // Tokens a procesar por transacción (no subas de 5)
  tokensPerTx: 4,

  // true = solo simulación, false = ejecutar de verdad
  dryRun: true,
};

const RPC = 'https://api.mainnet-beta.solana.com';

async function main() {
  const conn        = new Connection(RPC, 'confirmed');
  const compromised = keypairFromPrivateKey(CONFIG.compromisedKey);
  const funder      = keypairFromPrivateKey(CONFIG.funderKey);
  const dest        = new PublicKey(CONFIG.destination);

  console.log('\n=== SOLANA WALLET RECOVERY ===');
  console.log('Wallet comprometida :', compromised.publicKey.toBase58());
  console.log('Funder (paga fees)  :', funder.publicKey.toBase58());
  console.log('Destino             :', CONFIG.destination);
  console.log('Modo                :', CONFIG.dryRun ? 'DRY RUN (simulación)' : 'REAL');

  const funderBal = await conn.getBalance(funder.publicKey);
  console.log(`Balance funder: ${(funderBal / LAMPORTS_PER_SOL).toFixed(5)} SOL`);
  if (funderBal < 0.01 * LAMPORTS_PER_SOL) {
    throw new Error('El funder necesita al menos 0.01 SOL.');
  }

  console.log('Buscando tokens...');
  const [splRes, t22Res] = await Promise.all([
    conn.getParsedTokenAccountsByOwner(compromised.publicKey, { programId: TOKEN_PROGRAM_ID }),
    conn.getParsedTokenAccountsByOwner(compromised.publicKey, { programId: TOKEN_2022_PROGRAM_ID }),
  ]);

  const tokens = [
    ...splRes.value.map(a => ({ ...a, programId: TOKEN_PROGRAM_ID })),
    ...t22Res.value.map(a => ({ ...a, programId: TOKEN_2022_PROGRAM_ID })),
  ].filter(a => BigInt(a.account.data.parsed.info.tokenAmount.amount) > 0n);

  if (tokens.length === 0) {
    console.log('No hay tokens con balance > 0.');
    return;
  }

  console.log(`Tokens encontrados: ${tokens.length}`);
  for (const t of tokens) {
    const i = t.account.data.parsed.info;
    console.log(`  ${i.mint} | ${i.tokenAmount.uiAmountString}`);
  }

  if (CONFIG.dryRun) {
    console.log('\n✓ DRY RUN completado. Cambia dryRun a false para ejecutar.');
    return;
  }

  console.log('\nVerificando ATAs destino...');
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
  console.log(`  ATAs existentes: ${tokenData.length - newATAs} | Por crear: ${newATAs}`);

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
      process.stdout.write(`  [${batchNum}/${totalBatches}] Enviando ${batch.length} tokens... `);
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

  console.log(`\n✓ Completado: ${totalMoved}/${tokenData.length} tokens movidos.`);
  if (totalMoved < tokenData.length) {
    console.log('  Algunos batches fallaron. Corre el script de nuevo para intentar los restantes.');
  }
  console.log('  ⚠️  Borra este archivo cuando termines (contiene tus private keys).');
}

main().catch(err => {
  console.error('\n✗ Error fatal:', err.message);
  process.exit(1);
});
```

---

## Uso

### 1. Instalar dependencias
```bash
npm install
```

### 2. Configurar `recover.js`
Edita la sección `CONFIG`:
```js
const CONFIG = {
  compromisedKey: 'TU_PRIVATE_KEY_COMPROMETIDA',  // base58, ~88 chars
  funderKey:      'TU_PRIVATE_KEY_FUNDER',         // wallet limpia con SOL
  destination:    'TU_WALLET_DESTINO',             // dirección pública
  tokensPerTx:    4,
  dryRun:         true,  // empieza en true para verificar
};
```

### 3. Dry run primero
```bash
node recover.js
# Verifica que las wallets sean las correctas
```

### 4. Ejecutar real
Cambia `dryRun: false` y corre de nuevo:
```bash
node recover.js
```

### 5. Si el funder se queda sin SOL
El script reporta los batches que fallaron. Recarga el funder y vuelve a correr — los tokens ya transferidos se saltan automáticamente.

### 6. Al terminar
```bash
rm recover.js  # contiene tus private keys
```

---

## Cuánto SOL necesita el funder

| Tokens a recuperar | SOL estimado |
|---|---|
| 10 | ~0.03 SOL |
| 25 | ~0.07 SOL |
| 50 | ~0.13 SOL |
| 100 | ~0.26 SOL |

Siempre ten un 20% extra de margen por variación en priority fees.

---

## Resultado Final

| Métrica | Valor |
|---|---|
| Tokens recuperados | 103/103 |
| SOL gastado (fees + ATAs) | ~0.40 SOL total |
| Tiempo total | ~3 horas (incluyendo debugging) |
| Runs necesarios | 3 (funder se agotó 2 veces) |

---

## Lecciones

1. **Nunca firmes transacciones de sitios desconocidos.** Una sola firma puede dar acceso total a tu wallet.
2. **El sweeper bot no puede robarte SOL que nunca llega.** El patrón funder-paga-todo es la solución correcta.
3. **Mueve tus tokens antes de que el atacante los drene.** Los tokens SPL no se roban automáticamente como el SOL — hay una ventana de tiempo.
4. **Ten siempre una wallet funder separada con algo de SOL** para emergencias como esta.
5. **Jito Bundles no son necesarios** si el diseño de la transacción nunca le da SOL al sweeper bot.

---

## Dependencias

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

> `rpc-websockets` debe pinarse en `7.6.1` — versiones más nuevas tienen un formato incompatible con `@solana/web3.js` 1.91.8 en Node 20+.
