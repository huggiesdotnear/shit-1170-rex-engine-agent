# shit-1170-rex-engine-agent

💩 **REX** — the shit-1170 rex engine agent.

REX is the agent for the [shit-1170 rex engine](https://github.com/huggiesdotnear/shit-1170-rex-engine): a feed and a transaction queue for NEAR. REX prepares the work; a human signs it in their wallet.

- live site: https://shit-1170.com/
- the skills it draws on: [shit-1170-rex-engine-skills](https://github.com/huggiesdotnear/shit-1170-rex-engine-skills)

---

## install

REX is an [awesome-agents](https://github.com/pablof7z/awesome-agents) profile. Install it with the `awesome-agents` CLI:

```sh
# add
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex
# --list — see what is in the repo before installing
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --list
```

Pick the harness you want it in:

```sh
# one harness
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex --harness opencode
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex --harness claude-code
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex --harness codex

# every harness the CLI finds on your PATH
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex

# preview the writes without making them
bunx awesome-agents add huggiesdotnear/shit-1170-rex-engine-agent --agent rex --dry-run
```

Manage it afterwards:

```sh
bunx awesome-agents list
bunx awesome-agents update rex
bunx awesome-agents remove rex
```

### from a checkout

```sh
git clone https://github.com/huggiesdotnear/shit-1170-rex-engine-agent.git
cd shit-1170-rex-engine-agent
bunx awesome-agents add . --agent rex --harness opencode
```

### where it lands

| harness | global install | project install |
| --- | --- | --- |
| Codex | `~/.codex/rex.config.toml` | not supported |
| Claude Code | `~/.claude/agents/rex.md` | `.claude/agents/rex.md` |
| OpenCode | `~/.config/opencode/agents/rex.md` | `.opencode/agents/rex.md` |
| Goose | `~/.agents/agents/rex.md` | `.agents/agents/rex.md` |
| Hermes | `~/.hermes/profiles/rex/SOUL.md` | not supported |
| pi | `~/.pi/rex/` + a `pi-rex` launcher | not supported |

Run it:

```sh
opencode --agent rex        # opencode
claude --agent rex          # claude code
codex --profile rex         # codex
pi-rex                      # pi
```

---

## what REX does

REX reads NEAR chain state and the engine's feed, verifies contract calls against the chain, and prepares transactions and swaps as queued batches that a human signs.

**REX never signs or sends anything.** Every state-changing call becomes a queued payload — `{ contractId, method, args, gas, deposit }` — and the engine holds it until a human signs it in the web app.

REX is proactive: when you name an account, REX goes and looks up what it holds, what it can trade, whether it belongs to a DAO, what the feed holds, and what the account has done on chain, before answering.

| | |
| --- | --- |
| **read the chain** | contract methods, receiver accounts, balances, token metadata, holders, receipts |
| **read the engine** | the queued batches and swaps, who is waiting on what, what has been sent |
| **read the history** | an account's transaction history, and a transaction by its hash |
| **read the keys** | the access keys an account has, and which of them are limited |
| **verify** | that a contract and its methods exist, that gas and deposit are right, that the receiver can hold the token, that the amount matches the token's decimals |
| **queue** | a batch of calls, or a swap |
| **track** | watch a queued item until it is sent, then read the receipt on chain |

The engine's typed client is [`@huggies/shit-1170-rex-engine-middleman`](https://www.npmjs.com/package/@huggies/shit-1170-rex-engine-middleman).

### the skills it uses

REX draws on the skills in [shit-1170-rex-engine-skills](https://github.com/huggiesdotnear/shit-1170-rex-engine-skills). They are installed separately, with the skills installer:

```sh
bunx skills add huggiesdotnear/shit-1170-rex-engine-skills
```

| skill | what it owns |
| --- | --- |
| `shit-1170-rex-engine` | the queue: propose, verify, queue, track, and the chain checks |
| `shit-1170-rex-tx-explorer` | an account's transaction history, and transactions by hash |
| `shit-1170-rex-ft` | NEP-141 tokens: balances, metadata, holders, transfers, wNEAR |
| `shit-1170-rex-keys` | limited access keys: generate, offer, remove, and read an account's keys |
| `shit-1170-rex-unit-conversion` | human amounts to chain units, against the token's real decimals |
| `shit-1170-rex-sputnik-dao` | DAO governance: policy, proposals, bounties, and the votes |
| `shit-1170-rex-rhea-ammv2` | the Rhea AMM: pools, quotes, swaps, LP positions, the deposited balance |
| `shit-1170-rex-rhea-dclv2` | the Rhea concentrated-liquidity DEX: pools, liquidity, limit orders |
| `shit-1170-rex-near-social` | near.social: feeds, profiles, posts, and the social writes |
| `shit-1170-rex-fastfs` | FastFS uploads: the borsh payload, the chunking, and the serving URL |
| `shit-1170-rex-hello-sleet-near` | the hello.sleet.near greeting contract |
| `shit-1170-rex-irc` | the shit-1170 IRC server: read a channel, post to it, hold the connection open |
| `shit-1170-rex-chess` | on-chain chess on app.chess-game.near, in propose mode and in autonomous mode |

This repo holds the agent itself — its identity, its rules, and its reference. The mechanics live with the skills.

### it uses the skills' tools, and only the skills' tools

Every fact REX reports comes out of a bin in the skills library: the balances from the ft skill, a transaction from the tx explorer, the access keys from the keys skill, the queue from the engine skill.

**REX does not hand-roll a chain read.** No raw RPC calls, no hand-written JSON-RPC, no curling a chain endpoint for data a skill already covers. The bins hold the fact-checking — the decimals, the storage checks, the gas, the pagination — and a hand-rolled call skips all of it.

**When a skill does not cover something, REX does not work around it.** It says the library does not support it yet, and it contributes the feature — primarily as an issue on the repo that owns the gap: the skills repo for a missing bin, a missing flag, or a wrong contract fact; the engine repo for a missing route, a queue shape, or a page. A pull request follows when the bin is straightforward.

The library is supposed to cover everything. If it does not, the gap is a bug in the library, not a reason to improvise.

### the one exception to "the agent never signs"

Autonomous chess is the only place REX signs. The chess skill can sign its own calls on `app.chess-game.near` with a limited access key — a key that names one contract and cannot move the account's NEAR, that a human approved itself, and whose method list is checked before it is used. `storage_deposit`, `add_key` and `delete_key`, and anything with a token deposit stay queued for a human in both modes.

---

## the rule every skill follows

**the agent prepares the transaction and queues it. the human signs it.** agents do not sign, send, or wait for a signature.

the engine keeps the hash of every send, so the agent reads it back from the queue it queued — the human never pastes one into a chat. the live feed's websocket tells the agent WHEN something changed; the engine's own routes say what it is.

---

## the layout

```text
agents/
  rex/
    agent.yaml           # the profile: identity, instructions, references
    references/
      ENGINE.md          # the engine's routes, data model, filters, pages
      QUEUEING.md        # payload shapes, the checks, the reporting rules
      SKILLS.md          # every skill, every bin, and what each one answers
```

REX is one agent, so this is a profile repo and not a skills repo. There is no `skills/` folder here — the skills stay in their own repo and are installed with the skills installer above.

---

## the local store

```text
${HOME}/.shit-1170-rex-engine/
├── accounts/     # one private api key file per account
├── contacts.json # the ft skill's contact list
├── queue/        # a local copy of every queued batch and swap
├── keys/         # the keys skill's store of limited access keys
├── tx/           # the tx explorer skill's store of transactions read
├── irc/          # the irc skill's config, credentials, and history
└── chess/        # the chess skill's daemon state and game history
```

set `SHIT_1170_REX_HOME` to use another directory.

---

## the model

REX does not pin a model. It inherits whatever the harness is configured with, so you can change models without reinstalling.

=====================
<br/>
copyright 2026 by sleet.near
