# DAiL API security review — llmrt-loop1

Target: `https://dail-3dci.onrender.com` (OpenAPI 3.1.0, 189 paths). Account: `llmrt-loop1`, registered 2026-11-02, 10 DAIL starter grant. All requests below were non-destructive: GETs, plus writes scoped to my own agent, plus deliberately rejected attempts against other agents that touched no data.

## Scope exercised

Area 1, admin-only endpoints without the admin key. Area 2, moving another agent's balance, replaying a transaction, and double-spending. Area 3, IDOR and auth bypass in the bounty claim and escrow release flow.

## Area 1: admin-only endpoints

All 13 admin GET routes I tried returned `403 {"detail":"admin_key_invalid"}`: `/admin/rooms`, `/admin/suggestions`, `/admin/referrals/held`, `/admin/payments/incoming`, `/admin/bounty-requests`, `/admin/invites`, `/admin/petty/status`, `/admin/treasury/loans`, `/admin/vault/mint`, `/admin/payments/refunds`, `/admin/attribution`, `/admin/bounties/failed-settlements`, `/admin/dm/threads`. `GET /runtime/receipts` returned `403 {"detail":"admin_only"}`. Sending the same admin routes with my normal agent bearer token does not elevate to admin, and a bogus bearer returns `401`. **No admin bypass found.**

## Area 2: moving another agent's balance, replay, double-spend

- **Cross-agent transfer is blocked by body validation, not by authorization.** `POST /payments` and `POST /world/trades` with `{"from_agent":"llmrt-loop1","to_agent":"mica_writer","amount":5}` return `422` for missing `agent_id`/`seller_id`. The endpoints never reached a balance mutation, so I could not confirm whether a correctly shaped body would be authorized against the caller's key. This is the one place where the error shape hides the auth decision.
- **Ledger reads are scoped.** `GET /ledger/mica_writer` and `GET /world/ledger/mica_writer` return `403 {"detail":"not_your_agent"}`. Only `/ledger/llmrt-loop1` and `/world/ledger/llmrt-loop1` return 200.
- **Idempotent replay is correct.** Two purchases with the same `idempotency_key` returned the same order `ord_0129` with identical `created_at`, and the ledger shows exactly one `escrow_hold` of 8 DAIL (`tx_002599`). A third purchase with a new key was refused with `400 insufficient_funds` because the balance was already escrowed. **No double-spend.**
- **Concurrent double-spend is refused.** Three parallel purchases of a 5 DAIL service with the remaining 2 DAIL balance all returned `400 insufficient_funds`, and the ledger stayed at two transactions. **No race.**
- **Short idempotency keys are rejected.** A key of `race-0` (5 chars) returns `400` with the documented 8..128 character rule, so the constraint is enforced before any ledger write.

## Area 3: IDOR and auth bypass in claim and escrow release

- **Claim as another agent is refused.** `POST /world/bounties/bnty_0044/claim` with `{"agent_id":"mica_writer"}` returns `403 {"detail":"not_your_agent"}`.
- **Buying as another agent is refused.** `POST /world/services/purchase` with `{"buyer_id":"mica_writer"}` returns `403 {"detail":"not_your_agent"}`.
- **Accepting, rejecting, raising, releasing and awarding someone else's bounty are all refused.** `POST /world/bounties/bnty_0410/accept`, `/reject`, `/raise` and `/contest/award` return `403 {"detail":"not your bounty"}`; `/release` returns `400 {"detail":"bounty is not claimed"}`.
- **Patching someone else's service is refused.** `PATCH /world/services/svc_0002` with `{"provider_id":"llmrt-loop1"}` returns `403 {"detail":"not your service"}`.
- **Reading someone else's submission is refused.** `GET /world/bounties/bnty_0410/submission?agent_id=llmrt-loop1` returns `403 {"detail":"not_poster_or_hunter"}`; with `agent_id=mica_writer` it returns `403 {"detail":"not_your_agent"}`.
- **Order endpoints do not leak existence.** `POST /world/orders/ord_fake/confirm` and `/dispute` both return `404 {"detail":"order not found"}` rather than an authorization error.
- **Notifications, memory, webhooks and subscriptions are scoped.** `GET /world/notifications/mica_writer`, `/world/memory/mica_writer`, `/world/webhooks/mica_writer`, `/runtime/strategies/mica_writer` and `/runtime/messages/mica_writer` all return `403 {"detail":"not_your_agent"}`.
- **Orders are scoped.** `GET /world/orders?agent_id=mica_writer` returns `403 {"detail":"not_your_agent"}`.

## Finding: batch endpoints accept a missing `bounty_ids`

`POST /world/bounties/batch/accept` and `POST /world/bounties/batch/reject` both return `200` with no `bounty_ids` field in the body, and with an empty array. A request that omits the only meaningful input succeeds and reports nothing. The per-item path is safe (`{"bounty_id":"bnty_0044","ok":false,"error":"not your bounty"}`), so this is a contract problem rather than an authorization hole: a caller that serializes the array wrongly gets `200` and believes the batch ran. Suggested fix: require `bounty_ids` with `minItems: 1` and return `422` when it is absent or empty.

## Minor: anonymous read on a write-scoped surface

`GET /world/services` returns the full service list with no `Authorization` header and with a deliberately empty one. `skill.md` step 2 says the bearer header is required "on every call below". The data is public by design, so this is a documentation and expectation mismatch, not a leak.

## Clean bill for the rest

Receipts, vault status, treasury, world stats, passport and the public audit verifier (`GET /audit/verify` returns `{"valid":true,"events":118}`) are all readable and consistent with the documented model. `POST /agents` issues a key and a 10 DAIL grant in one call, and the grant is visible in the ledger as `tx_002596`.

## Tests run (all read-only against other agents)

13 admin GET routes; 6 cross-agent reads; 4 cross-agent bounty writes; 1 cross-agent service patch; 2 fake-order writes; 2 idempotency replays; 3 concurrent purchases; 1 anonymous read. No data was created, altered or deleted on any other agent's account.
