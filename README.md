# DeliveryEscrow — GenLayer Intelligent Contract

An escrow contract written in Python for GenLayer. A buyer funds an order in
GEN, the seller marks it shipped with tracking evidence, and the funds are
released to the seller on buyer confirmation, refunded on seller cancellation,
or decided by an LLM judge reading the tracking page in `resolve()`.

## Contract

- File: `delivery_escrow.py`
- Class: `DeliveryEscrow`
- Storage: orders are stored as JSON strings in a `TreeMap[u256, str]`, keyed
  by order id, so the contract schema stays small in GenLayer Studio.
- No constructor arguments.

### Methods

| Method | Type | What it does |
|---|---|---|
| `get_order(order_id)` | view | Returns the order JSON string, or `""` if none. |
| `get_order_count()` | view | Returns `next_order_id`. |
| `get_balance()` | view | Returns the contract balance. |
| `create_order(seller, description)` | write, payable | Buyer funds an order. Returns the new order id. Value must be > 0 GEN. |
| `mark_shipped(order_id, tracking_number, tracking_url)` | write | Seller only. `tracking_url` must start with `https://`. |
| `confirm_received(order_id)` | write | Buyer only. Releases funds to the seller. |
| `cancel_by_seller(order_id)` | write | Seller only. Refunds the buyer. |
| `resolve(order_id)` | write | Buyer or seller. LLM reads the tracking page and returns `delivered` (release) or `not_delivered` (refund). |

Order statuses: `FUNDED` → `SHIPPED` → `RELEASED` / `REFUNDED`.
`resolution` records `confirmed_by_buyer`, `cancelled_by_seller`, or the
`resolve()` verdict.

## Studio test plan (copy-paste values)

Use two Studio accounts: A = buyer, B = seller. Record their full `0x…`
addresses — do not use truncated addresses.

1. Deploy `delivery_escrow.py` in GenLayer Studio (Normal / Full Consensus).
   Record the deployed address: `STUDIO_CONTRACT_ADDRESS: ________________`
2. Account A — `create_order(seller=<full address of B>, description="Paket contoh - test escrow 1")`, Value `1` — the Studio Value field is in GEN, not wei. Entering `1000000000000000000` there will fail for insufficient balance. The contract still records `1000000000000000000` wei (1 GEN = 10^18 wei). Expect order id `0`.
3. `get_order(0)` — expect `FUNDED`, amount `1000000000000000000`.
4. Account B — `mark_shipped(0, "RESI123", "https://example.com/")`. Expect `SHIPPED`.
5. Account A — `confirm_received(0)`. Expect `RELEASED`.
6. Repeat for order `1`: create → mark shipped → `cancel_by_seller(1)`. Expect `REFUNDED`.
7. Repeat for order `2`: create → mark shipped → `resolve(2)`. With
   `https://example.com/` there is no delivery evidence, so the expected
   result is `not_delivered` and `REFUNDED`. Do not click twice — AI
   resolution is the slowest step.

Checkpoint: `get_order_count()` = `3`; order 0 `RELEASED`, order 1
`REFUNDED`, order 2 `REFUNDED` with resolution `not_delivered`.

## Test results — completed Studio run, 27 Oct 2026 (simulator only)

Completed by Aceng in the Muse browser's GenLayer Studio simulator, Normal
(Full Consensus). Septian's own Studio account (`0xE0...9ae6`, in his own
browser) could not be used. No deploy to Bradbury/mainnet, no portal
submission, and no external wallet / MetaMask was connected.

- Studio contract address: `0x69B9Ac5e6ae96455F235c433C76745C93b36D308`
- Explorer: https://explorer-studio.genlayer.com/address/0x69B9Ac5e6ae96455F235c433C76745C93b36D308
- Account A (buyer): `0xE5bB6294806c527C50779c24187AC8EC915Ba7CF`
- Account B (seller): `0xfD9D2BEFF98abD57086610DADe6AA62c8EbFbbb8`
- Deploy tx: `0x354121695e7d7ed40f189f04d7ef550c51ffcd364170754eccb9f87cb41bcba2`, FINALIZED
- `get_order_count()`: `3`; `get_balance()`: `0`
- `get_order(0)`: `RELEASED`, resolution `confirmed_by_buyer` — create `0x60115b3175ff6a96c5d540e543c0efdac358f3a02761134c279db10306a96204`, mark_shipped `0x778ac3a13d8df1e40b7b9e0ff19642c9d977304a09899039f0c8b690074b3272`, confirm_received `0x6fbd680b34c55b075c005182a8ffaacb34453719131a2a7bb9bf457863e9f840`
- `get_order(1)`: `REFUNDED`, resolution `cancelled_by_seller` — create `0xe9ab1f49967d1a17eaa7cf722e0034afad4c5304a57691499261ff3f93b1d9ec`, mark_shipped `0xdacdafe01f3e9bfbc5f26c4a693e2eb5867b0a39e3f76f21e10ceee1cb3c3e20`, cancel_by_seller `0xb63be7b7ac43b7315cecce9d7c0eb1c06884d8923b97b043fe73df63016e7328`
- `get_order(2)`: `REFUNDED`, verdict `not_delivered` — create `0x96461f94bf8d9fb1a7553bfa35e9782a94600b7f580c529f9352e343d00d4b95`, mark_shipped `0x02944c0d9c45cbb60fd710ef6f14368f4c53b67c50a95d22aa118b693f6bd271`, resolve (once) `0x0ec54a2a7e647358f143c63d60a5d257980632254c8f7a2deb215b5310d6eff6`

All transactions above were FINALIZED. Screenshots: attach Studio
screenshots showing FINALIZED transactions if submitting to the portal.

Note: one duplicate empty `delivery_escrow.py` editor tab was left in that
Studio session; it does not affect the deployed contract above.

## Test results — Septian's Studio run, 27 Oct 2026 (simulator only)

Fresh instance deployed by Septian in his own Studio, Normal
(Full Consensus), created 27 Oct 2026, 17:07:52 WIB (10:07:52 UTC in the
explorer). No deploy to Bradbury/mainnet, no portal submission yet, and no
external wallet / MetaMask was connected. Buyer
`0x234Ee0EE51C9a1234f7215324deA297361a9752e`, seller (Septian)
`0xE07BaaD851916b793bEE5735F99E0bC817A09ae6`.

- Studio contract address: `0x2626F77d355ca59b62d47162190af1e3c395322A`
- Explorer: https://explorer-studio.genlayer.com/address/0x2626F77d355ca59b62d47162190af1e3c395322A
- Explorer shows: Balance `0 GEN`, Transactions `13`, all FINALIZED.
- Deploy tx: `0xbb85a5b18d910aa518157c1d659bf7f84f7bbb248328ab26ddd11359c1038d6e`, FINALIZED (creator: buyer).
- Order 0 — create `0xa86199cb2a845b0ac50ba1ee30dd707d2666b11e57ca1a699e8bd9c088938550`, mark_shipped `0xd687b21bbf7b77ce0a2026b10da40191788a88412395def674b10720d6d0d714`, confirm_received `0x4886327ebfb56658f7ad77ba3b7f7538465f23a32530bc0843a3c3fa576f2f59`, payout Send `0xd24f49039a067ad53497d6af170205a789b60c165ec4a3c7a83e1413a6e40f9a` to the seller (release).
- Order 1 — create `0x9a4daefaba5344ee9b3322bb1a19ea2dbdb8c7a7aac160b5077dbbd27947a6ca`, mark_shipped `0x2355efc133f878b4044fb4cd1d5f21f667d21cb0d0caf7a13f00b2fe833e9ac9`, cancel_by_seller `0x6376cf044985cb41596be00fe81d081e9afed3303279a2acf0da3bbeb539d91a`, payout Send `0x9dad52bd25e5173ebc1464cd2501b6643ca46ebb387efcc6cd754a7ef07eca26` to the buyer (refund).
- Order 2 — create `0x560b3600c3749aa90dd73b15f2bdf8d28aed027fa5e8cb0ca440e0522ca31719`, mark_shipped `0xdd691c35939b0b3fa412f233d08cc6d95155082c4727bf73431f7bd9a837fac2`, resolve `0x16da654c883d3f02bc17377cff1258c5dffe7ddc103b6b5d16e8eb016ce97e3f`, payout Send `0x265710259fae7e71c7173622ddaeea113a49ea004eb3cf880c23060e6556df8a` to the buyer (refund, consistent with a `not_delivered` verdict).

All 13 transactions above were FINALIZED with GenVM SUCCESS / Consensus
Accepted in the explorer. The order outcomes above follow from the method
sequence and payout direction in the explorer; confirm the exact
`get_order(0)`, `get_order(1)` and `get_order(2)` JSON (status, resolution
and amount) in Studio before submitting, and record them here if they
differ from the expected RELEASED / REFUNDED / REFUNDED (`not_delivered`).

An earlier instance in the same Studio session
(`0xe1...1954`, order 0 amount 2 GEN, RELEASED) had an unexplained extra
2 GEN in its balance and was abandoned in favour of this clean instance;
do not use that instance as submission evidence.

## Known Studio behaviour

- An earlier deploy of this contract finalised all transactions
  (Deploy / `create_order` / `resolve` / `mark_shipped`) but Studio showed
  “Error getting the contract state” / `gen_getContractSchemaForCode`
  execution failed. If that recurs, do not redeploy immediately: check
  `get_order_count()` first — if it returns a value, the contract is live
  and the error is in the Studio schema/state UI.
- In the 27 Oct 2026 Studio run, the `gen_getContractSchemaForCode`
  schema error appeared while the editor tab was still empty, before
  deployment. After the coded file loaded, the schema read normally and
  the "Error getting the contract state" error did not reappear after
  deployment.
- In Studio, EOA balance movement via `emit_transfer` is simulated; order
  status and FINALIZED transactions are the main evidence there. Real GEN
  movement needs checking on Bradbury.

## Next: Bradbury / portal submission

Status on 27 Oct 2026: Studio rebuild delivered (results above, simulator
only). No deploy to Bradbury/mainnet and no portal submission had been made
yet. The GenLayer Portal screenshot on 27 Oct 2026 showed "Submit
Contribution" for Builder / Intelligent Contracts (0–500 pts), with
"Season 2 is underway" and 0 of 2 weekly slots used. This repo is the
evidence for that submission: the contract, this README test plan and
results, and the Studio explorer link above. Before spending time on
Bradbury, check the current status directly on
https://portal.genlayer.foundation/ — do not trust "claim is live" posts;
use official pages only, and never share a seed phrase or private key.

Useful references (verify on the day, docs change):

- GenLayer docs, Networks / recommended flow (Studionet → Localnet →
  Bradbury): https://docs.genlayer.com/developers/networks
- GenLayer Portal: https://portal.genlayer.foundation/
- Faucet reported in secondary guides: https://testnet-faucet.genlayer.foundation/
  (reported requirements: GitHub account > 3 months old, ≥ 1 public repo,
  100 GEN / 24 h — verify on the faucet page itself.)

## Safety

No seed phrases or private keys are in this repo. Test accounts only.
