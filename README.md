# genesis-blake2b-guide

**From spare laptop to sovereign BLAKE2b Bitcoin miner:** your own Bitcoin Knots node, CONVOY DATUM Gateway in solo mode, and payouts to a hardware wallet. Written by an analytics guy who is not a sysadmin, with an AI assistant reading the official docs alongside him, and every security-critical step verified by hand.

## What's here

| File | What it is | Who it's for |
|---|---|---|
| [`genesis-sovereign-mining-guide.pdf`](genesis-sovereign-mining-guide.pdf) | The journal/guide, formatted for reading or printing | You |
| [`genesis-sovereign-mining-guide.md`](genesis-sovereign-mining-guide.md) | Same content as the PDF, with copy-pasteable commands | You |
| [`genesis-assistant-briefing.md`](genesis-assistant-briefing.md) | A structured runbook with rules for the assistant: official sources first, one step at a time, never accept secrets, the human verifies signatures and addresses | Your AI assistant |

## Quick start

1. Read the guide (PDF or markdown).
2. Paste `genesis-assistant-briefing.md` into your AI assistant with a description of your hardware and say: *"Walk me through this one step at a time. I'll run each command and paste the output back."*
3. Do the GPG verification and the hardware-wallet address check yourself. That's the whole point.

## Official sources (these win over anything in this repo)

- https://bitcoin-blake2b.org/getting-started
- https://bitcoin-blake2b.org/mining
- https://bitcoinknots.org (Knots v29.4.1.knots20260508)
- https://github.com/bitcoinknots/bitcoin
- https://github.com/CONVOYMining/datum_gateway

## Status

The "Genesis" node has been live on the BLAKE2b chain since fork week, serving 100+ inbound peers from a laptop. Miners arrive mid-September 2026; the miner-side section of the guide will be filled in then.

Corrections and pull requests welcome, especially from people who know what they're doing.
