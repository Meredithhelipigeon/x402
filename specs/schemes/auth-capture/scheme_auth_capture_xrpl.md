# Scheme: `auth-capture` on `XRPL`

## Summary

This is the XRP Ledger binding of [`auth-capture`](./scheme_auth_capture.md). It realizes the scheme with a native **conditional Escrow**, so no contract is deployed and every guarantee below is enforced by XRPL consensus.

It is a *full-capture-only profile*: the client authorizes an exact amount, and the hold is either captured in full or reclaimed after the deadline.

The client signs an `EscrowCreate` that locks `amount` drops for `payTo`, guarded by a PREIMAGE-SHA-256 crypto-condition and a `CancelAfter` deadline. The facilitator relays only that collect. A **capture authorizer** chosen by the client holds the preimage and releases the escrow with `EscrowFinish` once it accepts what the resource server delivered. If it never does, anyone may submit `EscrowCancel` after the deadline, and the funds can only return to the client.

Use cases: verify-then-pay for agents buying data or work, where the seller is paid only after an independent check of the delivery; delayed delivery with buyer recourse.

## Network and asset

`network` is `xrpl:0` (mainnet) or `xrpl:1` (testnet). Network binding follows [`exact` XRPL §5](../exact/scheme_exact_xrpl.md#5-network-binding): `NetworkID` MUST be omitted, the facilitator MUST submit only to the requested network, and clients SHOULD use different accounts per network.

`asset` is `"XRP"` and `amount` is a string of drops. Issued currencies are out of scope for this profile.

## Payment flow

`paymentFlow` is `escrow` (settle → resource → settle).

| Step | Operation | XRPL | Who |
| --- | --- | --- | --- |
| Before the handler | `authorize` | presigned `EscrowCreate`, relayed by `/settle` | Facilitator |
| After the handler | — | nothing | — |
| Out of band | `capture` | `EscrowFinish` with the fulfillment, before `CancelAfter` | Capture authorizer |
| After the deadline | `reclaim` | `EscrowCancel`, after `CancelAfter` | Anyone; funds go only to the client |

`operatorType` MUST be `"custom"` and `captureMode` MUST be `"deferred"`: the resource server skips the after-handler settle, and capture happens when the authorizer accepts the delivery.

## PaymentRequirements

```json
{
  "scheme": "auth-capture",
  "network": "xrpl:1",
  "amount": "1000000",
  "asset": "XRP",
  "payTo": "rSELLER...",
  "maxTimeoutSeconds": 60,
  "extra": {
    "paymentFlow": "escrow",
    "assetTransferMethod": "sequence",
    "areFeesSponsored": false,
    "operatorType": "custom",
    "captureAuthorizer": "rARBITER...",
    "captureMode": "deferred",
    "orderId": "ord_7f3c...",
    "condition": "A0258020<64 hex>810120",
    "captureDeadline": 1761330000
  }
}
```

### `extra` fields

| Field | Required | Meaning |
| --- | --- | --- |
| `paymentFlow` | Yes | `"escrow"`. |
| `assetTransferMethod` | No | `"sequence"` (default) or `"ticketSequence"`, resolved as in [`exact` XRPL §7](../exact/scheme_exact_xrpl.md#7-expiry-and-account-sequencing). |
| `areFeesSponsored` | Yes | `false`. The payer pays the `EscrowCreate` fee. |
| `operatorType` | Yes | `"custom"`. |
| `captureAuthorizer` | Yes | Classic address of the party holding the preimage. |
| `captureMode` | Yes | `"deferred"`. |
| `orderId` | Yes | The capture authorizer's order id. Fresh on every `402`. |
| `condition` | Yes | PREIMAGE-SHA-256 condition, 78 uppercase hex characters: `A0258020` ‖ SHA-256(preimage) ‖ `810120`. Fresh on every `402`. |
| `captureDeadline` | Yes | Absolute Unix seconds. `CancelAfter = captureDeadline − 946684800`. Fresh on every `402`. |

`orderId`, `condition` and `captureDeadline` are dynamic. A resource server MUST validate them as part of the requirements the client paid against and MUST NOT regenerate them on the paid retry. `refundDeadline`, `feeRecipient`, `minFeeBps` and `maxFeeBps` are absent, because refunds and fees are unsupported.

The preimage MUST be 32 bytes from a CSPRNG and MUST NOT be reused across orders.

## Client payment payload

```json
{ "x402Version": 2, "accepted": { "...": "the requirements above" }, "payload": { "signedTxBlob": "120001..." } }
```

`signedTxBlob` is the hex of a fully signed `EscrowCreate`:

| Field | Value |
| --- | --- |
| `TransactionType` | `EscrowCreate` |
| `Account` | payer |
| `Destination` | `payTo` |
| `Amount` | `amount` |
| `Condition` | `extra.condition` |
| `CancelAfter` | `captureDeadline − 946684800` |
| `Sequence` | the payer's current sequence, or `0` with `TicketSequence` |
| `Fee` | at most the facilitator's policy maximum |
| `LastLedgerSequence` | within the policy in [`exact` XRPL §7](../exact/scheme_exact_xrpl.md#7-expiry-and-account-sequencing) |

`FinishAfter`, `DestinationTag`, `Memos`, `Delegate`, `Signers` and non-zero `Flags` MUST be absent.

### Payment identity

The escrow is identified by `(Owner = Account, OfferSequence = Sequence, or TicketSequence if a ticket was used)`, which anyone holding the blob can derive. The per-order `condition` binds the escrow to one order; `EscrowCreate` has no `InvoiceID`.

### Client policy

A client SHOULD refuse to sign unless `captureAuthorizer` is an authorizer it trusts and `captureDeadline − now` falls inside its own lock window: long enough for the authorizer to act, short enough to bound how long funds are held.

## Verification

In order. Reasons are namespaced `invalid_auth_capture_xrpl_*` unless noted.

1. **Envelope.** `x402Version === 2`; `accepted.scheme` and `requirements.scheme` are `auth-capture`; `accepted.network === requirements.network` ∈ {`xrpl:0`, `xrpl:1`}; the selected `assetTransferMethod` follows `exact` XRPL §7; `asset === "XRP"`.
2. **`extra`.** The constant fields above; `condition` well-formed; `captureAuthorizer` a valid classic address; `captureDeadline` an integer in `[now + minLock, now + maxLock]` under facilitator policy; `orderId`, `condition`, `captureDeadline` and `captureAuthorizer` equal between `accepted.extra` and `requirements.extra`.
3. **Decode** `signedTxBlob`; `TransactionType === "EscrowCreate"`.
4. **Destination** `=== payTo`; no `DestinationTag`.
5. **Amount** is a string of drops `=== requirements.amount`.
6. **Condition** `=== extra.condition` (case-insensitive).
7. **Deadline.** `CancelAfter === captureDeadline − 946684800`; no `FinishAfter`.
8. **Network binding.** `NetworkID` absent.
9. **Sequencing and expiry.** As in `exact` XRPL §7: `LastLedgerSequence` present, after the current validated ledger and within policy; `sequence` requires `Sequence` equal to the account's current sequence and no `TicketSequence`; `ticketSequence` requires `Sequence === 0` and an available ticket.
10. **Safety.** `Fee` ≤ policy maximum; no `Memos`, `Delegate` or `Signers`; `Flags` is `0`.
11. **Signature.** Single-signed, valid, and the signing key derives to `Account`, whose master key is not disabled.
12. **Accounts.** The payer's balance covers `amount` + `Fee` + the reserve with this escrow added (base + increment × (OwnerCount + 1)). The destination exists and has neither `lsfDepositAuth` nor `lsfRequireDestTag` set.

A payload carrying `payload.type` is a lifecycle operation, which this binding never relays; the facilitator MUST reject it with `invalid_auth_capture_xrpl_lifecycle_not_relayed`.

## Settlement

`/settle` re-runs verification, claims the transaction hash in the duplicate-settlement guard, submits the blob to the requested network, waits for a validated result, and succeeds only on `tesSUCCESS`.

The `SettlementResponse` carries the standard fields: `success`, `transaction` (the `EscrowCreate` hash), `network`, `payer` and `amount`.

### Duplicate settlement mitigation (REQUIRED)

As in [`exact` XRPL](../exact/scheme_exact_xrpl.md#duplicate-settlement-mitigation-required): XRPL answers every resubmission of a known blob with the same validated success. The facilitator MUST check and record the transaction hash atomically before submitting, retain it until `LastLedgerSequence` has passed (also after a failed submit), and reject a hit with `duplicate_settlement`.

## Capture and reclaim

Neither is relayed by the facilitator.

- **Capture.** The capture authorizer submits `EscrowFinish { Owner, OfferSequence, Condition, Fulfillment }`, with `Fulfillment = A0228020 ‖ preimage`, while the parent ledger's close time is before `CancelAfter`. It pays the fee, which exceeds the base fee because of the fulfillment. The preimage becomes public once the finish validates; the escrow no longer exists by then.
- **Reclaim.** Anyone submits `EscrowCancel { Owner, OfferSequence }` once close time is past `CancelAfter`. Funds can only return to `Owner`. A watcher run by the facilitator, client or authorizer can make this automatic.

Ledger close time has coarse resolution (up to about 10 s). Implementations SHOULD keep margins of about 20 s on both sides of the deadline.

## `/supported`

```json
{
  "x402Version": 2,
  "scheme": "auth-capture",
  "network": "xrpl:1",
  "extra": {
    "areFeesSponsored": false,
    "operatorTypes": ["custom"],
    "captureModes": ["deferred"],
    "assetTransferMethods": ["sequence"],
    "maxLockSeconds": 900
  }
}
```

The facilitator signs nothing on this path, so it advertises no `signers` for `xrpl:*`. `maxLockSeconds` is the upper bound of the rule 2 lock window. The EVM `operators` allowlist does not apply: there is no operator contract, only the escrow condition.

## Error codes

Verification failures use the reasons below. Standard reasons keep their canonical names.

| Reason | Rule |
| --- | --- |
| `invalid_auth_capture_xrpl_x402_version`, `_scheme`, `_network`, `_asset_transfer_method`, `_asset` | 1 |
| `invalid_auth_capture_xrpl_payment_flow`, `_operator_type`, `_capture_mode`, `_fees_sponsored`, `_condition`, `_capture_authorizer`, `_capture_deadline`, `_order_id` | 2 |
| `invalid_auth_capture_xrpl_payload`, `_transaction_type` | 3 |
| `invalid_auth_capture_xrpl_destination`, `_destination_tag` | 4 |
| `invalid_auth_capture_xrpl_amount` | 5 |
| `invalid_auth_capture_xrpl_condition_mismatch` | 6 |
| `invalid_auth_capture_xrpl_cancel_after`, `_finish_after` | 7 |
| `invalid_auth_capture_xrpl_network_id` | 8 |
| `invalid_auth_capture_xrpl_last_ledger_sequence`, `_sequence`, `_ticket` | 9 |
| `invalid_auth_capture_xrpl_fee`, `_memos`, `_delegate`, `_signers`, `_flags` | 10 |
| `invalid_auth_capture_xrpl_signature` | 11 |
| `invalid_auth_capture_xrpl_insufficient_funds`, `_destination_not_found`, `_destination_flags` | 12 |
| `invalid_auth_capture_xrpl_lifecycle_not_relayed` | lifecycle payload |
| `duplicate_settlement` | settlement guard |

Ledger results map onto the same reasons: `tecUNFUNDED` and `tecINSUFFICIENT_RESERVE` to `_insufficient_funds`, `tecNO_DST` and `tecNO_DST_INSUF_XRP` to `_destination_not_found`, `tecNO_PERMISSION` to `_destination_flags`, `tefPAST_SEQ` to `_sequence`. Any other non-`tesSUCCESS` result is `invalid_auth_capture_xrpl_ledger_rejected`.

## Security considerations

**Trust minimization.** The facilitator holds no key and no funds and cannot redirect a payment: `Destination` is fixed in the signed blob. The capture authorizer can only pay `payTo` the full amount or let the escrow refund.

**Choosing the authorizer.** The client trusts the authorizer to judge honestly, so the client picks it and refuses unknown `captureAuthorizer` addresses. A resource server acting as its own authorizer reduces the scheme to pay-first.

**Delivery binding.** Resource servers SHOULD sign the bytes they deliver (for example SHA-256 of `orderId + "\n" + body` with the `payTo` key), so the authorizer judges exactly what the client received and the client cannot forge a bad delivery.

**Reclaim is open.** The core scheme describes `reclaim` as the client's escape hatch. On XRPL anyone may submit `EscrowCancel` after the deadline; funds still go only to `Owner`.

## Limitations

- **No void before the deadline.** XRPL cannot cancel an escrow before `CancelAfter`, so a rejected delivery refunds at the deadline.
- **Full capture only.** An escrow finishes for its whole amount. Partial capture would need Payment Channels, as in [`upto` XRPL](../upto/scheme_upto_xrpl.md).
- **No refund after capture and no fee split.** `EscrowFinish` pays `Destination` the full amount.
- **XRP only.** Issued-currency holds (`TokenEscrow`) are a natural extension.
- **Owner reserve.** Each open escrow holds one owner-reserve increment on the payer until it closes.

## Appendix

### Network requirements

| # | Requirement | This binding |
| --- | --- | --- |
| 1 | Hold / settlement mechanism | Conditional `EscrowCreate`. Capture is `EscrowFinish`, full amount only. The deadline is `CancelAfter`, enforced by consensus. No void or refund. |
| 2 | Client authorization format | Presigned `EscrowCreate` in `signedTxBlob`; identity `(Account, Sequence \| TicketSequence)`. |
| 3 | Operator model | `"custom"`. The facilitator relays only `authorize`. Capture is driven by `captureAuthorizer`, reclaim by anyone, both out of band. |
| 4 | Server consent | No lifecycle operation is relayed. The right to capture is the fulfillment, held by the authorizer the client chose. |
| 5 | Replay protection | The ledger consumes each escrow once. The collect is deduplicated by transaction hash. |
| 6 | Per-operation verification and settlement | `authorize`: [Verification](#verification), then [Settlement](#settlement). Capture and reclaim are checked by consensus. |
| 7 | Refund funding | Refunds are unsupported. `reclaim` returns the client's own hold; the facilitator is never a source of value. |
| 8 | Sync capture-and-void | Unsupported; `captureMode` MUST be `"deferred"`. |

## Version History

| Version | Date | Changes | Authors |
| --- | --- | --- | --- |
| v0.1 | 2026-10-26 | Initial draft (full-capture-only profile) | @Meredithhelipigeon |
