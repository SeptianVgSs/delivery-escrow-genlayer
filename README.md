# DeliveryEscrow — GenLayer Intelligent Contract

An escrow contract written in Python for GenLayer. A buyer funds an order in
GEN, the seller marks it shipped with tracking evidence, and the funds are
released to the seller on buyer confirmation, refunded on seller cancellation,
or decided by an LLM judge reading the tracking page in `resolve()`.

**Current version: v2** (see "Revision — steward feedback, Nov 2026" below).
The seller can no longer pick an arbitrary page as the sole evidence for
releasing funds: the buyer chooses an approved carrier at order creation, the
tracking URL must come from that carrier's host and must contain the order's
tracking number, and `resolve()` refuses to release funds unless the fetched
record shows that tracking number.

## Contract

- Files: `delivery_escrow.py` (current) and `delivery_escrow_v2.py`
  (identical content; this is the filename of the deployed Studio file).
  Either file is the same 266-line contract, SHA-256
  `9351d6c1ec9b571ea133dd5c4ac4e523c4c3c56273a7d7e69b1a8aa34ff64e60`.
- Class: `DeliveryEscrow`
- Storage: orders are stored as JSON strings in a `TreeMap[u256, str]`, keyed
  by order id, so the contract schema stays small in GenLayer Studio.
- No constructor arguments.
- Approved carriers: `jne`, `sicepat`, `jnt`, `anteraja`, `pos`, `dhl`
  (hosts `jne.co.id`, `sicepat.com`, `jet.co.id`, `anteraja.id`,
  `posindonesia.co.id`, `dhl.com`).

### Methods

| Method | Type | What it does |
|---|---|---|
| `get_order(order_id)` | view | Returns the order JSON string, or `""` if none. |
| `get_order_count()` | view | Returns `next_order_id`. |
| `get_balance()` | view | Returns the contract balance. |
| `get_allowed_carriers()` | view | Returns `"jne, sicepat, jnt, anteraja, pos, dhl"`. |
| `create_order(seller, description, carrier)` | write, payable | Buyer funds an order with an approved carrier. Returns the new order id. Value must be > 0 GEN; the carrier must be in the allowlist. |
| `mark_shipped(order_id, tracking_number, tracking_url)` | write | Seller only. The URL must start with `https://`, its host must be the approved host for the order's carrier, and the URL must contain the tracking number. |
| `confirm_received(order_id)` | write | Buyer only. Releases funds to the seller. |
| `cancel_by_seller(order_id)` | write | Seller only. Refunds the buyer. |
| `resolve(order_id)` | write | Buyer or seller. A deterministic gate first checks that the fetched page contains the order's tracking number; if not, the verdict is forced `not_delivered` (refund) without any LLM judgement. Otherwise an LLM reads the page, must return the tracking number it found, and a mismatch also forces `not_delivered`. `delivered` releases funds to the seller. |

Order statuses: `FUNDED` → `SHIPPED` → `RELEASED` / `REFUNDED`.
`resolution` records `confirmed_by_buyer`, `cancelled_by_seller`, or the
`resolve()` verdict. Each order JSON also records its `carrier`.

## Revision — steward feedback, 9 Nov 2026

The Portal submission of v1 received an "Action needed" request from steward
Joaquin (9 Nov 2026): *"Please revise the escrow so the seller cannot choose
an arbitrary page as the sole evidence used to release funds. Use an
authoritative or mutually approved delivery source and verify that the fetched
record is bound to the order's tracking number. Then provide matching submitted
and deployed source for review. Respond by Nov 23, 2026."*

What changed from v1 to v2:

1. **Approved carrier chosen by the buyer at `create_order`.**
   The carrier is locked into the order. Any carrier outside the allowlist
   is rejected at creation (`"Carrier is not an approved delivery source"`),
   so no order can ever be created for an unapproved source.
2. **Host + tracking-number binding in `mark_shipped`.** The seller supplies
   only a tracking number and a URL; the contract rejects the URL unless its
   host is the approved host for the order's carrier and the URL contains the
   order's tracking number.
3. **Deterministic binding gate in `resolve()`, before any LLM judgement.**
   If the fetched page does not contain the order's tracking number, the
   verdict is forced `not_delivered` and the buyer is refunded — the LLM
   cannot release funds on an unbound record. When the LLM does run, it must
   return the tracking number it found; a mismatch also forces
   `not_delivered`.

Why the contract does not build tracking URLs per carrier: real Indonesian
carrier tracking pages are form-based with captcha/verification (JNE
`jne.co.id/id/tracking/trace`, SiCepat `sicepat.com/checkAwb`, J&T
`jet.co.id/track`), so there is no simple GET URL that returns a readable
delivery record. Inventing URL templates would only fetch empty forms. v2
therefore validates the host and the tracking-number binding instead of
inventing templates, and `resolve()` re-checks the fetched record itself.

## Studio test plan — v2 (copy-paste values)

Use two Studio accounts: A = buyer, B = seller. Record their full `0x…`
addresses — do not use truncated addresses. Deploy
`delivery_escrow_v2.py` in GenLayer Studio (Normal / Full Consensus).

1. Record the deployed address: `STUDIO_CONTRACT_ADDRESS: ________________`
2. `get_allowed_carriers()` — expect `"jne, sicepat, jnt, anteraja, pos, dhl"`.
3. Test A (negative): Account A — `create_order(seller=<full address of B>,
   description="Test carrier tidak disetujui", carrier="grab")`, Value `1`.
   Expect ERROR: `"Carrier is not an approved delivery source"`. No order
   is created. (Note: with Value `0` the amount check fires first —
   `"Must send GEN with create_order"` — so use Value `1` for this test.)
4. Test B (negative): Account A — `create_order(seller=<B>,
   description="Test JNE resi tidak ada di halaman", carrier="jne")`,
   Value `1`. Expect order id `0`, `FUNDED`.
5. Account B — `mark_shipped(0, "TEST123456",
   "https://www.jne.co.id/id/tracking/trace?noResi=TEST123456")`.
   Expect `SHIPPED`. (Host `jne.co.id` matches carrier `jne`; the URL
   contains the tracking number.)
6. Account A — `resolve(0)`. The fetched JNE form page contains no delivery
   record for `TEST123456`, so expect verdict `not_delivered`,
   status `REFUNDED`, and the 1 GEN refunded to the buyer. Do not click
   twice — AI resolution is the slowest step.
7. Test C (happy path): Account A — `create_order(seller=<B>,
   description="Test konfirmasi buyer", carrier="sicepat")`, Value `1`.
   Expect order id `1`, `FUNDED`.
8. Account B — `mark_shipped(1, "SCP987654",
   "https://www.sicepat.com/checkAwb?awb=SCP987654")`. Expect `SHIPPED`.
9. Account A — `confirm_received(1)`. Expect `RELEASED` and the 1 GEN paid
   to the seller.

Checkpoint: `get_order_count()` = `2`; order 0 `REFUNDED` with resolution
`not_delivered`, order 1 `RELEASED` with resolution `confirmed_by_buyer`.

## Test results — v2, Septian's Studio run, 11 Nov 2026 (simulator only)

Deployed and tested by Septian in his own Studio, Normal (Full Consensus),
11 Nov 2026. No deploy to Bradbury/mainnet, no external wallet / MetaMask
was connected.

- Studio contract address: `0x703aE9c18D15Ecc7F1d4D069A9ea099B98064CAB`
- Explorer: https://explorer-studio.genlayer.com/address/0x703aE9c18D15Ecc7F1d4D069A9ea099B98064CAB
- Buyer: `0xa48B463Bd4a7D34AeddbF69619BEced6009B1a6c`
- Seller: `0x7B1e4F0733039c74A658c0661F59a4126D326Cde`
- Deploy tx: `0x276fa03ee683fadf85d202b641b6ba979585cd4432611f8ca1ec483032b24f39`, FINALIZED, SUCCESS (3 of 5 validators SUCCESS/Agree; 2 ERROR "Validator execution cancelled after quorum", normal once quorum is reached)
- Test 0 — `get_allowed_carriers()`: `"jne, sicepat, jnt, anteraja, pos, dhl"`.
- Test A — carrier `grab`, Value `1`: tx `0xe23a3c76b0a717a350d32ded53bc4cedb23cbcce30b3b87cbaf5661c7dbddd1`, FINALIZED, Result ERROR, `[rollback] Carrier is not an approved delivery source` (3 validators Agree). Earlier Value-`0` attempt: tx `0x0893f03a45ac507c120e0c108e1cfb1f2fa21451b1c2a304e2e370794f03518a`, FINALIZED, ERROR, `[rollback] Must send GEN with create_order`. `get_order_count()` after Test A: `0` — no order created.
- Test B — order 0: create tx `0x2a113b420fc8f64edf5be888e8917cddf4f66bdad16e42f6d04ab994730c2af1`, ACCEPTED, SUCCESS, Output `0`; `get_order(0)` showed `FUNDED`, amount `1000000000000000000`, carrier `jne`. `mark_shipped` tx `0x4397113a7d2f305719f29f2c779ed890fb52651395e423bdd0562d89ded3ba0d`, ACCEPTED, SUCCESS. `resolve` tx `0x6eaf71643ad8760749b36842bbad47745c004e9cd7531a2b69eb1e6b8d91e62d`, ACCEPTED, SUCCESS, Output `"not_delivered"` (3 validators SUCCESS/Agree; the equivalence reasoning notes the fetched evidence is only HTML source of a tracking page). `get_order(0)` final: status `REFUNDED`, resolution `not_delivered`, tracking_number `TEST123456`, tracking_url `https://www.jne.co.id/id/tracking/trace?noResi=TEST123456`. Refund verified: buyer balance 8 GEN → 9 GEN after resolve.
- Test C — order 1: create tx `0x31833e7d872aca7695ab3602bc7a3e470f1ede5943f4ac93d8344fa5e6d40d4c`, ACCEPTED, SUCCESS, Output `1`. `mark_shipped` tx `0x27e575a4bc44e12a06e2af72803e9e60ac9d971ad08a3481757c68e929dc03e9`, ACCEPTED, SUCCESS (5 of 5 validators SUCCESS/Agree). `confirm_received` tx `0x8b0eb010fa4bf0b152606b282df378202cf92992e3db2163d92199b76079ecea`, FINALIZED, SUCCESS (5 of 5 validators SUCCESS/Agree). `get_order(1)` final: status `RELEASED`, resolution `confirmed_by_buyer`, carrier `sicepat`, tracking_number `SCP987654`, tracking_url `https://www.sicepat.com/checkAwb?awb=SCP987654`.
- Final tallies: `get_order_count()` = `2`; `get_balance()` = `1000000000000000000` (1 GEN); buyer 8 GEN, seller 11 GEN, contract 1 GEN — total 20 GEN, matching the two 10 GEN starting balances.

Note on the contract's residual 1 GEN: it is the Value of the reverted
Test A `grab` attempt, retained by the Studio simulator. On a real EVM a
reverted transaction returns `msg.value`; the simulator keeps it in the
contract instead. It is not order funds — both orders' funds left the
contract (order 0 refunded to the buyer, order 1 released to the seller).
The Studio account-header balance can also lag behind the account menu;
the menu values above are the verified ones.

## Test results — v1, completed Studio run, 27 Oct 2026 (simulator only)

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

All transactions above were FINALIZED.

Note: one duplicate empty `delivery_escrow.py` editor tab was left in that
Studio session; it does not affect the deployed contract above.

## Test results — v1, Septian's Studio run, 27 Oct 2026 (simulator only)

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
- Order 1 — create `0x9a4daefaba5344ee9b3322bb1a19ea2dbdb8c7a7aac160b5077dbbd27947a6ca`, mark_shipped `0x2355efc133f878b4044fb4cd1d5f21f667d21cb0d0caf7a13f00b2fe833e9ac9`, cancel_by_seller `0x6376cf044985cb41596be00fe81d081e9afed3303279a2acf0da3bbeb539d91a`, payout `0x9dad52bd25e5173ebc1464cd2501b6643ca46ebb387efcc6cd754a7ef07eca26` to the buyer (refund).
- Order 2 — create `0x560b3600c3749aa90dd73b15f2bdf8d28aed027fa5cb0ca440e0522ca31719`, mark_shipped `0xdd691c35939b0b3fa412f233d08cc6d95155082c4727bf73431f7bd9a837fac2`, resolve `0x16da654c883d3f02bc17377cff1258c5dffe7ddc103b6b5d16e8eb016ce97e3f`, payout Send `0x265710259fae7e71c7173622ddaeea113a49ea004eb3cf880c23060e6556df8a` to the buyer (refund, consistent with a `not_delivered` verdict).

All 13 transactions above were FINALIZED with GenVM SUCCESS / Consensus
Accepted in the explorer. The order outcomes above follow from the method
sequence and payout direction in the explorer; the exact
`get_order(0)`, `get_order(1)` and `get_order(2)` JSON (status, resolution
and amount) for this instance were not confirmed — do not present the
inferred statuses as confirmed.

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
- Pasting code into the Studio editor on a phone can silently corrupt
  indentation (seen 11 Nov 2026: `IndentationError: unexpected indent` at
  `class Write:` from `gen_getContractSchemaForCode`). If the schema fails
  to load for a file that compiles locally, upload the file into Studio
  instead of pasting it.
- In Studio, EOA balance movement via `emit_transfer` is simulated; order
  status and FINALIZED transactions are the main evidence there. Real GEN
  movement needs checking on Bradbury. A reverted payable transaction's
  Value can stay in the contract in the simulator (seen 11 Nov 2026), and
  the account-header balance can lag behind the account menu.

## Next: Bradbury / portal submission

Status on 11 Nov 2026: the v1 Intelligent Contracts submission received an
"Action needed" request from steward Joaquin (9 Nov 2026; respond by
23 Nov 2026). v2 above is the revision; it was deployed and tested in
Studio (simulator only) at
`0x703aE9c18D15Ecc7F1d4D069A9ea099B98064CAB`. No deploy to Bradbury/mainnet
and no external wallet / MetaMask was connected. This repo is the evidence
for the resubmission: the contract (v2, identical in `delivery_escrow.py`
and `delivery_escrow_v2.py`), this README test plan and results, and the
Studio explorer link above. Before spending time on Bradbury, check the
current status directly on https://portal.genlayer.foundation/ — do not
trust "claim is live" posts; use official pages only, and never share a
seed phrase or private key.

Useful references (verify on the day, docs change):

- GenLayer docs, Networks / recommended flow (Studionet → Localnet →
  Bradbury): https://docs.genlayer.com/developers/networks
- GenLayer Portal: https://portal.genlayer.foundation/
- Faucet reported in secondary guides: https://testnet-faucet.genlayer.foundation/
  (reported requirements: GitHub account > 3 months old, ≥ 1 public repo,
  100 GEN / 24 h — verify on the faucet page itself.)

## Safety

No seed phrases or private keys are in this repo. Test accounts only.
