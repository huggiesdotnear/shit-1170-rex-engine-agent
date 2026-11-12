# shit-1170-rex-engine-agent

this repo holds **REX**, the agent profile for the shit-1170 rex engine.

- see the skill list in [README.md](./README.md)
- see the live site at 🌐 https://shit-1170.com/
- see the engine's code at 🐱 https://github.com/huggiesdotnear/shit-1170-rex-engine
- see the skills it draws on at 🐱 https://github.com/huggiesdotnear/shit-1170-rex-engine-skills

---

#### GIT

`git pull` before you start, commit and push to `main` when you are done.

never create a branch. work on `main` and push it.

---

#### WHAT THIS REPO IS

one agent, packaged as an [awesome-agents](https://github.com/pablof7z/awesome-agents)
profile. the profile lives in `agents/rex/`:

- `agent.yaml` — the profile: id, name, description, and the instructions
  that make REX behave the way it does.
- `references/ENGINE.md` — the engine's routes, data model, filters, pages.
- `references/QUEUEING.md` — the queue payload shapes, the checks, and the
  reporting rules.
- `references/SKILLS.md` — every skill, every bin, and what each one
  answers.

the mechanics — the bins, the typed client, the contract knowledge — live
in the skills repo, not here. this repo is the agent's identity and its
operating rules. do not copy skill code in here.

---

#### THE RULES THAT MUST STAY TRUE

these are the ones worth breaking the profile over.

**the agent never calls a contract.** every state-changing call becomes a
queued payload — `{ contractId, method, args, gas, deposit }` — and a human
signs it in the web app. the agent prepares and queues. it does not sign,
send, or wait for a signature.

**autonomous chess is the one exception.** the chess skill may sign its own
calls on `app.chess-game.near` with a limited access key. it is bounded: one
contract, no ability to move the account's NEAR, the human approved the
`add_key`, and the method list is checked before the key is used.
`storage_deposit`, `add_key` and `delete_key`, and anything with a token
deposit stay queued for a human in both modes.

**the hash comes from the engine.** the engine writes the send's hash on the
account row, so the agent reads it back from the queue it queued. no step
ever asks the human to paste a hash.

**a queue id is not a receipt.** it means the engine is holding the proposal
for a human to sign. never describe a queued batch as sent or executed.

**the queue id and the signing link go together.** the id is what the human
quotes back; the link is where they sign it. always both.

**verify before queueing.** the engine records whatever it is given. it does
not check the economics of a call. a wrong method fails loudly; a wrong
amount succeeds and moves the wrong thing.

**the chain is the truth.** the engine's status is a record of what it was
told. when the engine and the chain disagree, the chain is right.

**the skills' bins are the tools.** every fact the agent reports comes out of
a bin in the skills repo. no hand-rolled RPC calls, no hand-written
JSON-RPC, and no curling a third-party api for data a skill already covers.
the bins hold the fact-checking — the decimals, the storage checks, the gas,
the pagination, the caching — and a hand-rolled call skips all of it.

**a missing feature is an issue, not a workaround.** when a skill does not
cover something, the agent says the library does not support it yet and
contributes the feature — primarily as an issue on the repo that owns the
gap: the skills repo for a missing bin, a missing flag, or a wrong contract
fact; the engine repo for a missing route, a queue shape, or a page. the
issue says what was asked for, which skill should own it and which bin it
would be, the flags it would take and the shape of the answer it would
print, and the chain or api call it would sit on. a pull request follows
when the bin is straightforward. the library is supposed to cover
everything, so a gap is a bug in the library.

---

#### WHERE A FACT COMES FROM

the engine's typed client is the source of truth for the engine's routes and
payload shapes — `@huggies/shit-1170-rex-engine-middleman`. one function per
route, one type per request and response shape.

**a skill's `SKILL.md` is the source of truth for its bins.** it holds the
flags, the answer shape, the edge cases, and the contract facts. read the
matching one before running a bin that is new to you. `SKILLS.md` in this
repo is the index — every skill, every bin, and what each one answers.

for a contract's method, gas, and deposit, the evidence is ranked:

| source | what it is | how strong |
| --- | --- | --- |
| a **fun helper** in the TS or RUST tool box | the function that builds and sends the exact JSON | strongest — it proves the argument shape, the gas, and the deposit |
| a **typed contract** in the RUST tool box | a hand-written wrapper | weaker — it has been wrong once, so check it before use |
| the **skill library** | the written reference per contract | the reference to write against; the tool boxes win on a disagreement |
| `near contract inspect` | the deployed build's own ABI | authoritative for the method list |

a fun helper proves the **shape** of the arguments, not that the method
exists. near-sdk only exports methods inside a `#[near]` impl block — a
plain `pub fn` on the contract is not callable.

`near contract inspect` reports the build that account runs, not the newest
build, so check each account you intend to call.

when a fact is in none of them, record it as **not documented yet** rather
than guessing.

---

#### THE PROFILE FORMAT

the loader is `awesome-agents`. the fields this profile uses:

- `schema`, `id`, `name`, `description` — the identity.
- `model: inherit` — REX does not pin a model. it takes whatever the harness
  is configured with, so models can change without a reinstall.
- `instructions` — the operating manual, as a block scalar.
- `references/` — installed to `~/.agents/homes/rex/references` and listed
  in the installed prompt, so the profile refers to them by relative path.

`description` has a **60-word limit** and the install throws if you exceed
it. the longer model card lives in the repo `README.md`, which is the
sibling of `agents/` and not part of the profile.

there is no `adapters/` folder. with no model pinned, there is nothing an
adapter would override. there is no `skills/` folder either — the skills
live in their own repo and are installed with the skills installer.

---

#### CHECKING A CHANGE

the CLI resolves profiles from the source layout, so a dry run against the
checkout is the real check:

```sh
bunx awesome-agents add . --agent rex --list
bunx awesome-agents add . --agent rex --harness opencode --dry-run --json
```

a real install, to see the rendered prompt:

```sh
bunx awesome-agents add . --agent rex --harness opencode
cat ~/.config/opencode/agents/rex.md
```

then remove it again:

```sh
bunx awesome-agents remove rex
```

---

#### DOCS

- [ENGINE.md](./agents/rex/references/ENGINE.md) — the engine's routes, data model, and pages
- [QUEUEING.md](./agents/rex/references/QUEUEING.md) — the payload shapes, the checks, the reporting rules
- [SKILLS.md](./agents/rex/references/SKILLS.md) — every skill, every bin, and what each one answers
- [the skills repo](https://github.com/huggiesdotnear/shit-1170-rex-engine-skills/blob/main/DOCS/TOOLS.md) — where the facts come from

=====================
<br/>
copyright 2026 by sleet.near
