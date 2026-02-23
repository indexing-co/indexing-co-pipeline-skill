---
name: indexing-co-pipelines
description: "Build and deploy blockchain data pipelines with Indexing Co. Use when user wants to index onchain data, set up event listeners, stream blockchain events to databases/webhooks, write transformation functions, or backfill historical data."
---

# Indexing Co Data Pipelines

**Your job:** Help users build onchain data pipelines — from raw blockchain events to structured data in their database. A pipeline = filter + transformation + destination.

|                |                                          |
| -------------- | ---------------------------------------- |
| **Base URL**   | `https://app.indexing.co/dw`             |
| **Auth**       | `X-API-KEY: {key}` header               |
| **Docs**       | `https://docs.indexing.co`               |
| **Sign up**    | `accounts.indexing.co` |

---

## Credentials

Check `~/.indexing-co/credentials` on every session start:

**File exists with `API_KEY`** -> load it. Don't prompt.

**File missing** -> ask user for their API key. Tell them to register at `accounts.indexing.co` if they don't have one. Save:

```bash
mkdir -p ~/.indexing-co && cat > ~/.indexing-co/credentials << 'EOF'
API_KEY=...
EOF
```

---

## Pipeline Architecture

Every pipeline has 3 parts. Build them in order:

```
[Filter] --> [Transformation] --> [Destination]
  what         how to reshape       where to send
```

| Component        | What it does                                        |
| ---------------- | --------------------------------------------------- |
| **Filter**       | Which addresses/contracts to watch                  |
| **Transformation** | JavaScript function that reshapes raw block data  |
| **Destination**  | Where processed data lands (Postgres, webhook, etc) |

---

## Step 1: Create a Filter

Filters target specific contract or wallet addresses. Supports millions of addresses per filter.

```bash
curl 'https://app.indexing.co/dw/filters/{FILTER_NAME}' \
  -H 'X-API-KEY: $API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"values": ["0x87870bca3f3fd6335c3f4ce8392d69350b4fa4e2"]}'
```

| Endpoint                  | Method   | Purpose                |
| ------------------------- | -------- | ---------------------- |
| `/filters/{name}`         | `POST`   | Create/update filter   |
| `/filters/{name}`         | `GET`    | List filter values     |
| `/filters/{name}`         | `DELETE` | Delete filter          |

---

## Step 2: Write a Transformation

JavaScript function that receives a `block` object and returns structured data. Runs server-side.

### Available Helpers

| Function | Purpose |
| -------- | ------- |
| `utils.evmDecodeLog(log, signatures[])` | Decode EVM log against event signatures |
| `utils.evmDecodeLogWithMetadata(log, signatures[])` | Same but includes event name |
| `utils.evmMethodSignatureToHex(signature)` | Convert signature to topic0 hash |
| `utils.blockToTimestamp(block)` | Get block timestamp |
| `utils.evmChainToId(chain)` | Chain name to chainId |
| `templates.tokenTransfers(block)` | Extract all token transfers |

### Available Packages

`BigNumber.js`, `viem`, `borsh`, `bs58`, `Buffer`, `zlib`

### Template: EVM Event Extraction

```javascript
function transform(block) {
  const results = [];
  const signatures = [
    // Get exact signatures from contract ABI — include `indexed` keyword
    'event Transfer(address indexed from, address indexed to, uint256 value)'
  ];

  for (const tx of block.transactions || []) {
    for (const log of tx.receipt?.logs || []) {
      const decoded = utils.evmDecodeLogWithMetadata(log, signatures);
      if (decoded) {
        results.push({
          chain: block._network,
          block: Number(block.number),
          transaction_hash: tx.hash,
          log_index: log.logIndex,
          contract_address: log.address?.toLowerCase(),
          event_name: decoded.eventName,
          decoded: decoded.decoded,
        });
      }
    }
  }
  return results;
}
```

### Template: Token Transfers (Shortcut)

```javascript
function transform(block) {
  return templates.tokenTransfers(block);
}
```

### Finding Event Signatures

**Critical:** Signatures must include `indexed` keywords exactly as declared in the ABI.

1. Get contract ABI from Etherscan/block explorer
2. Find events with `"type": "event"` entries
3. Format as: `event Name(type1 indexed param1, type2 param2, ...)`

### Test Before Deploying

Dry-run a transformation against a real block:

```bash
curl 'https://app.indexing.co/dw/transformations/test?network=ethereum&beat=22282149&filter={FILTER}&filterKeys[0]=contract_address' \
  -H 'X-API-KEY: $API_KEY' \
  -F 'code=@transform.js'
```

### Register Transformation

```bash
curl 'https://app.indexing.co/dw/transformations/{NAME}' \
  -H 'X-API-KEY: $API_KEY' \
  -F 'code=@transform.js'
```

| Endpoint                    | Method | Purpose                  |
| --------------------------- | ------ | ------------------------ |
| `/transformations/{name}`   | `POST` | Register transformation  |
| `/transformations/test`     | `POST` | Test without committing  |
| `/transformations/{name}`   | `GET`  | Retrieve transformation  |
| `/transformations`          | `GET`  | List all transformations |

---

## Step 3: Choose a Destination

### PostgreSQL (Most Common)

```json
{
  "adapter": "POSTGRES",
  "connectionUri": "postgresql://user:pass@host:5432/db",
  "table": "events_table",
  "uniqueKeys": ["chain", "transaction_hash", "log_index"]
}
```

**Always set `uniqueKeys`** — delivery is at-least-once, unique keys prevent duplicates.

**Recommended schema pattern:**

```sql
CREATE TABLE events (
  chain TEXT NOT NULL,
  block BIGINT NOT NULL,
  transaction_hash TEXT NOT NULL,
  log_index INTEGER NOT NULL,
  contract_address TEXT NOT NULL,
  event_name TEXT NOT NULL,
  decoded JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  PRIMARY KEY (chain, transaction_hash, log_index)
);
```

### Webhook

```json
{
  "adapter": "HTTP",
  "connection": {
    "host": "https://your-endpoint.com/webhook",
    "headers": {"auth-key": "secret"}
  }
}
```

### WebSocket

```json
{
  "adapter": "WEBSOCKET",
  "connection": { "host": "ws://your-host.com/stream" }
}
```

### Other Adapters

`Kafka`, `Kinesis`, `Pulsar`, `GCP PubSub`, `AWS S3`, `GCS`, `MongoDB`, `BigQuery`, `Firestore`, `Neo4j`, `ArangoDB`, `MySQL`, `SQLite`

---

## Step 4: Deploy the Pipeline

```bash
curl 'https://app.indexing.co/dw/pipelines/' \
  -H 'X-API-KEY: $API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{
    "name": "my_pipeline",
    "transformation": "{TRANSFORM_NAME}",
    "filter": "{FILTER_NAME}",
    "filterKeys": ["contract_address"],
    "networks": ["ethereum"],
    "enabled": true,
    "delivery": {
      "adapter": "POSTGRES",
      "connectionUri": "postgresql://user:pass@host:5432/db",
      "table": "events",
      "uniqueKeys": ["chain", "transaction_hash", "log_index"]
    }
  }'
```

| Endpoint                          | Method   | Purpose                  |
| --------------------------------- | -------- | ------------------------ |
| `/pipelines`                      | `POST`   | Create pipeline          |
| `/pipelines`                      | `GET`    | List pipelines           |
| `/pipelines/{name}`               | `GET`    | Get pipeline details     |
| `/pipelines/{name}`               | `DELETE` | Disable pipeline         |
| `/pipelines/{name}/networks`      | `POST`   | Enable networks          |
| `/pipelines/{name}/networks`      | `DELETE` | Disable networks         |
| `/pipelines/{name}/backfill`      | `POST`   | Backfill historical data |

---

## Backfill Historical Data

After deploying a pipeline, backfill past blocks:

```bash
curl -X POST 'https://app.indexing.co/dw/pipelines/{NAME}/backfill' \
  -H 'X-API-KEY: $API_KEY' \
  -H 'Content-Type: application/json' \
  -d '{"network": "ethereum", "fromBlock": 18000000}'
```

---

## Supported Networks

**150+ networks** across EVM, SVM, MoveVM, UTXO, CosmWasm.

**Major EVM:** `ethereum`, `arbitrum`, `optimism`, `base`, `polygon`, `avalanche`, `fantom`, `bsc`, `celo`

**Non-EVM:** `solana`, `aptos`, `sui`, `bitcoin`, `cardano`, `cosmos`, `osmosis`

**Testnets:** `sepolia`, `base-sepolia`, `avalanche-fuji`

Network status: `https://jiti.indexing.co/status/{NETWORK_KEY}`

---

## Workflow Checklist

When a user asks to set up a pipeline, walk through:

1. **What data?** Identify contracts, events, chains
2. **Create filter** with target addresses
3. **Find event signatures** from ABI/block explorer
4. **Write transformation** using the template above
5. **Test transformation** against a real block
6. **Create destination table** (SQL schema from test output)
7. **Register transformation**
8. **Deploy pipeline**
9. **Backfill** if historical data needed

---

## Common Patterns

### DEX Swaps (Uniswap V2)

```javascript
// Filter: pool addresses
// Signature:
'event Swap(address indexed sender, uint256 amount0In, uint256 amount1In, uint256 amount0Out, uint256 amount1Out, address indexed to)'
```

### Aave Lending Events

```javascript
// Filter: Aave pool contract
// Signatures:
'event Supply(address indexed reserve, address user, address indexed onBehalfOf, uint256 amount, uint16 indexed referralCode)'
'event Borrow(address indexed reserve, address user, address indexed onBehalfOf, uint256 amount, uint8 interestRateMode, uint256 borrowRate, uint16 indexed referralCode)'
```

### ERC-20 Transfers

```javascript
// Filter: token contract address
// Signature:
'event Transfer(address indexed from, address indexed to, uint256 value)'
```

### Universal Token Transfers

```javascript
// No filter needed — captures all transfers
// Use the built-in template:
function transform(block) {
  return templates.tokenTransfers(block);
}
```

---

## TableMap (Multi-Table Routing)

Route different event types to different tables from a single pipeline. Return an object with table names as keys:

```javascript
function transform(block) {
  return {
    "supplies": [...suppliesArray],
    "borrows": [...borrowsArray]
  };
}
```

Configure delivery with `tableMap` instead of `table`.

---

## Reliability

- **At-least-once delivery** — duplicates prevented by `uniqueKeys`
- **Chain reorg handling** — automatic reprocessing of affected blocks
- **Sub-second latency** for real-time data
- **99.95% uptime** SLA

---

## References

| Resource | URL |
| -------- | --- |
| Full docs | [docs.indexing.co](https://docs.indexing.co) |
| LLM-optimized docs | [docs.indexing.co/llms-full.txt](https://docs.indexing.co/llms-full.txt) |
| Network status | [jiti.indexing.co/status](https://jiti.indexing.co/status) |
| Account signup | [accounts.indexing.co](https://accounts.indexing.co) |
