# Indexing Co — Claude Code Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that helps you build and deploy blockchain data pipelines using [Indexing Co](https://indexing.co).

Tell Claude what onchain data you need, and it will walk you through creating filters, writing transformation functions, and deploying pipelines — all through natural conversation.

## What it does

Once installed, Claude Code can:

- **Set up data pipelines** — filter contracts, transform events, deliver to your database
- **Stream live data into Claude Code** — real-time blockchain events via the [Indexing Co MCP server](https://github.com/indexing-co/indexing-co-mcp), queryable with SQL
- **Write transformation functions** — JavaScript that reshapes raw block data into structured output
- **Generate SQL schemas** — table definitions that match your pipeline output
- **Deploy and manage pipelines** — create, test, backfill, and monitor via the Indexing Co API
- **Handle 100+ networks** — EVM, Solana, Aptos, Bitcoin, Cosmos, and more

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) installed
- An Indexing Co API key — sign up at [accounts.indexing.co](https://accounts.indexing.co)

## Install

Clone this repo and copy the skill folder into your Claude Code skills directory:

```bash
git clone https://github.com/indexing-co/indexing-co-pipeline-skill.git
cp -r indexing-co-pipeline-skill/skills/indexing-co-pipelines ~/.claude/skills/
```

That's it. Claude Code will pick up the skill automatically on your next session.

## Usage

Just ask Claude Code to help you with onchain data. The skill activates automatically when relevant. Examples:

```
> I want to index all Uniswap V2 swaps on Ethereum and send them to my Postgres database

> Set up a pipeline to track ERC-20 transfers for this contract: 0x...

> Help me monitor Aave lending events across Ethereum and Arbitrum

> I need to backfill historical token transfers for the last 6 months
```

Claude will walk you through each step: creating filters, writing the transformation function, testing it against a real block, generating the SQL schema, and deploying the pipeline.

## How it works

The skill teaches Claude Code about the Indexing Co pipeline API:

```
[Filter] --> [Transformation] --> [Destination]
  what         how to reshape       where to send
```

1. **Filter** — which contract/wallet addresses to watch
2. **Transformation** — a JavaScript function that extracts and reshapes events from raw block data
3. **Destination** — where to deliver the output (PostgreSQL, webhooks, Kafka, S3, and 16+ other adapters)

## Supported destinations

PostgreSQL, Webhooks (HTTP), WebSocket, DIRECT (stream to Claude Code), Kafka, Kinesis, Pulsar, GCP PubSub, AWS S3, GCS, MongoDB, BigQuery, Firestore, Neo4j, ArangoDB, MySQL, SQLite

## Real-time streaming with MCP

Pair this skill with the [Indexing Co MCP server](https://github.com/indexing-co/indexing-co-mcp) to stream live blockchain events directly into your Claude Code session. Events are stored in local SQLite and queryable with SQL — no external database needed.

```
> Stream all USDC transfers on Base into this session

> What are the largest transfers in the last 5 minutes?

> Show me which addresses are sending the most USDC
```

## Links

- [Indexing Co Docs](https://docs.indexing.co)
- [Indexing Co LLM Docs](https://docs.indexing.co/llms-full.txt)
- [Network Status](https://jiti.indexing.co/status)
- [Claude Code Docs](https://docs.anthropic.com/en/docs/claude-code)
