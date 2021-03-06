# BOLT #2: Peer Protocol for Channel Management

The peer channel protocol has three phases: establishment, normal
operation, and closing.

# Table of Contents

  * [Channel](#channel)
    * [Definition of `channel_id`](#definition-of-channel-id)
    * [Channel Establishment](#channel-establishment)
      * [The `open_channel` Message](#the-open_channel-message)
      * [The `accept_channel` Message](#the-accept_channel-message)
      * [The `funding_created` Message](#the-funding_created-message)
      * [The `funding_signed` Message](#the-funding_signed-message)
      * [The `funding_locked` Message](#the-funding_locked-message)
    * [Channel Close](#channel-close)
      * [Closing Initiation: `shutdown`](#closing-initiation-shutdown)
      * [Closing Negotiation: `closing_signed`](#closing-negotiation-closing_signed)
    * [Normal Operation](#normal-operation)
      * [Forwarding HTLCs](#forwarding-htlcs)
      * [`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)
      * [Adding an HTLC: `update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [Committing Updates So Far: `commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [Completing the Transition to the Updated State: `revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [Updating Fees: `update_fee`](#updating-fees-update_fee)
    * [Message Retransmission: `channel_reestablish` message](#message-retransmission)
  * [Authors](#authors)

# Channel

## Definition of `channel_id`

Some messages use a `channel_id` to identify the channel. It's
derived from the funding transaction by combining the `funding_txid`
and the `funding_output_index`, using big-endian exclusive-OR
(i.e. `funding_output_index` alters the last 2 bytes).

Prior to channel establishment, a `temporary_channel_id` is used,
which is a random nonce.

Note that as duplicate `temporary_channel_id`s may exist from different
peers, APIs which reference channels by their channel id before the funding
transaction is created are inherently unsafe. The only protocol-provided
identifier for a channel before funding_created has been exchanged is the
(source_node_id, destination_node_id, temporary_channel_id) tuple. Note that
any such APIs which reference channels by their channel id before the funding
transaction is confirmed are also not persistent - until you know the script
pubkey corresponding to the funding output nothing prevents duplicative channel
ids.


## Channel Establishment

After authenticating and initializing a connection ([BOLT #8](08-transport.md)
and [BOLT #1](01-messaging.md#the-init-message), respectively), channel establishment may begin.
This consists of the funding node (funder) sending an `open_channel` message,
followed by the responding node (fundee) sending `accept_channel`. With the
channel parameters locked in, the funder is able to create the funding
transaction and both versions of the commitment transaction, as described in
[BOLT #3](03-transactions.md#bolt-3-bitcoin-transaction-and-script-formats).
The funder then sends the outpoint of the funding output with the `funding_created`
message, along with the signature for the fundee's version of the commitment
transaction. Once the fundee learns the funding outpoint, it's able to
generate the signature for the funder's version of the commitment transaction and send it
over using the `funding_signed` message.

Once the channel funder receives the `funding_signed` message, it
must broadcast the funding transaction to the Bitcoin network. After
the `funding_signed` message is sent/received, both sides should wait
for the funding transaction to enter the blockchain and reach the
specified depth (number of confirmations). After both sides have sent
the `funding_locked` message, the channel is established and can begin
normal operation. The `funding_locked` message includes information
that will be used to construct channel authentication proofs.


        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |   A   |--(3)--  funding_created  --->|   B   |
        |       |<-(4)--  funding_signed  -----|       |
        |       |                              |       |
        |       |--(5)--- funding_locked  ---->|       |
        |       |<-(6)--- funding_locked  -----|       |
        +-------+                              +-------+

        - where node A is 'funder' and node B is 'fundee'

If this fails at any stage, or if one node decides the channel terms
offered by the other node are not suitable, the channel establishment
fails.

Note that multiple channels can operate in parallel, as all channel
messages are identified by either a `temporary_channel_id` (before the
funding transaction is created) or a `channel_id` (derived from the
funding transaction).

### The `open_channel` Message

This message contains information about a node and indicates its
desire to set up a new channel. This is the first step toward creating
the funding transaction and both versions of the commitment transaction.

1. type: 32 (`open_channel`)
2. data:
   * [`chain_hash`:`chain_hash`]
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`funding_satoshis`]
   * [`u64`:`push_msat`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`feerate_per_kw`]
   * [`u16`:`to_self_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`revocation_basepoint`]
   * [`point`:`payment_basepoint`]
   * [`point`:`delayed_payment_basepoint`]
   * [`point`:`htlc_basepoint`]
   * [`point`:`first_per_commitment_point`]
   * [`byte`:`channel_flags`]
   * [`u16`:`shutdown_len`] (`option_upfront_shutdown_script`)
   * [`shutdown_len*byte`:`shutdown_scriptpubkey`] (`option_upfront_shutdown_script`)

The `chain_hash` value denotes the exact blockchain that the opened channel will
reside within. This is usually the genesis hash of the respective blockchain.
The existence of the `chain_hash` allows nodes to open channels
across many distinct blockchains as well as have channels within multiple
blockchains opened to the same peer (if it supports the target chains).

The `temporary_channel_id` is used to identify this channel on a per-peer basis until the
funding transaction is established, at which point it is replaced
by the `channel_id`, which is derived from the funding transaction.

`funding_satoshis` is the amount the sender is putting into the
channel. `push_msat` is an amount of initial funds that the sender is
unconditionally giving to the receiver. `dust_limit_satoshis` is the
threshold below which outputs should not be generated for this node's
commitment or HTLC transactions (i.e. HTLCs below this amount plus
HTLC transaction fees are not enforceable on-chain). This reflects the
reality that tiny outputs are not considered standard transactions and
will not propagate through the Bitcoin network. `channel_reserve_satoshis`
is the minimum amount that the other node is to keep as a direct
payment. `htlc_minimum_msat` indicates the smallest value HTLC this
node will accept.

`max_htlc_value_in_flight_msat` is a cap on total value of outstanding
HTLCs, which allows a node to limit its exposure to HTLCs; similarly,
`max_accepted_htlcs` limits the number of outstanding HTLCs the other
node can offer.

`feerate_per_kw` indicates the initial fee rate in satoshi per 1000-weight
(i.e. 1/4 the more normally-used 'satoshi per 1000 vbytes') that this
side will pay for commitment and HTLC transactions, as described in
[BOLT #3](03-transactions.md#fee-calculation) (this can be adjusted
later with an `update_fee` message).

`to_self_delay` is the number of blocks that the other node's to-self
outputs must be delayed, using `OP_CHECKSEQUENCEVERIFY` delays; this
is how long it will have to wait in case of breakdown before redeeming
its own funds.

`funding_pubkey` is the public key in the 2-of-2 multisig script of
the funding transaction output.

The various `_basepoint` fields are used to derive unique
keys as described in [BOLT #3](03-transactions.md#key-derivation) for each commitment
transaction. Varying these keys ensures that the transaction ID of
each commitment transaction is unpredictable to an external observer,
even if one commitment transaction is seen; this property is very
useful for preserving privacy when outsourcing penalty transactions to
third parties.

`first_per_commitment_point` is the per-commitment point to be used
for the first commitment transaction,

Only the least-significant bit of `channel_flags` is currently
defined: `announce_channel`. This indicates whether the initiator of
the funding flow wishes to advertise this channel publicly to the
network, as detailed within [BOLT #7](07-routing-gossip.md#bolt-7-p2p-node-and-channel-discovery).

The `shutdown_scriptpubkey` allows the sending node to commit to where
funds will go on mutual close, which the remote node should enforce
even if a node is compromised later.

[ FIXME: Describe dangerous feature bit for larger channel amounts. ]

#### Requirements

The sending node:
  - MUST ensure the `chain_hash` value identifies the chain it wishes to open the channel within.
  - MUST ensure `temporary_channel_id` is unique from any other channel ID with the same peer.
  - MUST set `funding_satoshis` to less than 2^24 satoshi.
  - MUST set `push_msat` to equal or less than 1000 * `funding_satoshis`.
  - MUST set `funding_pubkey`, `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, and `delayed_payment_basepoint` to valid DER-encoded, compressed, secp256k1 pubkeys.
  - MUST set `first_per_commitment_point` to the per-commitment point to be used for the initial commitment transaction, derived as specified in [BOLT #3](03-transactions.md#per-commitment-secret-requirements).
  - MUST set `channel_reserve_satoshis` greater than or equal to `dust_limit_satoshis`.
  - MUST set undefined bits in `channel_flags` to 0.
  - if both nodes advertised the `option_upfront_shutdown_script` feature:
    - MUST include either a valid `shutdown_scriptpubkey` as required by `shutdown` `scriptpubkey`, or a zero-length `shutdown_scriptpubkey`.
  - otherwise:
    - MAY include a`shutdown_scriptpubkey`.

The sending node SHOULD:
  - set `to_self_delay` sufficient to ensure the sender can irreversibly spend a commitment transaction output, in case of misbehavior by the receiver.
  - set `feerate_per_kw` to at least the rate it estimates would cause the transaction to be immediately included in a block.
  - set `dust_limit_satoshis` to a sufficient value to allow commitment transactions to propagate through the Bitcoin network.
  - set `htlc_minimum_msat` to the minimum value HTLC it's willing to accept from this peer.

The receiving node MUST:
  - ignore undefined bits in `channel_flags`.
  - if the connection has been re-established after receiving a previous
 `open_channel`, BUT before receiving a `funding_created` message:
    - accept a new `open_channel` message.
    - discard the previous `open_channel` message.

The receiving node MAY fail the channel if:
  - `announce_channel` is `false` (`0`), yet it wishes to publicly announce the channel.
  - `funding_satoshis` is too small.
  - it considers `htlc_minimum_msat` too large.
  - it considers `max_htlc_value_in_flight_msat` too small.
  - it considers `channel_reserve_satoshis` too large.
  - it considers `max_accepted_htlcs` too small.
  - it considers `dust_limit_satoshis` too small and plans to rely on the sending node publishing its commitment transaction in the event of a data loss (see [message-retransmission](02-peer-protocol.md#message-retransmission)).

The receiving node MUST fail the channel if:
  - the `chain_hash` value is set to a hash of a chain that is unknown to the receiver.
  - `push_msat` is greater than `funding_satoshis` * 1000.
  - `to_self_delay` is unreasonably large.
  - `max_accepted_htlcs` is greater than 483.
  - it considers `feerate_per_kw` too small for timely processing or unreasonably large.
  - `funding_pubkey`, `revocation_basepoint`, `htlc_basepoint`, `payment_basepoint`, or `delayed_payment_basepoint`
are not valid DER-encoded compressed secp256k1 pubkeys.
  - `dust_limit_satoshis` is greater than `channel_reserve_satoshis`.
  - the funder's amount for the initial commitment transaction is not sufficient for full [fee payment](03-transactions.md#fee-payment).
  - both `to_local` and `to_remote` amounts for the initial commitment transaction are less than or equal to `channel_reserve_satoshis` (see [BOLT 3](03-transactions.md#commitment-transaction-outputs)).

The receiving node MUST NOT:
  - consider funds received, using `push_msat`, to be received until the funding transaction has reached sufficient depth.

#### Rationale

The requirement for `funding_satoshi` to be less than 2^24 satoshi is a temporary self-imposed limit while implementations are not yet considered stable.
It can be lifted at any point in time, or adjusted for other currencies, since it is solely enforced by the endpoints of a channel.
Specifically, [the routing gossip protocol](07-routing-gossip.md) does not discard channels that have a larger capacity.

The *channel reserve* is specified by the peer's `channel_reserve_satoshis`: 1% of the channel total is suggested. Each side of a channel maintains this reserve so it always has something to lose if it were to try to broadcast an old, revoked commitment transaction. Initially, this reserve may not be met, as only one side has funds; but the protocol ensures that there is always progress toward meeting this reserve, and once met, it is maintained.

The sender can unconditionally give initial funds to the receiver using a non-zero `push_msat`, but even in this case we ensure that the funder has sufficient remaining funds to pay fees and that one side has some amount it can spend (which also implies there is at least one non-dust output). Note that, like any other on-chain transaction, this payment is not certain until the funding transaction has been confirmed sufficiently (with a danger of double-spend until this occurs) and may require a separate method to prove payment via on-chain confirmation.

The `feerate_per_kw` is generally only of concern to the sender (who pays the fees), but there is also the fee rate paid by HTLC transactions; thus, unreasonably large fee rates can also penalize the recipient.

Separating the `htlc_basepoint` from the `payment_basepoint` improves security: a node needs the secret associated with the `htlc_basepoint` to produce HTLC signatures for the protocol, but the secret for the `payment_basepoint` can be in cold storage.

The requirement that `channel_reserve_satoshis` is not considered dust
according to `dust_limit_satoshis` eliminates cases where all outputs
would be eliminated as dust.  The similar requirements in
`accept_channel` ensure that both sides' `channel_reserve_satoshis`
are above both `dust_limit_satoshis`.

Details for how to handle a channel failure can be found in [BOLT 5:Failing a Channel](05-onchain.md#failing-a-channel).

#### Future

It would be easy to have a local feature bit which indicated that a
receiving node was prepared to fund a channel, which would reverse this
protocol.

### The `accept_channel` Message

This message contains information about a node and indicates its
acceptance of the new channel. This is the second step toward creating the
funding transaction and both versions of the commitment transaction.

1. type: 33 (`accept_channel`)
2. data:
   * [`32*byte`:`temporary_channel_id`]
   * [`u64`:`dust_limit_satoshis`]
   * [`u64`:`max_htlc_value_in_flight_msat`]
   * [`u64`:`channel_reserve_satoshis`]
   * [`u64`:`htlc_minimum_msat`]
   * [`u32`:`minimum_depth`]
   * [`u16`:`to_self_delay`]
   * [`u16`:`max_accepted_htlcs`]
   * [`point`:`funding_pubkey`]
   * [`point`:`revocation_basepoint`]
   * [`point`:`payment_basepoint`]
   * [`point`:`delayed_payment_basepoint`]
   * [`point`:`htlc_basepoint`]
   * [`point`:`first_per_commitment_point`]
   * [`u16`:`shutdown_len`] (`option_upfront_shutdown_script`)
   * [`shutdown_len*byte`:`shutdown_scriptpubkey`] (`option_upfront_shutdown_script`)

#### Requirements

The `temporary_channel_id` MUST be the same as the `temporary_channel_id` in
the `open_channel` message.

The sender:
  - SHOULD set `minimum_depth` to a number of blocks it considers reasonable to
avoid double-spending of the funding transaction.
  - MUST set `channel_reserve_satoshis` greater than or equal to `dust_limit_satoshis` from the `open_channel` message.
  - MUST set `dust_limit_satoshis` less than or equal to `channel_reserve_satoshis` from the `open_channel` message.

The receiver:
  - if `minimum_depth` is unreasonably large:
    - MAY reject the channel.
  - if `channel_reserve_satoshis` is less than `dust_limit_satoshis` within the `open_channel` message:
	- MUST reject the channel.
  - if `channel_reserve_satoshis` from the `open_channel` message is less than `dust_limit_satoshis`:
	- MUST reject the channel.
Other fields have the same requirements as their counterparts in `open_channel`.

### The `funding_created` Message

This message describes the outpoint which the funder has created for
the initial commitment transactions. After receiving the peer's
signature, via `funding_signed`, it will broadcast the funding transaction.

1. type: 34 (`funding_created`)
2. data:
    * [`32*byte`:`temporary_channel_id`]
    * [`sha256`:`funding_txid`]
    * [`u16`:`funding_output_index`]
    * [`signature`:`signature`]

#### Requirements

The sender MUST set:
  - `temporary_channel_id` the same as the `temporary_channel_id` in the `open_channel` message.
  - `funding_txid` to the transaction ID of a non-malleable transaction,
    - and MUST NOT broadcast this transaction.
  - `funding_output_index` to the output number of that transaction that corresponds the funding transaction output, as defined in [BOLT #3](03-transactions.md#funding-transaction-output).
  - `signature` to the valid signature using its `funding_pubkey` for the initial commitment transaction, as defined in [BOLT #3](03-transactions.md#commitment-transaction).

The sender:
  - when creating the funding transaction:
    - SHOULD use only BIP141 (Segregated Witness) inputs.

The recipient:
  - if `signature` is incorrect:
    - MUST fail the channel.

#### Rationale

The `funding_output_index` can only be 2 bytes, since that's how it's packed into the `channel_id` and used throughout the gossip protocol. The limit of 65535 outputs should not be overly burdensome.

A transaction with all Segregated Witness inputs is not malleable, hence the funding transaction recommendation.

### The `funding_signed` Message

This message gives the funder the signature it needs for the first
commitment transaction, so it can broadcast the transaction knowing that funds
can be redeemed, if need be.

This message introduces the `channel_id` to identify the channel. It's derived from the funding transaction by combining the `funding_txid` and the `funding_output_index`, using big-endian exclusive-OR (i.e. `funding_output_index` alters the last 2 bytes).

1. type: 35 (`funding_signed`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`signature`:`signature`]

#### Requirements

Both peers:
  - if `option_static_remotekey` was negotiated:
    - `option_static_remotekey` applies to all commitment transactions
  - otherwise:
    - `option_static_remotekey` does not apply to any commitment transactions

The sender MUST set:
  - `channel_id` by exclusive-OR of the `funding_txid` and the `funding_output_index` from the `funding_created` message.
  - `signature` to the valid signature, using its `funding_pubkey` for the initial commitment transaction, as defined in [BOLT #3](03-transactions.md#commitment-transaction).

The recipient:
  - if `signature` is incorrect:
    - MUST fail the channel.
  - MUST NOT broadcast the funding transaction before receipt of a valid `funding_signed`.
  - on receipt of a valid `funding_signed`:
    - SHOULD broadcast the funding transaction.

#### Rationale

We decide on `option_static_remotekey` at this point when we first have to generate the commitment
transaction.  Even if a later reconnection does not negotiate this parameter, this channel will continue to use `option_static_remotekey`; we don't support "downgrading".
This simplifies channel state, particularly penalty transaction handling.

### The `funding_locked` Message

This message indicates that the funding transaction has reached the `minimum_depth` asked for in `accept_channel`. Once both nodes have sent this, the channel enters normal operating mode.

1. type: 36 (`funding_locked`)
2. data:
    * [`channel_id`:`channel_id`]
    * [`point`:`next_per_commitment_point`]

#### Requirements

The sender MUST:
  - NOT send `funding_locked` unless outpoint of given by `funding_txid` and
   `funding_output_index` in the `funding_created` message pays exactly `funding_satoshis` to the scriptpubkey specified in [BOLT #3](03-transactions.md#funding-transaction-output).
  - wait until the funding transaction has reached
`minimum_depth` before sending this message.
  - set `next_per_commitment_point` to the
per-commitment point to be used for the following commitment
transaction, derived as specified in
[BOLT #3](03-transactions.md#per-commitment-secret-requirements).

A non-funding node (fundee):
  - SHOULD forget the channel if it does not see the correct
funding transaction after a reasonable timeout.

From the point of waiting for `funding_locked` onward, either node MAY
fail the channel if it does not receive a required response from the
other node after a reasonable timeout.

#### Rationale

The non-funder can simply forget the channel ever existed, since no
funds are at risk. If the fundee were to remember the channel forever, this
would create a Denial of Service risk; therefore, forgetting it is recommended
(even if the promise of `push_msat` is significant).

## Channel Close

Nodes can negotiate a mutual close of the connection, which unlike a
unilateral close, allows them to access their funds immediately and
can be negotiated with lower fees.

Closing happens in two stages:
1. one side indicates it wants to clear the channel (and thus will accept no new HTLCs)
2. once all HTLCs are resolved, the final channel close negotiation begins.

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |   A   |                 ...          |   B   |
        |       |                              |       |
        |       |--(3)-- closing_signed  F1--->|       |
        |       |<-(4)-- closing_signed  F2----|       |
        |       |              ...             |       |
        |       |--(?)-- closing_signed  Fn--->|       |
        |       |<-(?)-- closing_signed  Fn----|       |
        +-------+                              +-------+

### Closing Initiation: `shutdown`

Either node (or both) can send a `shutdown` message to initiate closing,
along with the `scriptpubkey` it wants to be paid to.

1. type: 38 (`shutdown`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u16`:`len`]
   * [`len*byte`:`scriptpubkey`]

#### Requirements

A sending node:
  - if it hasn't sent a `funding_created` (if it is a funder) or a `funding_signed` (if it is a fundee):
    - MUST NOT send a `shutdown`
  - MAY send a `shutdown` before a `funding_locked`, i.e. before the funding transaction has reached `minimum_depth`.
  - if there are updates pending on the receiving node's commitment transaction:
    - MUST NOT send a `shutdown`.
  - MUST NOT send an `update_add_htlc` after a `shutdown`.
  - if no HTLCs remain in either commitment transaction:
	- MUST NOT send any `update` message after a `shutdown`.
  - SHOULD fail to route any HTLC added after it has sent `shutdown`.
  - if it sent a non-zero-length `shutdown_scriptpubkey` in `open_channel` or `accept_channel`:
    - MUST send the same value in `scriptpubkey`.
  - MUST set `scriptpubkey` in one of the following forms:

    1. `OP_DUP` `OP_HASH160` `20` 20-bytes `OP_EQUALVERIFY` `OP_CHECKSIG`
   (pay to pubkey hash), OR
    2. `OP_HASH160` `20` 20-bytes `OP_EQUAL` (pay to script hash), OR
    3. `OP_0` `20` 20-bytes (version 0 pay to witness pubkey), OR
    4. `OP_0` `32` 32-bytes (version 0 pay to witness script hash)

A receiving node:
  - if it hasn't received a `funding_signed` (if it is a funder) or a `funding_created` (if it is a fundee):
    - SHOULD fail the connection
  - if the `scriptpubkey` is not in one of the above forms:
    - SHOULD fail the connection.
  - if it hasn't sent a `funding_locked` yet:
    - MAY reply to a `shutdown` message with a `shutdown`
  - once there are no outstanding updates on the peer, UNLESS it has already sent a `shutdown`:
    - MUST reply to a `shutdown` message with a `shutdown`
  - if both nodes advertised the `option_upfront_shutdown_script` feature, and the receiving node received a non-zero-length `shutdown_scriptpubkey` in `open_channel` or `accept_channel`, and that `shutdown_scriptpubkey` is not equal to `scriptpubkey`:
    - MUST fail the connection.

#### Rationale

If channel state is always "clean" (no pending changes) when a
shutdown starts, the question of how to behave if it wasn't is avoided:
the sender always sends a `commitment_signed` first.

As shutdown implies a desire to terminate, it implies that no new
HTLCs will be added or accepted.  Once any HTLCs are cleared, the peer
may immediately begin closing negotiation, so we ban further updates
to the commitment transaction (in particular, `update_fee` would be
possible otherwise).

The `scriptpubkey` forms include only standard forms accepted by the
Bitcoin network, which ensures the resulting transaction will
propagate to miners.

The `option_upfront_shutdown_script` feature means that the node
wanted to pre-commit to `shutdown_scriptpubkey` in case it was
compromised somehow.  This is a weak commitment (a malevolent
implementation tends to ignore specifications like this one!), but it
provides an incremental improvement in security by requiring the cooperation
of the receiving node to change the `scriptpubkey`.

The `shutdown` response requirement implies that the node sends `commitment_signed` to commit any outstanding changes before replying; however, it could theoretically reconnect instead, which would simply erase all outstanding uncommitted changes.

### Closing Negotiation: `closing_signed`

Once shutdown is complete and the channel is empty of HTLCs, the final
current commitment transactions will have no HTLCs, and closing fee
negotiation begins.  The funder chooses a fee it thinks is fair, and
signs the closing transaction with the `scriptpubkey` fields from the
`shutdown` messages (along with its chosen fee) and sends the signature;
the other node then replies similarly, using a fee it thinks is fair.  This
exchange continues until both agree on the same fee or when one side fails
the channel.

1. type: 39 (`closing_signed`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`fee_satoshis`]
   * [`signature`:`signature`]

#### Requirements

The funding node:
  - after `shutdown` has been received, AND no HTLCs remain in either commitment transaction:
    - SHOULD send a `closing_signed` message.

The sending node:
  - MUST set `fee_satoshis` less than or equal to the
 base fee of the final commitment transaction, as calculated in [BOLT #3](03-transactions.md#fee-calculation).
  - SHOULD set the initial `fee_satoshis` according to its
 estimate of cost of inclusion in a block.
  - MUST set `signature` to the Bitcoin signature of the close
 transaction, as specified in [BOLT #3](03-transactions.md#closing-transaction).

The receiving node:
  - if the `signature` is not valid for either variant of closing transaction
  specified in [BOLT #3](03-transactions.md#closing-transaction):
    - MUST fail the connection.
  - if `fee_satoshis` is equal to its previously sent `fee_satoshis`:
    - SHOULD sign and broadcast the final closing transaction.
    - MAY close the connection.
  - otherwise, if `fee_satoshis` is greater than
the base fee of the final commitment transaction as calculated in
[BOLT #3](03-transactions.md#fee-calculation):
    - MUST fail the connection.
  - if `fee_satoshis` is not strictly
between its last-sent `fee_satoshis` and its previously-received
`fee_satoshis`, UNLESS it has since reconnected:
    - SHOULD fail the connection.
  - if the receiver agrees with the fee:
    - SHOULD reply with a `closing_signed` with the same `fee_satoshis` value.
  - otherwise:
    - MUST propose a value "strictly between" the received `fee_satoshis`
  and its previously-sent `fee_satoshis`.

#### Rationale

The "strictly between" requirement ensures that forward
progress is made, even if only by a single satoshi at a time. To avoid
keeping state and to handle the corner case, where fees have shifted
between disconnection and reconnection, negotiation restarts on reconnection.

Note there is limited risk if the closing transaction is
delayed, but it will be broadcast very soon; so there is usually no
reason to pay a premium for rapid processing.

## Normal Operation

Once both nodes have exchanged `funding_locked` (and optionally [`announcement_signatures`](07-routing-gossip.md#the-announcement_signatures-message)), the channel can be used to make payments via Hashed Time Locked Contracts.

Changes are sent in batches: one or more `update_` messages are sent before a
`commitment_signed` message, as in the following diagram:

> 両ノードが `funding_locked`（およびオプションで[` announcement_signatures`]（07-routing-gossip.md＃the-announcement_signatures-message））を交換したら、チャネルを使用してHased Time Locked Contracts を介して支払いを行うことができます。
> 変更はバッチで送信されます：1つ以上の `update_`メッセージが送信される前に次の図のような「commitment_signed」メッセージ：

        +-------+                               +-------+
        |       |--(1)---- update_add_htlc ---->|       |
        |       |--(2)---- update_add_htlc ---->|       |
        |       |<-(3)---- update_add_htlc -----|       |
        |       |                               |       |
        |       |--(4)--- commitment_signed --->|       |
        |   A   |<-(5)---- revoke_and_ack ------|   B   |
        |       |                               |       |
        |       |<-(6)--- commitment_signed ----|       |
        |       |--(7)---- revoke_and_ack ----->|       |
        |       |                               |       |
        |       |--(8)--- commitment_signed --->|       |
        |       |<-(9)---- revoke_and_ack ------|       |
        +-------+                               +-------+

Counter-intuitively, these updates apply to the *other node's*
commitment transaction; the node only adds those updates to its own
commitment transaction when the remote node acknowledges it has
applied them via `revoke_and_ack`.

> 直感に反して、これらの更新は*他のノードの* commitment transaction に適用されます。 remote node が `revoke_and_ack` を介して更新を適用したことを確認した場合、ノードはそれらの更新を自身の commitment transaction に追加するだけです。

Thus each update traverses through the following states:
> したがって、各更新は次の状態を通過します。

1. pending on the receiver
2. in the receiver's latest commitment transaction
3. ... and the receiver's previous commitment transaction has been revoked,
   and the update is pending on the sender
4. ... and in the sender's latest commitment transaction
5. ... and the sender's previous commitment transaction has been revoked

> 1. rediverが保留中
> 2. reciver の最新の commitment transaction
> 3. そして、reciver の以前の commitment transaction が取り消され、更新が sender で保留中です
> 4. および sender の最新の commitment transaction
> 5. そして、sender の以前の commitment transactoin を取り消し

As the two nodes' updates are independent, the two commitment
transactions may be out of sync indefinitely. This is not concerning:
what matters is whether both sides have irrevocably committed to a
particular update or not (the final state, above).

> 2つのノードの更新は独立しているため、2つの commitment transaciton がいつまでも同期されない場合があります。 これは関係ありません：　重要なのは、双方が irreveocably committed を持ち、特定の更新ができるかどうかです（上記の最終状態）。


### Forwarding HTLCs

In general, a node offers HTLCs for two reasons: to initiate a payment of its own,
or to forward another node's payment. In the forwarding case, care must
be taken to ensure the *outgoing* HTLC cannot be redeemed unless the *incoming*
HTLC can be redeemed. The following requirements ensure this is always true.

The respective **addition/removal** of an HTLC is considered *irrevocably committed* when:

1. The commitment transaction **with/without** it is committed to by both nodes, and any
previous commitment transaction **without/with** it has been revoked, OR
2. The commitment transaction **with/without** it has been irreversibly committed to
the blockchain.

> 一般的に、ノードは2つの理由でHTLCを提供します。それは、独自の支払いを開始するため、または別のノードの支払いを転送するためです。 転送の場合、*着信* HTLCを引き換えられない限り、*発信* HTLCが引き換えられないように注意する必要があります。 次の要件により、これが常に真になることが保証されます。

> HTLCのそれぞれの**追加/削除**は、次の場合に*irrebersibly comitted*と見なされます

> 1.両ノードによってコミットされる**あり/なし**のcommitment transaction、および**取り消される以前の commitment transaction**

> 2.ブロックチェーンに展開された commitment transaction **あり/なし**

#### Requirements

A node:
  - until an incoming HTLC has been irrevocably committed:
    - MUST NOT offer the corresponding outgoing HTLC (`update_add_htlc`) in response to that incoming HTLC.
  - until the removal of an outgoing HTLC is irrevocably committed, OR until the outgoing on-chain HTLC output has been spent via the HTLC-timeout transaction (with sufficient depth):
    - MUST NOT fail the incoming HTLC (`update_fail_htlc`) that corresponds
to that outgoing HTLC.
  - once the `cltv_expiry` of an incoming HTLC has been reached, OR if `cltv_expiry` minus `current_height` is less than `cltv_expiry_delta` for the corresponding outgoing HTLC:
    - MUST fail that incoming HTLC (`update_fail_htlc`).
  - if an incoming HTLC's `cltv_expiry` is unreasonably far in the future:
    - SHOULD fail that incoming HTLC (`update_fail_htlc`).
  - upon receiving an `update_fulfill_htlc` for an outgoing HTLC, OR upon discovering the `payment_preimage` from an on-chain HTLC spend:
    - MUST fulfill the incoming HTLC that corresponds to that outgoing HTLC.
    
> A node:
>   - HTLC が来て取り消し不可能になるまで：
>     - その HTLCに対応して、対応する HTLC（ `update_add_htlc`）を提供してはなりません。
>   - HTLC の削除が取り消し不能になる、または on-chain HTLC output が HTLC-timeout transaction を介して消費されるまで（十分な深さで）：
>     - その HTLC に対応する HTLC（ `update_fail_htlc`）を失敗させてはいけません。
>   - HTLC の 'cltv_expiry' が不当に遠い場合：
>     - HTLC (`update_fail_htlc`) を失敗する必要があります。
>   - HTLCの `update_fulfill_htlc` を受信したとき、または on-chain の HTLC の支出から `payment_preimage` を発見したとき
>     - その HTLC に対応する HTLC を満たさなければなりません。

#### Rationale

In general, one side of the exchange needs to be dealt with before the other.
Fulfilling an HTLC is different: knowledge of the preimage is, by definition,
irrevocable and the incoming HTLC should be fulfilled as soon as possible to
reduce latency.

An HTLC with an unreasonably long expiry is a denial-of-service vector and
therefore is not allowed. Note that the exact value of "unreasonable" is currently unclear
and may depend on network topology.

> 一般的に、交換の一方は他方の前に対処する必要があります。
> HTLCの実現は異なります: 定義により、preimage の情報は取消不能であり、遅延を減らすために HTLC をできるだけ早く満たす必要があります。

> 有効期限が不当に長い HTLC はサービス拒否ベクトルであるため、許可されません。 `unreasonable` の正確な値は現在不明であり、ネットワークトポロジに依存する可能性があることに注意してください。

### `cltv_expiry_delta` Selection

Once an HTLC has timed out, it can either be fulfilled or timed-out;
care must be taken around this transition, both for offered and received HTLCs.

Consider the following scenario, where A sends an HTLC to B, who
forwards to C, who delivers the goods as soon as the payment is
received.

> HTLC がタイムアウトになったら、それを実行するか タイムアウト にすることができます。 提供された HTLC と受信された HTLC の両方について、この移行に注意する必要があります。

> A が HTLC を B に送信し C に転送する次のシナリオを考えます。Cは、支払いが完了するとすぐに商品を配達します

1. C needs to be sure that the HTLC from B cannot time out, even if B becomes
   unresponsive; i.e. C can fulfill the incoming HTLC on-chain before B can
   time it out on-chain.

2. B needs to be sure that if C fulfills the HTLC from B, it can fulfill the
   incoming HTLC from A; i.e. B can get the preimage from C and fulfill the incoming
   HTLC on-chain before A can time it out on-chain.
   
> 1. C は、B が応答しなくなっても、B からの HTLC がタイムアウトにならないことを確認する必要があります。 つまり、C　は　HTLC　を on-chain で満たす前に、Bは on-chain でタイムアウトすることができます。
> 2. B は、C が B からの HTLC を実行する場合、A からの HTLC を実行できることを確認する必要があります。 つまり、A が on-chain 上でタイムアウトする前に、B は C から preimage を取得し、HTLC を on-chain で実行できます。


The critical settings here are the `cltv_expiry_delta` in
[BOLT #7](07-routing-gossip.md#the-channel_update-message) and the
related `min_final_cltv_expiry` in [BOLT #11](11-payment-encoding.md#tagged-fields).
`cltv_expiry_delta` is the minimum difference in HTLC CLTV timeouts, in
the forwarding case (B). `min_final_cltv_expiry` is the minimum difference
between HTLC CLTV timeout and the current block height, for the
terminal case (C).

Note that a node is at risk if it accepts an HTLC in one channel and
offers an HTLC in another channel with too small of a difference between
the CLTV timeouts.  For this reason, the `cltv_expiry_delta` for the
*outgoing* channel is used as the delta across a node.

> ここでの重要な設定は、[BOLT＃7]（07-routing-gossip.md＃the-channel_update-message）に関連する `cltv_expiry_delta` および[BOLT＃11]（11-payment-encoding.md＃tagged-fields）に関連する `min_final_cltv_expiry` です。 `cltv_expiry_delta`は、転送の場合の HTLC CLTV timeoutsの最小差です（B）。 `min_final_cltv_expiry`は、HTLC CLTV timeoutと最新ブロックの高さとの最小の差です（端末の場合）（C）。

> 1つのチャネルで HTLC を受け入れ、CLTV timeouts の差が小さい別チャネルで HTLCを提供する場合、ノードが危険にさらされることに注意してください。 このため、* outgoing *チャネルの `cltv_expiry_delta`はノード全体の差分として使用されます。

The worst-case number of blocks between outgoing and
incoming HTLC resolution can be derived, given a few assumptions:

* a worst-case reorganization depth `R` blocks
* a grace-period `G` blocks after HTLC timeout before giving up on
  an unresponsive peer and dropping to chain
* a number of blocks `S` between transaction broadcast and the
  transaction being included in a block

> いくつかの仮定が与えられた場合、送った HTLC 受け取った HTLC の間のブロックの最悪ケース数を導き出すことができます。

> * 最悪の場合の再編成の深さ `R` ブロック
> * 猶予期間 `G` は、HTLC timeout 後、応答しないピアをあきらめて chain に展開する前にブロックします
> * transaction broadcast　とブロックに含まれるトランザクションとの間の多数のブロック `S` 

The worst case is for a forwarding node (B) that takes the longest
possible time to spot the outgoing HTLC fulfillment and also takes
the longest possible time to redeem it on-chain:

1. The B->C HTLC times out at block `N`, and B waits `G` blocks until
   it gives up waiting for C. B or C commits to the blockchain,
   and B spends HTLC, which takes `S` blocks to be included.
2. Bad case: C wins the race (just) and fulfills the HTLC, B only sees
   that transaction when it sees block `N+G+S+1`.
3. Worst case: There's reorganization `R` deep in which C wins and
   fulfills. B only sees transaction at `N+G+S+R`.
4. B now needs to fulfill the incoming A->B HTLC, but A is unresponsive: B waits `G` more
   blocks before giving up waiting for A. A or B commits to the blockchain.
5. Bad case: B sees A's commitment transaction in block `N+G+S+R+G+1` and has
   to spend the HTLC output, which takes `S` blocks to be mined.
6. Worst case: there's another reorganization `R` deep which A uses to
   spend the commitment transaction, so B sees A's commitment
   transaction in block `N+G+S+R+G+R` and has to spend the HTLC output, which
   takes `S` blocks to be mined.
7. B's HTLC spend needs to be at least `R` deep before it times out,
   otherwise another reorganization could allow A to timeout the
   transaction.

> 最悪の場合は、転送ノード（B）が HTLC 実行を見つけるのに可能な限り長い時間を要し、on-chain 上でそれを引き換えるのに可能な限り長い時間を要する場合です。

> 1. B-> C HTLCはブロック `N` でタイムアウトし、BはCが待機を放棄するまで `G` ブロックを待機します。BまたはCは blockchain へ展開し、Bは `S` ブロック含むHTLCを使用します。
> 2. 悪いケース： Cは（ちょうど）レースに勝ち、HTLCを実行します。Bはブロック `N + G + S + 1` を検出したときにのみそのトランザクションを参照します。
> 3. 最悪の場合： Cが勝ち、実現する再編成 `R` があります。 Bは `N + G + S + R` のトランザクションのみを見ます。
> 4. B は A -> B HTLCを実行する必要がありますが、Aは応答しません。Bは A を待つことをあきらめる前に、さらに `G` ブロック待機します。AまたはBがブロックチェーンに展開します。
> 5. 悪いケース： BはAの commitment transaction をブロック `N + G + S + R + G + 1` で認識し、HTLC output を費やさなければなりません。
> 6. 最悪の場合： Aが commitment transaction を費やすために使用する別の再編成 `R` があり、BはAの commitment transaction をブロック `N + G + S + R + G + R` で確認し、HTLC出力を費やすこれは、マイニングされる `S` ブロックを取ります。
> 7. Bの HTLC output は、タイムアウトする前に少なくとも `R` の深さである必要があります。そうでない場合、別の再編成により A が transaction をタイムアウトする可能性があります。

Thus, the worst case is `3R+2G+2S`, assuming `R` is at least 1. Note that the
chances of three reorganizations in which the other node wins all of them is
low for `R` of 2 or more. Since high fees are used (and HTLC spends can use
almost arbitrary fees), `S` should be small; although, given that block times are
irregular and empty blocks still occur, `S=2` should be considered a
minimum. Similarly, the grace period `G` can be low (1 or 2), as nodes are
required to timeout or fulfill as soon as possible; but if `G` is too low it increases the
risk of unnecessary channel closure due to networking delays.

> したがって、最悪の場合は `3R + 2G + 2S` であり、`R` が少なくとも1であると仮定すると、`R` が2以上の場合、他のノードがすべてを勝ち取る3つの再編成の可能性は低くなります。 高い手数料が使用されるため（HTLCの支出にはほぼ任意の手数料を使用できるため）、`S` は小さくする必要があります。 ただし、block time が不規則で空のブロックが引き続き発生する場合、 `S = 2` を最小と見なす必要があります。 同様に、ノードはできるだけ早くタイムアウトするか実行する必要があるため、猶予期間 `G`は短くすることができます（1または2）。 しかし、`G` が低すぎると、ネットワーク遅延により不必要なチャネルが閉じられるリスクが高くなります。

There are four values that need be derived:

1. the `cltv_expiry_delta` for channels, `3R+2G+2S`: if in doubt, a
   `cltv_expiry_delta` of 12 is reasonable (R=2, G=1, S=2).

2. the deadline for offered HTLCs: the deadline after which the channel has to be failed
   and timed out on-chain. This is `G` blocks after the HTLC's
   `cltv_expiry`: 1 block is reasonable.

3. the deadline for received HTLCs this node has fulfilled: the deadline after which
the channel has to be failed and the HTLC fulfilled on-chain before its
   `cltv_expiry`. See steps 4-7 above, which imply a deadline of `2R+G+S`
   blocks before `cltv_expiry`: 7 blocks is reasonable.

4. the minimum `cltv_expiry` accepted for terminal payments: the
   worst case for the terminal node C is `2R+G+S` blocks (as, again, steps
   1-3 above don't apply). The default in
   [BOLT #11](11-payment-encoding.md) is 9, which is slightly more
   conservative than the 7 that this calculation suggests.

> 導出する必要がある4つの値があります。

> 1. チャネルの `cltv_expiry_delta`、`3R + 2G + 2S`： 疑わしい場合は、12の `cltv_expiry_delta` が妥当です（R = 2、G = 1、S = 2）。

> 2. 提供される HTLC の期限： チャネルが失敗し、on-chain 上でタイムアウトするまでの期限。 これは、HTLC の `cltv_expiry` の後の `G` ブロックです。1ブロックが妥当です。

> 3. このノードが実行したHTLCの期限： チャネルが失敗し、HTLCが `cltv_expiry` の前にon-chain上で実行しなければならない期限。 上記の手順4〜7を参照してください。これは、`cltv_expiry` の前に `2R+G+S` ブロックの期限を意味します。7ブロックが妥当です。

> 4. 末端の支払いに受け入れられる最小の `cltv_expiry`： 末端ノード C の最悪のケースは `2R + G + S` ブロックです（これも上記の手順1〜3は適用されません）。 [BOLT＃11]（11-payment-encoding.md）のデフォルトは9です。これは、この計算が示唆する7よりわずかに控えめです。

#### Requirements

An offering node:
  - MUST estimate a timeout deadline for each HTLC it offers.
  - MUST NOT offer an HTLC with a timeout deadline before its `cltv_expiry`.
  - if an HTLC which it offered is in either node's current
  commitment transaction, AND is past this timeout deadline:
    - MUST fail the channel.

> offering node：
>   - 提供する各 HTLC のタイムアウト期限を推定する必要があります。
>   - `cltv_expiry` の前にタイムアウト期限を設定した HTLC を提供してはなりません。
>   - 提供した HTLC がいずれかのノードの現在の commitment transaction にあり、かつこのタイムアウト期限を過ぎている場合：
>     - チャネルに失敗する必要があります。

A fulfilling node:
  - for each HTLC it is attempting to fulfill:
    - MUST estimate a fulfillment deadline.
  - MUST fail (and not forward) an HTLC whose fulfillment deadline is already past.
  - if an HTLC it has fulfilled is in either node's current commitment
  transaction, AND is past this fulfillment deadline:
    - MUST fail the channel.

> fulfilling node:
>   - HTLCごとに、以下を実現しようとしています。
>     - 履行期限を見積もる必要があります。
>   - 履行期限がすでに過ぎている HTLC に失敗する（転送しない）必要があります。
>   - 履行した HTLC がいずれかのノードの現在の commitment transaction にあり、この履行期限を過ぎている場合：
>     -チャネルに失敗する必要があります。

### Adding an HTLC: `update_add_htlc`

Either node can send `update_add_htlc` to offer an HTLC to the other,
which is redeemable in return for a payment preimage. Amounts are in
millisatoshi, though on-chain enforcement is only possible for whole
satoshi amounts greater than the dust limit (in commitment transactions these are rounded down as
specified in [BOLT #3](03-transactions.md)).

The format of the `onion_routing_packet` portion, which indicates where the payment
is destined, is described in [BOLT #4](04-onion-routing.md).

> どちらのノードも `update_add_htlc」` 送信して、もう一方に HTLC を提供できます。これは、preimage と引き換えに利用できます。 量は millisatoshi 単位ですが、on-chain上での執行は、全体の satoshi がダスト制限を超える場合のみ可能です（ commitment transaction では、これらは[BOLT＃3]（03-transactions.md）で指定されているように切り捨てられます）。

> 支払いの宛先を示す `onion_routing_packet` 部分の形式は、[BOLT＃4]（04-onion-routing.md）で説明されています。

1. type: 128 (`update_add_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`u64`:`amount_msat`]
   * [`sha256`:`payment_hash`]
   * [`u32`:`cltv_expiry`]
   * [`1366*byte`:`onion_routing_packet`]

#### Requirements

A sending node:
  - MUST NOT offer `amount_msat` it cannot pay for in the
remote commitment transaction at the current `feerate_per_kw` (see "Updating
Fees") while maintaining its channel reserve.
  - MUST offer `amount_msat` greater than 0.
  - MUST NOT offer `amount_msat` below the receiving node's `htlc_minimum_msat`
  - MUST set `cltv_expiry` less than 500000000.
  - for channels with `chain_hash` identifying the Bitcoin blockchain:
    - MUST set the four most significant bytes of `amount_msat` to 0.
  - if result would be offering more than the remote's
  `max_accepted_htlcs` HTLCs, in the remote commitment transaction:
    - MUST NOT add an HTLC.
  - if the sum of total offered HTLCs would exceed the remote's
`max_htlc_value_in_flight_msat`:
    - MUST NOT add an HTLC.
  - for the first HTLC it offers:
    - MUST set `id` to 0.
  - MUST increase the value of `id` by 1 for each successive offer.

`id` MUST NOT be reset to 0 after the update is complete (i.e. after `revoke_and_ack` has
been received). It MUST continue incrementing instead.

> sending node:
> - 現在の `feerate_per_kw`での remote commitment transaction で支払うことができない `amount_msat` を提供してはいけません（"Updating
Fees"を参照）。
> - 0より大きい `amount_msat` を提供する必要があります。
>   - 受信ノードの `htlc_minimum_msat` の下に `amount_msat` を提供してはなりません
>   - `cltv_expiry` を 500000000未満に設定する必要があります。
>   - Bitcoin blockchain を識別する `chain_hash` を持つチャネルの場合：
>     - `amount_msat` の上位4バイトを0に設定しなければなりません。
>   - 結果が remote commitment transaction で、リモートの `max_accepted_htlcs` HTLC より多くを提供する場合：
>     - HTLCを追加しないでください。
>   - offerd HTLC の合計がリモートの `max_htlc_value_in_flight_msat` を超える場合：
>     - HTLCを追加しないでください。
>   - 最初の HTLC の場合：
>     -`id`を0に設定する必要があります。
>   - 連続するオファーごとに `id` の値を1ずつ増やす必要があります。

> 更新が完了した後（つまり、`revoke_and_ack` を受信した後）、`id` を 0 にリセットしてはなりません。代わりにそれを増やし続けなければなりません。

A receiving node:
  - receiving an `amount_msat` equal to 0, OR less than its own `htlc_minimum_msat`:
    - SHOULD fail the channel.
  - receiving an `amount_msat` that the sending node cannot afford at the current `feerate_per_kw` (while maintaining its channel reserve):
    - SHOULD fail the channel.
  - if a sending node adds more than receiver `max_accepted_htlcs` HTLCs to
    its local commitment transaction, OR adds more than receiver `max_htlc_value_in_flight_msat` worth of offered HTLCs to its local commitment transaction:
    - SHOULD fail the channel.
  - if sending node sets `cltv_expiry` to greater or equal to 500000000:
    - SHOULD fail the channel.
  - for channels with `chain_hash` identifying the Bitcoin blockchain, if the four most significant bytes of `amount_msat` are not 0:
    - MUST fail the channel.
  - MUST allow multiple HTLCs with the same `payment_hash`.
  - if the sender did not previously acknowledge the commitment of that HTLC:
    - MUST ignore a repeated `id` value after a reconnection.
  - if other `id` violations occur:
    - MAY fail the channel.

> receiving node:
>   - 0に等しい、または自身が設定した `htlc_minimum_msat` より小さい` amount_msat`を受信した場合：
>     - チャネルに失敗する必要があります。
>   - sending node が現在の `feerate_per_kw` で余裕がない `amount_msat` を受信した場合（チャネル予約を維持します）：
>     - チャネルに失敗する必要があります。
>   - sending node が受信側の `max_accepted_htlcs` HTLCを local commitment transaction に追加する場合、または受信側の `max_htlc_value_in_flight_msat` 相当以上の HTLC を local commitment transaction 追加する場合：
>     - チャネルに失敗する必要があります。
>   - sending node が `cltv_expiry` を500000000以上に設定する場合：
>     - チャネルに失敗する必要があります。
>   - Bitcoin blockchain を識別する `chain_hash` を持つチャネルの場合、 `amount_msat` の最上位4バイトが0でない場合：
>     - チャネルに失敗する必要があります。
>   - 同じ `payment_hash` を持つ複数のHTLCを許可する必要があります。
>     - 送信者が以前にその HTLC の実行を確認しなかった場合：
>   - 再接続後に繰り返される `id` 値を無視する必要があります。
>     - チャネルが失敗する場合があります。

The `onion_routing_packet` contains an obfuscated list of hops and instructions for each hop along the path.
It commits to the HTLC by setting the `payment_hash` as associated data, i.e. includes the `payment_hash` in the computation of HMACs.
This prevents replay attacks that would reuse a previous `onion_routing_packet` with a different `payment_hash`.

> `onion_routing_packet` には、難読化されたホップのリストと、パスに沿った各ホップの指示が含まれています。 関連するデータとして `payment_hash` を設定することで HTLC にコミットします。つまり、HMAC の計算に `payment_hash` を含めます。 これにより、以前の `onion_routing_packet` を別の `payment_hash`で再利用するリプレイ攻撃を防ぎます。

#### Rationale

Invalid amounts are a clear protocol violation and indicate a breakdown.

If a node did not accept multiple HTLCs with the same payment hash, an
attacker could probe to see if a node had an existing HTLC. This
requirement, to deal with duplicates, leads to the use of a separate
identifier; it's assumed a 64-bit counter never wraps.

Retransmissions of unacknowledged updates are explicitly allowed for
reconnection purposes; allowing them at other times simplifies the
recipient code (though strict checking may help debugging).

`max_accepted_htlcs` is limited to 483 to ensure that, even if both
sides send the maximum number of HTLCs, the `commitment_signed` message will
still be under the maximum message size. It also ensures that
a single penalty transaction can spend the entire commitment transaction,
as calculated in [BOLT #5](05-onchain.md#penalty-transaction-weight-calculation).

`cltv_expiry` values equal to or greater than 500000000 would indicate a time in
seconds, and the protocol only supports an expiry in blocks.

`amount_msat` is deliberately limited for this version of the
specification; larger amounts are not necessary, nor wise, during the
bootstrap phase of the network.

> 無効な金額は明確なプロトコル違反であり、故障を示しています。

> ノードが同じ支払いハッシュを持つ複数の HTLC を受け入れなかった場合、攻撃者はノードに既存の HTLC があるかどうかを調べることができます。重複を処理するためのこの要件は、個別の識別子の使用につながります。 64ビットカウンターがラップしないことを前提としています。

> 未接続の更新の再送信は、再接続の目的で明示的に許可されます。他のときにそれらを許可すると、受信者コードが簡素化されます（ただし、厳密なチェックがデバッグに役立つ場合があります）。

> `max_accepted_htlcs` は 483 に制限されており、双方が最大数の HTLC を送信した場合でも、`commitment_signed` メッセージは最大メッセージサイズ未満のままです。また、[BOLT＃5]（05-onchain.md＃penalty-transaction-weight-calculation）で計算されているように、単一の penalty transaction が commitment transaction 全体を使用できるようにします。

> 500000000以上の `cltv_expiry` 値は秒単位の時間を示し、プロトコルはブロック単位の有効期限のみをサポートします。

> このバージョンの仕様では、`amount_msat` は意図的に制限されています。ネットワークのブートストラップ段階では、これ以上の量は必要ありませんし、賢明でもありません。

### Removing an HTLC: `update_fulfill_htlc`, `update_fail_htlc`, and `update_fail_malformed_htlc`

For simplicity, a node can only remove HTLCs added by the other node.
There are four reasons for removing an HTLC: the payment preimage is supplied,
it has timed out, it has failed to route, or it is malformed.

> 簡単にするために、ノードは他のノードによって追加された HTLC のみを削除できます。 HTLCを削除する理由は4つあります。preimage が提供された、タイムアウトした、ルーティングに失敗した、または不正な形式の4つです。

To supply the preimage:

1. type: 130 (`update_fulfill_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`32*byte`:`payment_preimage`]

For a timed out or route-failed HTLC:

1. type: 131 (`update_fail_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`u16`:`len`]
   * [`len*byte`:`reason`]

The `reason` field is an opaque encrypted blob for the benefit of the
original HTLC initiator, as defined in [BOLT #4](04-onion-routing.md);
however, there's a special malformed failure variant for the case where
the peer couldn't parse it: in this case the current node instead takes action, encrypting
it into a `update_fail_htlc` for relaying.

> `reason` フィールドは、[BOLT＃4]（04-onion-routing.md）で定義されているように、元の　HTLC 作成者の利益のために不透明な暗号化されたblobです。 しかし、ピアがそれを解析できなかった場合のために、特別な不正な形式の障害バリアントがあります。この場合、現在のノードは代わりにアクションを実行し、中継のために `update_fail_htlc` に暗号化します。


For an unparsable HTLC:

1. type: 135 (`update_fail_malformed_htlc`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`id`]
   * [`sha256`:`sha256_of_onion`]
   * [`u16`:`failure_code`]

#### Requirements

A node:
  - SHOULD remove an HTLC as soon as it can.
  - SHOULD fail an HTLC which has timed out.
  - until the corresponding HTLC is irrevocably committed in both sides'
  commitment transactions:
    - MUST NOT send an `update_fulfill_htlc`, `update_fail_htlc`, or
`update_fail_malformed_htlc`.

> node:
>   - できるだけ早く HTLC を削除する必要があります。
>   - タイムアウトした HTLC に失敗する必要があります。
>   - 対応する HTLC が両サイドの commitment transactions で irrevocably committed されるまで：
>     - `update_fulfill_htlc`, `update_fail_htlc`、または `update_fail_malformed_htlc` を送信してはいけません。

A receiving node:
  - if the `id` does not correspond to an HTLC in its current commitment transaction:
    - MUST fail the channel.
  - if the `payment_preimage` value in `update_fulfill_htlc`
  doesn't SHA256 hash to the corresponding HTLC `payment_hash`:
    - MUST fail the channel.
  - if the `BADONION` bit in `failure_code` is not set for
  `update_fail_malformed_htlc`:
    - MUST fail the channel.
  - if the `sha256_of_onion` in `update_fail_malformed_htlc` doesn't match the
  onion it sent:
    - MAY retry or choose an alternate error response.
  - otherwise, a receiving node which has an outgoing HTLC canceled by `update_fail_malformed_htlc`:
    - MUST return an error in the `update_fail_htlc` sent to the link which
      originally sent the HTLC, using the `failure_code` given and setting the
      data to `sha256_of_onion`.

> receiving node:
>   - `id` が現在の commitment transaction の HTLC に対応しない場合：
>     - チャネルに失敗する必要があります。
>   - `update_fulfill_htlc` の `payment_preimage`値が、対応するHTLC `payment_hash` にSHA256 しない場合：
>     - チャネルに失敗する必要があります。
>   - `update_fail_malformed_htlc` に `failure_code` の `BADONION` bit が設定されていない場合：
>     - チャネルに失敗する必要があります。
>   - `update_fail_malformed_htlc` の `sha256_of_onion` が送信した onion と一致しない場合：
>     - 再試行するか、代替エラー応答を選択することができます。
>   - それ以外の場合、`update_fail_malformed_htlc` によってキャンセルされた発信HTLCを持つ受信ノード：
>     - 指定された `failure_code` を使用してデータを `sha256_of_onion` に設定し、HTLC を最初に送信したリンクに送信された `update_fail_htlc` でエラーを返さなければなりません。

#### Rationale

A node that doesn't time out HTLCs risks channel failure (see
[`cltv_expiry_delta` Selection](#cltv_expiry_delta-selection)).

A node that sends `update_fulfill_htlc`, before the sender, is also
committed to the HTLC and risks losing funds.

If the onion is malformed, the upstream node won't be able to extract
the shared key to generate a response — hence the special failure message, which
makes this node do it.

The node can check that the SHA256 that the upstream is complaining about
does match the onion it sent, which may allow it to detect random bit
errors. However, without re-checking the actual encrypted packet sent,
it won't know whether the error was its own or the remote's; so
such detection is left as an option.

> HTLC をタイムアウトしないノードは、チャネル障害のリスクがあります（[`cltv_expiry_delta`選択]（＃cltv_expiry_delta-selection）を参照）。

> sender の前に `update_fulfill_htlc` を送信するノードも HTLC が実行され、資金を失うリスクがあります。

> onion の形式が正しくない場合、上流ノードは共有キーを抽出して応答を生成できません。そのため、このノードが実行する特別な失敗メッセージです。

> ノードは、上流が不満を言う SHA256が 送信した onion と一致することを確認できます。これにより、ランダムビットエラーを検出できる場合があります。 ただし、送信された実際の暗号化されたパケットを再チェックしないと、エラーがそれ自身のものであるかリモートのものであるかがわかりません。 そのため、このような検出はオプションとして残されています。

### Committing Updates So Far: `commitment_signed`

When a node has changes for the remote commitment, it can apply them,
sign the resulting transaction (as defined in [BOLT #3](03-transactions.md)), and send a
`commitment_signed` message.

> ノードに remote commitment の変更がある場合、それらを適用し、結果のトランザクションに署名し（[BOLT＃3]（03-transactions.md）で定義）、 `commitment_signed`メッセージを送信できます。

1. type: 132 (`commitment_signed`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`signature`:`signature`]
   * [`u16`:`num_htlcs`]
   * [`num_htlcs*signature`:`htlc_signature`]

#### Requirements

A sending node:
  - MUST NOT send a `commitment_signed` message that does not include any
updates.
  - MAY send a `commitment_signed` message that only
alters the fee.
  - MAY send a `commitment_signed` message that doesn't
change the commitment transaction aside from the new revocation number
(due to dust, identical HTLC replacement, or insignificant or multiple
fee changes).
  - MUST include one `htlc_signature` for every HTLC transaction corresponding
    to the ordering of the commitment transaction (see [BOLT #3](03-transactions.md#transaction-input-and-output-ordering)).
  - if it has not recently received a message from the remote node:
      - SHOULD use `ping` and await the reply `pong` before sending `commitment_signed`.

> sending node:
>   - アップデートを含まない `commitment_signed` メッセージを送信してはなりません。
>   - `commitment_signed` メッセージを送信して料金を変更します。
>   - 新しい取り消し番号を除いて、 commitment transaction を変更しない `commitment_signed` メッセージを送信することができます（dust、 同一の HTLC 交換、または重要でないまたは複数の料金変更）。
>   - コミットメントトランザクションの順序に対応するHTLCトランザクションごとに1つの「htlc_signature」を含める必要があります（[BOLT＃3]（03-transactions.md transaction-input-and-output-ordering）を参照）。
>   - remote node からメッセージを最近受信していない場合：
>     - `ping` を使用し、`commitment_signed` を送信する前に `pong` の返信を待つ必要があります。

A receiving node:
  - once all pending updates are applied:
    - if `signature` is not valid for its local commitment transaction:
      - MUST fail the channel.
    - if `num_htlcs` is not equal to the number of HTLC outputs in the local
    commitment transaction:
      - MUST fail the channel.
  - if any `htlc_signature` is not valid for the corresponding HTLC transaction:
    - MUST fail the channel.
  - MUST respond with a `revoke_and_ack` message.
 
> receiving node:
>   - すべての保留中の更新が適用されると：
>     - `signature` が local commitment transaction に対して有効でない場合：
>       - チャネルに失敗する必要があります。
>     - `num_htlcs` が local commitment transaction の HTLC outputs の数と等しくない場合：
>       - チャネルに失敗する必要があります。
>   - 対応する HTLC transaction に対して `htlc_signature` が有効でない場合：
>     - チャネルに失敗する必要があります。
>   - `revoke_and_ack` メッセージで応答する必要があります。

#### Rationale

There's little point offering spam updates: it implies a bug.

The `num_htlcs` field is redundant, but makes the packet length check fully self-contained.

The recommendation to require recent messages recognizes the reality
that networks are unreliable: nodes might not realize their peers are
offline until after sending `commitment_signed`.  Once
`commitment_signed` is sent, the sender considers itself bound to
those HTLCs, and cannot fail the related incoming HTLCs until the
output HTLCs are fully resolved.

Note that the `htlc_signature` implicitly enforces the time-lock mechanism in the case of offered HTLCs being timed out or received HTLCs being spent. This is done to reduce fees by creating smaller scripts compared to explicitly stating time-locks on HTLC outputs.

> スパムの更新を提供する意味はほとんどありません。これはバグを意味します。

> `num_htlcs`フィールドは冗長ですが、パケット長のチェックは完全に自己完結しています。

> 最近のメッセージを要求する推奨事項は、ネットワークが信頼できないという現実を認識しています。ノードは、`commitment_signed` を送信するまで、ピアがオフラインであることを認識しない可能性があります。 一度 `commitment_signed` が送信され、送信者は自分自身がそれらの HTLC 、および関連する HTLC が失敗するまで
output HTLC は完全に解決されます。

> `htlc_signature` は、提供されたHTLCがタイムアウトした場合、または受信したHTLCが消費された場合に、暗黙的にタイムロックメカニズムを実施することに注意してください。 これは、HTLC outputs のタイムロックを明示的に示すのに比べて、より小さなスクリプトを作成することにより、手数料を削減するために行われます。

### Completing the Transition to the Updated State: `revoke_and_ack`

Once the recipient of `commitment_signed` checks the signature and knows
it has a valid new commitment transaction, it replies with the commitment
preimage for the previous commitment transaction in a `revoke_and_ack`
message.

This message also implicitly serves as an acknowledgment of receipt
of the `commitment_signed`, so this is a logical time for the `commitment_signed` sender
to apply (to its own commitment) any pending updates it sent before
that `commitment_signed`.

The description of key derivation is in [BOLT #3](03-transactions.md#key-derivation).

> `commitment_signed` の受信者が署名を確認し、有効な新しい commitment transaction があることを知ると、`revoke_and_ack` メッセージで以前の commitment transaction の commitment preimage で応答します。

> このメッセージは暗黙的に `commitment_signed` の受信確認としても機能するため、これは `commitment_signed` 送信者が `commitment_signed` の前に送信した保留中の更新を（独自のコミットメントに）適用する論理的な時間です。

> キー派生の説明は[BOLT＃3]（03-transactions.md＃key-derivation）にあります。

1. type: 133 (`revoke_and_ack`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`32*byte`:`per_commitment_secret`]
   * [`point`:`next_per_commitment_point`]

#### Requirements

A sending node:
  - MUST set `per_commitment_secret` to the secret used to generate keys for
  the previous commitment transaction.
  - MUST set `next_per_commitment_point` to the values for its next commitment
  transaction.

> sending node:
>   - `per_commitment_secret` を前の commitment transaction のキーを生成するために使用されるシークレットに設定する必要があります。
>   - `next_per_commitment_point`を次の commitment transaction の値に設定する必要があります。


A receiving node:
  - if `per_commitment_secret` does not generate the previous `per_commitment_point`:
    - MUST fail the channel.
  - if the `per_commitment_secret` was not generated by the protocol in [BOLT #3](03-transactions.md#per-commitment-secret-requirements):
    - MAY fail the channel.

> receiving node:
>   - `per_commitment_secret` が以前の `per_commitment_point` を生成しない場合：
>     - チャネルに失敗する必要があります。
>   - `per_commitment_secret` が[BOLT＃3]のプロトコルによって生成されなかった場合（03-transactions.md＃per-commitment-secret要件）：
>     - チャネルが失敗する場合があります。

A node:
  - MUST NOT broadcast old (revoked) commitment transactions,
    - Note: doing so will allow the other node to seize all channel funds.
  - SHOULD NOT sign commitment transactions, unless it's about to broadcast
  them (due to a failed connection),
    - Note: this is to reduce the above risk.

> node:
>   - 古い (revoked) commitment transactions をブロードキャストしてはいけません。
>     - 注：これを行うと、他のノードがすべてのチャネル資金を獲得できます。
>   - （失敗した接続のために）ブロードキャストしようとしない限り、commitment transactions に署名しないでください。
>     - 注：これは上記のリスクを減らすためです。

### Updating Fees: `update_fee`

An `update_fee` message is sent by the node which is paying the
Bitcoin fee. Like any update, it's first committed to the receiver's
commitment transaction and then (once acknowledged) committed to the
sender's. Unlike an HTLC, `update_fee` is never closed but simply
replaced.

There is a possibility of a race, as the recipient can add new HTLCs
before it receives the `update_fee`. Under this circumstance, the sender may
not be able to afford the fee on its own commitment transaction, once the `update_fee`
is finally acknowledged by the recipient. In this case, the fee will be less
than the fee rate, as described in [BOLT #3](03-transactions.md#fee-payment).

The exact calculation used for deriving the fee from the fee rate is
given in [BOLT #3](03-transactions.md#fee-calculation).

> `update_fee` メッセージは、Bitcoin 手数料を支払っているノードによって送信されます。 他の更新と同様に、最初に receiver's commitment transaction に実行され、次に（一度確認されると）sender に実行されます。 HTLC とは異なり、`update_fee` は決して閉じられず、単に置き換えられます。

> 受信者は `update_fee` を受信する前に新しい HTLC を追加できるため、レースの可能性があります。 この状況下では、`update_fee` が最終的に受信者によって確認されると、送信者は自身の commitment transaction に料金を支払うことができない場合があります。 この場合、[BOLT＃3]（03 Transactions.md＃fee-payment）で説明されているように、料金は料金レートより低くなります。

> 料金レートから料金を導出するために使用される正確な計算は、[BOLT＃3]（03-transactions.md＃fee-calculation）に記載されています。

1. type: 134 (`update_fee`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u32`:`feerate_per_kw`]

#### Requirements

The node _responsible_ for paying the Bitcoin fee:
  - SHOULD send `update_fee` to ensure the current fee rate is sufficient (by a
      significant margin) for timely processing of the commitment transaction.

The node _not responsible_ for paying the Bitcoin fee:
  - MUST NOT send `update_fee`.

> The node _responsible_ for paying the Bitcoin fee:
>   - `update_fee` を送信して、 commitment transaction をタイムリーに処理するために現在の料金率が（かなりのマージンで）十分であることを確認する必要があります。

> The node _not responsible_ for paying the Bitcoin fee:
>   - `update_fee`を送信してはいけません。

A receiving node:
  - if the `update_fee` is too low for timely processing, OR is unreasonably large:
    - SHOULD fail the channel.
  - if the sender is not responsible for paying the Bitcoin fee:
    - MUST fail the channel.
  - if the sender cannot afford the new fee rate on the receiving node's
  current commitment transaction:
    - SHOULD fail the channel,
      - but MAY delay this check until the `update_fee` is committed.

> receiving node:
>   - `update_fee` がタイムリーに処理するには低すぎる場合、またはORが不当に大きい場合：
>     - チャネルに失敗する必要があります。
>   - 送信者がビットコイン手数料の支払いに責任を負わない場合：
>     - チャネルに失敗する必要があります。
>   - 送信者が受信ノードの現在の commitment transaction で新しい料金レートを支払うことができない場合：
>     - チャネルに失敗する必要があります。
>        - しかし、`update_fee` が実行されるまでこのチェックを遅らせることができます。

#### Rationale

Bitcoin fees are required for unilateral closes to be effective —
particularly since there is no general method for the broadcasting node to use
child-pays-for-parent to increase its effective fee.

Given the variance in fees, and the fact that the transaction may be
spent in the future, it's a good idea for the fee payer to keep a good
margin (say 5x the expected fee requirement); but, due to differing methods of
fee estimation, an exact value is not specified.

Since the fees are currently one-sided (the party which requested the
channel creation always pays the fees for the commitment transaction),
it's simplest to only allow it to set fee levels; however, as the same
fee rate applies to HTLC transactions, the receiving node must also
care about the reasonableness of the fee.

> ビットコイン手数料は、一方的な成約が効果的であるために必要です。特に、ブロードキャストノードが有効な手数料を増やすために child-pays-for-paren を使用する一般的な方法がないためです。

> 手数料のばらつき、および取引が将来費やされる可能性があるという事実を考えると、手数料支払者が十分なマージンを維持することをお勧めします（予想される手数料の5倍など）。 ただし、料金の見積もり方法が異なるため、正確な値は指定されていません。

> 現在、料金は一方的であるため（チャネルの作成を要求した当事者は常に commitment transaction の料金を支払います）、料金レベルの設定のみを許可するのが最も簡単です。 ただし、HTLC transactions には同じ料金が適用されるため、受信ノードも手数料の妥当性に注意する必要があります。

## Message Retransmission

Because communication transports are unreliable, and may need to be
re-established from time to time, the design of the transport has been
explicitly separated from the protocol.

Nonetheless, it's assumed our transport is ordered and reliable.
Reconnection introduces doubt as to what has been received, so there are
explicit acknowledgments at that point.

This is fairly straightforward in the case of channel establishment
and close, where messages have an explicit order, but during normal
operation, acknowledgments of updates are delayed until the
`commitment_signed` / `revoke_and_ack` exchange; so it cannot be assumed
that the updates have been received. This also means that the receiving
node only needs to store updates upon receipt of `commitment_signed`.

Note that messages described in [BOLT #7](07-routing-gossip.md) are
independent of particular channels; their transmission requirements
are covered there, and besides being transmitted after `init` (as all
messages are), they are independent of requirements here.

1. type: 136 (`channel_reestablish`)
2. data:
   * [`channel_id`:`channel_id`]
   * [`u64`:`next_commitment_number`]
   * [`u64`:`next_revocation_number`]
   * [`32*byte`:`your_last_per_commitment_secret`] (option_data_loss_protect,option_static_remotekey)
   * [`point`:`my_current_per_commitment_point`] (option_data_loss_protect,option_static_remotekey)

`next_commitment_number`: A commitment number is a 48-bit
incrementing counter for each commitment transaction; counters
are independent for each peer in the channel and start at 0.
They're only explicitly relayed to the other node in the case of
re-establishment, otherwise they are implicit.


### Requirements

A funding node:
  - upon disconnection:
    - if it has broadcast the funding transaction:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

A non-funding node:
  - upon disconnection:
    - if it has sent the `funding_signed` message:
      - MUST remember the channel for reconnection.
    - otherwise:
      - SHOULD NOT remember the channel for reconnection.

A node:
  - MUST handle continuation of a previous channel on a new encrypted transport.
  - upon disconnection:
    - MUST reverse any uncommitted updates sent by the other side (i.e. all
    messages beginning with `update_` for which no `commitment_signed` has
    been received).
      - Note: a node MAY have already used the `payment_preimage` value from
    the `update_fulfill_htlc`, so the effects of `update_fulfill_htlc` are not
    completely reversed.
  - upon reconnection:
    - if a channel is in an error state:
      - SHOULD retransmit the error packet and ignore any other packets for
      that channel.
    - otherwise:
      - MUST transmit `channel_reestablish` for each channel.
      - MUST wait to receive the other node's `channel_reestablish`
        message before sending any other messages for that channel.

The sending node:
  - MUST set `next_commitment_number` to the commitment number of the
  next `commitment_signed` it expects to receive.
  - MUST set `next_revocation_number` to the commitment number of the
  next `revoke_and_ack` message it expects to receive.
  - if `option_static_remotekey` applies to the commitment transaction:
    - MUST set `my_current_per_commitment_point` to a valid point.
  - otherwise, if it supports `option_data_loss_protect`:
    - MUST set `my_current_per_commitment_point` to its commitment point for
      the last signed commitment it received from its channel peer (i.e. the commitment_point 
      corresponding to the commitment transaction the sender would use to unilaterally close).
  - if `option_static_remotekey` applies to the commitment transaction, or the sending node supports `option_data_loss_protect`:
    - if `next_revocation_number` equals 0:
      - MUST set `your_last_per_commitment_secret` to all zeroes
    - otherwise:
      - MUST set `your_last_per_commitment_secret` to the last `per_commitment_secret`
    it received

A node:
  - if `next_commitment_number` is 1 in both the `channel_reestablish` it
  sent and received:
    - MUST retransmit `funding_locked`.
  - otherwise:
    - MUST NOT retransmit `funding_locked`.
  - upon reconnection:
    - MUST ignore any redundant `funding_locked` it receives.
  - if `next_commitment_number` is equal to the commitment number of
  the last `commitment_signed` message the receiving node has sent:
    - MUST reuse the same commitment number for its next `commitment_signed`.
  - otherwise:
    - if `next_commitment_number` is not 1 greater than the
  commitment number of the last `commitment_signed` message the receiving
  node has sent:
      - SHOULD fail the channel.
    - if it has not sent `commitment_signed`, AND `next_commitment_number`
    is not equal to 1:
      - SHOULD fail the channel.
  - if `next_revocation_number` is equal to the commitment number of
  the last `revoke_and_ack` the receiving node sent, AND the receiving node
  hasn't already received a `closing_signed`:
    - MUST re-send the `revoke_and_ack`.
  - otherwise:
    - if `next_revocation_number` is not equal to 1 greater than the
    commitment number of the last `revoke_and_ack` the receiving node has sent:
      - SHOULD fail the channel.
    - if it has not sent `revoke_and_ack`, AND `next_revocation_number`
    is not equal to 0:
      - SHOULD fail the channel.

 A receiving node:
  - if `option_static_remotekey` applies to the commitment transaction:
    - if `next_revocation_number` is greater than expected above, AND
    `your_last_per_commitment_secret` is correct for that
    `next_revocation_number` minus 1:
      - MUST NOT broadcast its commitment transaction.
      - SHOULD fail the channel.
    - otherwise:
	  - if `your_last_per_commitment_secret` does not match the expected values:
        - SHOULD fail the channel.
  - otherwise, if it supports `option_data_loss_protect`, AND the `option_data_loss_protect`
  fields are present:
    - if `next_revocation_number` is greater than expected above, AND
    `your_last_per_commitment_secret` is correct for that
    `next_revocation_number` minus 1:
      - MUST NOT broadcast its commitment transaction.
      - SHOULD fail the channel.
      - SHOULD store `my_current_per_commitment_point` to retrieve funds
        should the sending node broadcast its commitment transaction on-chain.
    - otherwise (`your_last_per_commitment_secret` or `my_current_per_commitment_point`
    do not match the expected values):
      - SHOULD fail the channel.

A node:
  - MUST NOT assume that previously-transmitted messages were lost,
    - if it has sent a previous `commitment_signed` message:
      - MUST handle the case where the corresponding commitment transaction is
      broadcast at any time by the other side,
        - Note: this is particularly important if the node does not simply
        retransmit the exact `update_` messages as previously sent.
  - upon reconnection:
    - if it has sent a previous `shutdown`:
      - MUST retransmit `shutdown`.

### Rationale

The requirements above ensure that the opening phase is nearly
atomic: if it doesn't complete, it starts again. The only exception
is if the `funding_signed` message is sent but not received. In
this case, the funder will forget the channel, and presumably open
a new one upon reconnection; meanwhile, the other node will eventually forget
the original channel, due to never receiving `funding_locked` or seeing
the funding transaction on-chain.

There's no acknowledgment for `error`, so if a reconnect occurs it's
polite to retransmit before disconnecting again; however, it's not a MUST,
because there are also occasions where a node can simply forget the
channel altogether.

`closing_signed` also has no acknowledgment so must be retransmitted
upon reconnection (though negotiation restarts on reconnection, so it needs
not be an exact retransmission).
The only acknowledgment for `shutdown` is `closing_signed`, so one or the other
needs to be retransmitted.

The handling of updates is similarly atomic: if the commit is not
acknowledged (or wasn't sent) the updates are re-sent. However, it's not
insisted they be identical: they could be in a different order,
involve different fees, or even be missing HTLCs which are now too old
to be added. Requiring they be identical would effectively mean a
write to disk by the sender upon each transmission, whereas the scheme
here encourages a single persistent write to disk for each
`commitment_signed` sent or received.

A re-transmittal of `revoke_and_ack` should never be asked for after a
`closing_signed` has been received, since that would imply a shutdown has been
completed — which can only occur after the `revoke_and_ack` has been received
by the remote node.

Note that the `next_commitment_number` starts at 1, since
commitment number 0 is created during opening.
`next_revocation_number` will be 0 until the
`commitment_signed` for commitment number 1 is send and then
the revocation for commitment number 0 is received.

`funding_locked` is implicitly acknowledged by the start of normal
operation, which is known to have begun after a `commitment_signed` has been
received — hence, the test for a `next_commitment_number` greater
than 1.

A previous draft insisted that the funder "MUST remember ...if it has
broadcast the funding transaction, otherwise it MUST NOT": this was in
fact an impossible requirement. A node must either firstly commit to
disk and secondly broadcast the transaction or vice versa. The new
language reflects this reality: it's surely better to remember a
channel which hasn't been broadcast than to forget one which has!
Similarly, for the fundee's `funding_signed` message: it's better to
remember a channel that never opens (and times out) than to let the
funder open it while the fundee has forgotten it.

`option_data_loss_protect` was added to allow a node, which has somehow fallen behind
(e.g. has been restored from old backup), to detect that it's fallen-behind. A fallen-behind
node must know it cannot broadcast its current commitment transaction — which would lead to
total loss of funds — as the remote node can prove it knows the
revocation preimage. The error returned by the fallen-behind node
(or simply the invalid numbers in the `channel_reestablish` it has
sent) should make the other node drop its current commitment
transaction to the chain. This will, at least, allow the fallen-behind node to recover
non-HTLC funds, if the `my_current_per_commitment_point`
is valid. However, this also means the fallen-behind node has revealed this
fact (though not provably: it could be lying), and the other node could use this to
broadcast a previous state.

`option_static_remotekey` removes the changing `to_remote` key,
so the `my_current_per_commitment_point` is unnecessary and thus
ignored (for parsing simplicity, it remains and must be a valid point,
however), but the disclosure of previous secret still allows
fall-behind detection.  An implementation can offer both, however, and
fall back to the `option_data_loss_protect` behavior if
`option_static_remotekey` is not negotiated.

# Authors

[ FIXME: Insert Author List ]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
