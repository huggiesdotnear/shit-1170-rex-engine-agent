# QUEUEING

the queue payload shapes, the checks, and the reporting rules.

this is the reference Rex reads when it prepares work for a human to
sign.

---

#### THE SHAPE OF A WRITE

a write is DATA, not an action. the exact shape the engine's tx-queue
accepts:

```json
{
  "contractId": "usdt.tether-token.near",
  "method": "ft_transfer",
  "args": { "receiver_id": "bob.near", "amount": "25000000" },
  "gas": "30000000000000",
  "deposit": "1"
}
```

- `contractId` — the contract account id.
- `method` — the method name, exactly as the contract spells it.
- `args` — the contract's own argument object, not a string.
- `gas` — the gas, as a string.
- `deposit` — the attached native NEAR, as a string.

**some contracts take their arguments as raw bytes, not as an object.**
such a call carries `argsBase64` — the standard base64 of the raw bytes —
*instead of* `args`:

```json
{
  "contractId": "fastfs.near",
  "method": "__fastdata_fastfs",
  "argsBase64": "AAwAAABmYXN0bmVhci5wbmc=",
  "gas": "1",
  "deposit": "0"
}
```

a transaction sets **exactly one** of `args` and `argsBase64`. both at
once is a `400` (`args_and_args_base64`); a base64 that does not decode
is a `400` (`args_base64_not_base64`); neither is an empty `args`, which
is what a call with no arguments has always been.

use `argsBase64` only when the contract's arguments really are bytes. the
FastFS protocol (`__fastdata_fastfs`) is the one that needs them.

`gas` and `deposit` are raw digit strings here: gas units and
yoctoNEAR. the web app's `shared/gas_deposit.ts` also accepts
near-kit's human forms — `"30 Tgas"`, `"0.00125 NEAR"`, `"1 yocto"` —
and converts them, so a hand-written payload may use either. raw digits
are preferred, because there is no ambiguity about what they mean.

the middleman's `queue_call_fun` builds a SINGLE call with sane defaults
(gas `30000000000000`, deposit `0`) and refuses a payload it cannot
trust: an empty `contractId` or `method`, an `args` that is not a plain
object, an `args_base64` that is not base64, a payload carrying both
kinds of args, or a `gas`/`deposit` that is not a raw digit string.

---

#### A BATCH OF CALLS

a queue is a batch of one to fifty calls, in order, signed together.

```json
{
  "what": "Register storage on the token so the USDC can arrive",
  "when": "Before the USDC transfer",
  "why": "The receiving account has no NEP-145 storage on this token",
  "transactions": [
    {
      "contractId": "usdt.tether-token.near",
      "method": "storage_deposit",
      "args": {},
      "gas": "30000000000000",
      "deposit": "1250000000000000000000"
    }
  ],
  "accounts": ["alice.near"]
}
```

the `accounts` list says whose feed receives the batch. every account in
it must be an account the ENGINE has issued an API key for.

the list may be omitted, and then it defaults to the signing account —
the account the stored API key belongs to. an empty list would write zero
account rows server-side, so the batch would land in nobody's feed.

`deposit` is the amount of native NEAR attached. a `storage_deposit`
often needs a real deposit, not `0`.

`whenAt` sets the queue's earliest execution time, in epoch millis. it is
a number — a typo is refused rather than silently queued with no time.

**the local copy holds the request as the agent wrote it and the response
as the server returned it**, so it is a record of what was proposed, what
was queued, and what happened next.

---

#### A SWAP

a queued swap is not a route. it names the token sold, the raw amount
sold, and the token bought. the route is worked out when a human
executes it.

```json
{
  "tokenIn": "wrap.near",
  "amountIn": "1500000000000000000000000",
  "tokenOut": "usdt.tether-token.near",
  "amountOut": "1250000",
  "dexId": "rhea",
  "what": "Buy USDC with wNEAR",
  "when": "Now",
  "why": "The user asked for it",
  "whenAt": 1767225600000,
  "accounts": ["alice.near"]
}
```

`amountOut` and `dexId` are optional — a swap may be queued without a
quote, and the live quote is worked out when it is shown. `dexId`
records where a stored quote came from.

the token ids are the intear registry's ids — every token is its bare
contract id, except native NEAR, which is `near`.

`amountIn` is the token's smallest unit. when the human gave a decimal
amount, convert it against the token's real decimals first.

---

#### THE CHECKS

the engine does not validate the economics of a contract call. it
records the call. verifying it is the agent's job.

| check | how | what goes wrong without it |
| --- | --- | --- |
| contract exists | inspect the account on chain | the call goes to a stranger's account |
| method exists | the contract's own method list | `MethodNotFound` at signing time |
| argument shape | the contract's ABI or a typed helper | a missing field fails; a wrong type may not |
| gas | the contract skill or a tested helper | the first leg runs and the second does not |
| deposit | the contract skill | `storage_deposit` fails or a token transfer is rejected |
| receiver exists | inspect the account on chain | tokens go nowhere |
| receiver has token storage | `storage_balance_of` | the transfer succeeds and the tokens vanish |
| amount | the token's real decimals | a million times too much, or dust |
| the account holds the input token | read the balances | the swap is queued against money that is not there |
| intent | the human | the right call doing the wrong thing |

the last one is the one no tool can do. a technically correct call can
still be the wrong thing to do.

**probing a method is not the same as reading it.** a probe calls each
method with EMPTY arguments, so a method that needs arguments fails.
that failure does NOT mean the method is missing: it reports `exists:
true` with `reason: "call_failed"`. only `reason: "method_not_found"`
means the contract has no such method.

**the check order matters.** verify the contract and the method before
the amount — a wrong method fails loudly and costs nothing, while a
wrong amount usually succeeds and moves the wrong thing.

---

#### AFTER THE HUMAN SENDS

the whole sequence, in order:

1. the human signs in the web app. their wallet tells the engine, and
   the engine writes the hash on the account row.
2. watch the queue until it is sent — the agent waits instead of asking.
3. read the hashes off the queue's account rows. the human never supplies
   one.
4. read the receipt on chain, which is the real outcome.
5. sync the local copy with the server.

steps 3 and 4 answer two different questions. step 3 asks the engine
what it recorded. step 4 asks the chain what happened.

a hash the engine does not have yet means the batch has not been sent,
or the send was not reported to the engine — say so rather than guessing.

**the engine's status and the chain's status are two different things.**
the engine says `succeeded` when it was told the receipt worked. when
they disagree, the chain is right — the engine is a record of what it
was told.

the transactions api is read-only JSON over transactions, blocks, and
receipts. `https://tx.main.fastnear.com/` for mainnet and
`https://tx.test.fastnear.com/` for testnet. a hash the api does not
know is an error, not a silent success.

---

#### WATCHING THE FEED

the feed is a websocket. the server pushes every new queued batch and
every new queued swap to every connected client.

**the watch is bounded**, because every other call runs once and exits
and an agent cannot wait forever for a human. the deadline is set
explicitly, and the watch ends early when the work reaches the state
you are waiting for: `any` (the default — the agent only wants to know
something happened), `sent`, `succeeded`, or `failed`.

the filters are the list route's filters, with the same names and the
same meaning: `account_id`, `contract_id`, `method`, `token_in`,
`token_out`, and `hide_sent`. the socket itself is filter-agnostic — the
server carries everything and the client decides what matches — so one
filter vocabulary covers both reading and watching.

**the socket is the trigger; the server is the truth.** the socket says
THAT something changed. the engine's routes say WHAT it is. so the watch
refreshes the local record on every event, and the local copy is current
when the bin exits.

**a socket connects before it subscribes.** a subscriber only sees what
is published AFTER it subscribes, so a queue that was queued in the
moment before the watch opened is not in the watch. give the socket a
moment to open before you expect the event — or read the queue's own
record, which has the state either way.

**a watch is for waiting, not for polling.** watching and syncing
overlap on purpose: the watch tells you WHEN to refresh, and it does the
refresh. use the sync when you want to check an old queue without
waiting for anything.

---

#### THE LOCAL COPY

every queued item is written under:

```text
${HOME}/.shit-1170-rex-engine/
├── accounts/
│   └── <account_id>.json    # one private api key file per account
├── contacts.json            # the ft skill's contact list
└── queue/
    ├── index.json           # the list of every local record
    ├── tx/<queue_id>.json   # one local copy per queued batch
    └── swap/<swap_id>.json  # one local copy per queued swap
```

set `SHIT_1170_REX_HOME` to use a different directory.

the local copy survives a database outage. the local records are also
how an agent learns: a queue that failed again and again has a method
shape or a gas value that is wrong, and the records make that visible
without a database.

---

#### API KEYS

one key per account, in its own file with `0600` permissions. the list
command never prints a key. `whoami` asks the server which account the
key belongs to — use it when a key's account is uncertain.

when more than one account is stored, every call that queues work needs
`--account <id>`. without it the call refuses rather than guessing. a
batch queued for the wrong account lands in the wrong feed.

one file per account rather than one file with all of them: a file for
one account can be shared, moved, or deleted without touching the
others, a rotation replaces one file, a leaked file leaks one account
rather than every account, and the list can be shown without ever
showing a key.

the key must be one the ENGINE issued — get it from the site's
`/app/skill.html` page. the local keyring supplies the signing key; the
server checks its own `api_keys` table, so a key the server never issued
answers 400 `account_not_named_by_agent`. the signing account also comes
from the API key, so the key and the account must agree.

---

#### REPORTING

after queueing, tell the human in plain language:

- what the batch does, in one sentence
- every call: contract, method, and the arguments that matter
- the gas and attached deposit for each call
- what the account must approve or own for it to succeed
- the queue id and the link to sign it
- the local copy path
- that it is waiting for their signature, and that they can send it later

**the queue id and the link go together.** the id is what the human
quotes back to you; the link is where they sign it.

do not describe a queued batch as sent, submitted, or executed.

---

#### WHERE THE FACTS COME FROM

| source | what it is | how strong |
| --- | --- | --- |
| a **fun helper** in the TS or RUST tool box | the function that builds and sends the exact JSON | strongest — it proves the argument shape, the gas, and the deposit |
| a **typed contract** in the RUST tool box | a hand-written wrapper | weaker — it has been wrong once, so check it before use |
| the **skill library** | the written reference per contract | the reference to write against; the tool boxes win on a disagreement |
| `near contract inspect` | the deployed build's own ABI | authoritative for the method list |

`near contract inspect` reports the build that account runs, not the
newest build, so check each account you intend to call. daos deployed
from the same factory can sit on different builds.

a fun helper proves the **shape** of the arguments, not that the method
exists. near-sdk only exports methods inside a `#[near]` impl block — a
plain `pub fn` on the contract is not callable.

when a fact is in none of them, record it as **not documented yet** rather
than guessing.

=====================
<br/>
copyright 2026 by sleet.near
