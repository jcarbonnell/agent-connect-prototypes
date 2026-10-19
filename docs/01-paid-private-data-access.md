# Experiment 01 — Paid private data access

**Proposer:** Julien Carbonnell (NOVA)
**Theme:** Commerce

## Details

**Build in such a way that…** a buyer agent shops a public board of encrypted trade journals, each listed on NOVA with a price, a claimed P&L ("verifiable after purchase") and the journal's SHA-256 fingerprint, and buys access on its own. Its owner signs once, in the browser: "listings 1, 2 and 3, up to $0.25". The agent ranks the board by its own rule, pays from its Agent Connect account with a `withdraw` to the NOVA contract carrying `memo: "listing:<id>"`, and NOVA admits it to the dataset only after NOVA's TEE (Shade) has checked that payment on chain. A hostile seller posts listing #99 with the best-looking P&L; the agent ranks it first, tries it, and the destination whitelist refuses it. The agent buys #1 and #2, and its third purchase hits the budget. The 60-second clip: board → one owner signature → refusal on #99 → two journals unlock and their fingerprints match → budget refusal on #3 → end card: "The agent can be fooled; the money can't."

**For an AI agent, to be able to buy private data from strangers on its own, paying only for listings its owner approved and never more than its owner allowed.**

**To handle these use cases:**
- An agent buys access to a dataset (here, a trader's journal) and verifies it received exactly what was listed, with no marketplace server, storage network or dashboard seeing the data.
- An owner pre-approves several deals at once (up to 256 destination entries) and caps total spend, then lets the agent choose among them.
- A seller cannot redirect payment to a listing the owner never signed, however attractive its teaser.

**Variants (absorbed):** Illia Polosukhin's "sell your viewing keys for copy-trading" (*"NEAR Intents confidential trading is all private and so one can have a social trading app where people sell 'keys' for only select people to see. PnL can be publicly displayed so people can choose to buy-in."*). Proof point 1 is the primitive under it, paid private data access. Iteration 2 adds the copy: a `swap` policy for the buyer, per-trade `limits` biting on an oversized copy (provider-enforced on chain), and seller trading agents committing each trade to the journal as it happens.

## Configurations (specific to Agent Connect)

**Whitelist on an agreement that is made:** one buyer agent, created off camera with a withdraw-only, USDC-only policy that can pay nobody (`destinations: only []`), a 0.12 USDC per-transaction cap, `max_actions_per_hour: 10`, and `sign.recipients: ["nova-sdk.near"]` so the agent can sign in to NOVA with its own NEAR Intents account (NEP-413 identity challenge). One grant (`paid-handoff buyer`, 7 days) lives in the partner backend. On camera, the owner signs one `policy_update` that changes only `destinations` (three entries `{ action: "withdraw", chain: "near", address: "nova-sdk.near", memo: "listing:N" }`) and `budget.daily_usd: "0.25"`. Each entry is one agreement: the seller's listing id, price and fingerprint, which the owner reviewed. Listings are 0.10 USDC each, so the budget covers two.

```ts
import { AgentApi, AgentApiError, createIdempotencyKey } from "@near-intents-agent-api/sdk";
import type { Policy } from "@near-intents-agent-api/sdk";

const USDC = "nep141:17208628f84f5d6ad33f0da3bbbeb27ffcb398eac501a31bd6ad2011e36133a1"; // 6 decimals
const NOVA = "nova-sdk.near"; // NOVA contract: escrow for listings, admits paid members

// partner backend (paid-handoff BFF): holds naa_ and the buyer's ngt_; neither reaches the browser or a model
const api = new AgentApi({
  apiKey: process.env.NEAR_INTENTS_AGENT_API_KEY!,
  baseUrl: process.env.NEAR_INTENTS_AGENT_API_URL, // https://api.demo.agentsonintents.com
});
const agentId = process.env.BUYER_AGENT_ID!;
const agent = api.forGrant(process.env.BUYER_GRANT_TOKEN!);

// Stage 1, off camera (agent_create, nep366): withdraw-only, USDC-only, pays nobody yet.
const basePolicy: Policy = {
  frozen: false,
  actions: ["withdraw"],
  confidential: false,
  owner_approval: false,
  assets: [USDC],
  limits: { per_transaction: { [USDC]: "120000" } }, // 0.12 USDC per payment
  max_actions_per_hour: 10,
  destinations: { mode: "only", list: [] },
  budget: { daily_usd: null, weekly_usd: null, monthly_usd: null },
  timelock_ms: 0,
  sign: { recipients: [NOVA] }, // on-chain field: set at creation so stage 2 stays off-chain
};

// Stage 2, on camera: the owner approves listings 1–3 and a $0.25 budget.
// Only API-enforced fields change → the API returns an off-chain nep413 intent.
const view = await api.getPolicy(agentId);
const approve = await api.generateIntent({
  type: "policy_update",
  agent_id: agentId,
  expected_revision: view.revision!,
  policy: {
    ...basePolicy,
    destinations: {
      mode: "only",
      list: [1, 2, 3].map((id) => ({ action: "withdraw" as const, chain: "near", address: NOVA, memo: `listing:${id}` })),
    },
    budget: { daily_usd: "0.25", weekly_usd: null, monthly_usd: null },
  },
});
// → browser: wallet.signMessage({ ...approve.intent.payload, nonce }) ; BFF: api.submitIntent(...) ; wait for APPLIED + synced

type Listing = { id: number; price: string; claimedPnlPct: number };
const board: Listing[] = [
  { id: 99, price: "100000", claimedPnlPct: 412 }, // hostile seller: best-looking deal, never approved
  { id: 1, price: "100000", claimedPnlPct: 38 },
  { id: 2, price: "100000", claimedPnlPct: 21 },
  { id: 3, price: "100000", claimedPnlPct: 9 },
];

// The agent's intelligence plugs in here (an LLM could rank on the public teasers).
// v1 rule: best claimed P&L per USDC first, so the fake #99 is tried first.
const ranked = [...board].sort((a, b) => b.claimedPnlPct / Number(b.price) - a.claimedPnlPct / Number(a.price));

for (const listing of ranked) {
  const body = { asset: USDC, amount: listing.price, chain: "near", recipient: NOVA, memo: `listing:${listing.id}` };
  await agent.withdraw(agentId, { ...body, dry: true }); // quote: no grant use, no budget charge, nothing moves
  const idempotencyKey = createIdempotencyKey(); // persist with the body before sending
  try {
    const op = await agent.withdraw(agentId, body, { idempotencyKey });
    let status = await api.getStatus(op.correlation_id, { waitMs: 30_000 });
    while (["PROCESSING", "QUEUED"].includes(status.status)) {
      status = await api.getStatus(op.correlation_id, { waitMs: 30_000 });
    }
    // NOVA: agent.sign(agentId, { message: challenge, recipient: NOVA }) → wallet-tier session;
    // claim_purchase(listing.id, status.details.destination_tx_hash) → Shade (TEE) checks the tx
    // on chain → settle_purchase admits the agent → retrieve, recompute SHA-256, compare to the listing.
  } catch (error) {
    if (error instanceof AgentApiError && error.code === "policy_destination_denied") continue; // #99: never approved
    if (error instanceof AgentApiError && error.code === "spend_budget_exceeded") break; // #3: over the $0.25
    throw error;
  }
}
```

**Notes / constraints:**
- `destinations` and `budget` are enforced by the API before dispatch; a change touching only them is an off-chain signature (`nep413` for a NEAR owner), no transaction (`concepts.md`, "Who enforces what"). So the on-camera approval is a wallet `signMessage`, instant, and the 0:18 refusal is an API refusal, not an on-chain one. `limits`, `assets`, `actions` are provider-enforced on chain.
- `sign` is an on-chain policy field (`policy.md`, "Signing"); setting it at creation keeps the on-camera update off-chain.
- The `only` rule matches the destination entry exactly, memo included (`executions.md`), which is what pins a payment to one listing.
- `budget` is charged at authorization with 1Click USD prices; an asset without a fresh price fails with `spend_price_unavailable` while a budget is set (`policy.md`). Dry quotes need no grant, charge no budget and move nothing (`executions.md`).
- `agent_create` for a NEAR owner is a `nep366` delegate action: the sponsor pays gas, but the `store_wallet_policy` call attaches a storage deposit paid by the owner (observed 2026-10-16: 0.01458 NEAR). Policy edits have a 5-minute cooldown; `agent_create` starts none (`policy.md`). Executions fail with `policy_not_ready` until the policy is `APPLIED` and `provider_policy_synced` (`policy.md`).
- Deployment: the partner key is valid on `https://api.demo.agentsonintents.com` (mainnet, `outlayer.near`), a separate deployment from `api.agentsonintents.com`; keys do not cross. Real funds, tiny amounts, a dedicated owner account.
- **Open question:** does the withdrawal `memo` reach the chain (USDC `ft_transfer` args or NEP-141 event)? The listing binding depends on it; fallback is binding the claim with the payer's `sign` signature over the transaction hash.
- **Open question:** which account NEAR Intents' withdrawal event names as payer, and whether it equals the `near_account_id` from the agent's identity signature.
- **Open question:** whether the exact listed amount arrives after a dry quote, or fees are deducted first (the contract checks the price).
- NOVA side (new for this experiment): the NOVA contract registers once on the USDC token; a lite listing contract (`create_listing`, used-transaction set, `settle_purchase` callable only by NOVA's verifier, `withdraw_proceeds`); Shade verifies the payment transaction inside the TEE before settling. Automatic refunds are deferred: in v1 a failed claim is refunded manually by the contract owner, and the clip does not claim "or your money back".
- Label: selling trading signals is regulated in many places; the clip says "technical demo, simulated journals, not investment advice".

## Outcomes

- The 60-second demo, **0:00** — the board: four encrypted trade journals with price and claimed P&L, "verifiable after purchase"; #99 shows the best P&L.
- **0:08** — the owner signs "listings 1–3, $0.25 budget" in the browser wallet (one off-chain signature).
- **0:18** — the agent ranks the board, tries #99 first → refused, `policy_destination_denied`.
- **0:28** — buys #1 → NOVA's TEE checks the payment on chain → access granted → the journal unlocks and its fingerprint matches the listing.
- **0:40** — buys #2 → unlocks.
- **0:48** — tries #3 → refused, `spend_budget_exceeded`.
- **0:55** — end card: "The agent can be fooled; the money can't."
- The guardrail bites twice with no human in the loop: the agent's own ranking is fooled by a fake P&L and the destination whitelist holds; then the budget stops a purchase the owner never funded.
- The thesis: *My agent bought private data from strangers on its own, and it still couldn't pay a seller I never approved, or spend more than I allowed.*
