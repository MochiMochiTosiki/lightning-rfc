# BOLT #5: Recommendations for On-chain Transaction Handling

## Abstract

Lightning allows for two parties (a local node and a remote node) to conduct transactions
off-chain by giving each of the parties a *cross-signed commitment transaction*,
which describes the current state of the channel (basically, the current balance).
This *commitment transaction* is updated every time a new payment is made and
is spendable at all times.

> Lightning では、2つのパーティ（ローカルノードとリモートノード）が、各パーティにチャネルの現在の状態（基本的には現在のバランス）を記述する *cross-signed commitment transaction* を与えることにより、off-chain でトランザクションを実行できます。。 この*commitment transaction*は、新しい支払いが行われるたびに更新され、常に使用可能です。

There are three ways a channel can end:

1. The good way (*mutual close*): at some point the local and remote nodes agree
to close the channel. They generate a *closing transaction* (which is similar to a
commitment transaction, but without any pending payments) and publish it on the
blockchain (see [BOLT #2: Channel Close](02-peer-protocol.md#channel-close)).
2. The bad way (*unilateral close*): something goes wrong, possibly without evil
intent on either side. Perhaps one party crashed, for instance. One side
publishes its *latest commitment transaction*.
3. The ugly way (*revoked transaction close*): one of the parties deliberately
tries to cheat, by publishing an *outdated commitment transaction* (presumably,
a prior version, which is more in its favor).

> チャネルを終了するには3つの方法があります：
> 1. The good way（*mutual close*）： ある時点で、local と remote node はチャネルを閉じることに同意します。 彼らは *closing transaction* （commitment transaction に似ていますが、保留中の支払いはありません）を生成し、ブロックチェーンで公開します（[BOLT＃2：Channel Close]（02-peer-protocol.md＃channel-closeを参照） ））。
> 2. The bad way（*unilateral close*）： 何かがおかしくなり、おそらくどちらかの側に悪意がない。 たとえば、ある当事者がクラッシュした可能性があります。 一方は、*latest commitment transaction* を公開します。
> 3. The ugly way（*revoked transaction close*）： パーティの1つが、*outdated commitment transaction* （おそらく、以前のバージョンのほうが有利です）を公開することにより、意図的に不正を試みます。

Because Lightning is designed to be trustless, there is no risk of loss of funds
in any of these three cases; provided that the situation is properly handled.
The goal of this document is to explain exactly how a node should react when it
encounters any of the above situations, on-chain.

> Lightningは trustless に設計されているため、これら3つのケースのいずれにおいて適切に処理されていれば資金を失うリスクはありません。 このドキュメントの目標は、ノードが on-chain で上記の状況のいずれかに遭遇したときのノードの反応を正確に説明することです。

# Table of Contents
  * [General Nomenclature](#general-nomenclature)
  * [Commitment Transaction](#commitment-transaction)
  * [Failing a Channel](#failing-a-channel)
  * [Mutual Close Handling](#mutual-close-handling)
  * [Unilateral Close Handling: Local Commitment Transaction](#unilateral-close-handling-local-commitment-transaction)
      * [HTLC Output Handling: Local Commitment, Local Offers](#htlc-output-handling-local-commitment-local-offers)
      * [HTLC Output Handling: Local Commitment, Remote Offers](#htlc-output-handling-local-commitment-remote-offers)
  * [Unilateral Close Handling: Remote Commitment Transaction](#unilateral-close-handling-remote-commitment-transaction)
      * [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
      * [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
  * [Revoked Transaction Close Handling](#revoked-transaction-close-handling)
	  * [Penalty Transactions Weight Calculation](#penalty-transactions-weight-calculation)
  * [General Requirements](#general-requirements)
  * [Appendix A: Expected Weights](#appendix-a-expected-weights)
	* [Expected Weight of the `to_local` Penalty Transaction Witness](#expected-weight-of-the-to-local-penalty-transaction-witness)
	* [Expected Weight of the `offered_htlc` Penalty Transaction Witness](#expected-weight-of-the-offered-htlc-penalty-transaction-witness)
	* [Expected Weight of the `accepted_htlc` Penalty Transaction Witness](#expected-weight-of-the-accepted-htlc-penalty-transaction-witness)
  * [Authors](#authors)

# General Nomenclature

Any unspent output is considered to be *unresolved* and can be *resolved*
as detailed in this document. Usually this is accomplished by spending it with
another *resolving* transaction. Although, sometimes simply noting the output
for later wallet spending is sufficient, in which case the transaction containing
the output is considered to be its own *resolving* transaction.

Outputs that are *resolved* are considered *irrevocably resolved*
once the remote's *resolving* transaction is included in a block at least 100
deep, on the most-work blockchain. 100 blocks is far greater than the
longest known Bitcoin fork and is the same wait time used for
confirmations of miners' rewards (see [Reference Implementation](https://github.com/bitcoin/bitcoin/blob/4db82b7aab4ad64717f742a7318e3dc6811b41be/src/consensus/tx_verify.cpp#L223)).

> 未使用の outputs は、*unresolved* とみなされ、このドキュメントで詳しく説明されているように *resolved* できます。 通常、これは別の *resolving* transaction で使用することで達成されます。 ただし、後のウォレット支出のために単に output を記録するだけで十分な場合もありますが、その場合、output を含むtransaciton は独自の*resolving* transactionと見なされます。

> remote's *resolving* transaction が最も作業しているブロックチェーンで少なくとも100の深さのブロックに含まれると、*resolved* output は *irrevocably resolved* と見なされます。 100ブロックは、既知の最長のビットコインフォークよりもはるかに大きく、マイナー報酬の確認に使用されるのと同じ待機時間です。

## Requirements

A node:
  - once it has broadcast a funding transaction OR sent a commitment signature
  for a commitment transaction that contains an HTLC output:
    - until all outputs are *irrevocably resolved*:
      - MUST monitor the blockchain for transactions that spend any output that
      is NOT *irrevocably resolved*.
  - MUST *resolve* all outputs, as specified below.
  - MUST be prepared to resolve outputs multiple times, in case of blockchain
  reorganizations.
  - upon the funding transaction being spent, if the channel is NOT already
  closed:
    - SHOULD fail the channel.
    - MAY send a descriptive error packet.
  - SHOULD ignore invalid transactions.

> A node:
> - いったんfunding transaction をブロードキャストするか、HTLC出力を含む commitment transaction の commitment signatureを送信すると：
>   - すべてのoutputが *irrevocably resolved* まで：
>     - *irrevocably resolved* output を費やすトランザクションのブロックチェーンを監視しなければなりません。
> - 以下に指定されているように、すべての outputs を *resolve* する必要があります。
> - ブロックチェーンの再編成の場合、outputs を複数回解決する準備が必要です。
> - funding transaction が費やされ、チャネルがまだない場合
   閉まっている：
>   - チャネルに失敗する必要があります。
>   - 記述的なエラーパケットを送信する場合があります。
> - 無効なトランザクションを無視する必要があります。

## Rationale

Once a local node has some funds at stake, monitoring the blockchain is required
to ensure the remote node does not close unilaterally.

Invalid transactions (e.g. bad signatures) can be generated by anyone,
(and will be ignored by the blockchain anyway), so they should not
trigger any action.

> ローカルノードに何らかの資金が stake されたら、リモートノードが一方的に閉じないようにブロックチェーンを監視する必要があります。

> 無効なトランザクション（不正な署名など）は誰でも生成できるため（とにかくブロックチェーンでは無視されます）、アクションはトリガーされません。

# Commitment Transaction

The local and remote nodes each hold a *commitment transaction*. Each of these
commitment transactions has four types of outputs:

1. _local node's main output_: Zero or one output, to pay to the *local node's*
commitment pubkey.
2. _remote node's main output_: Zero or one output, to pay to the *remote node's*
commitment pubkey.
3. _local node's offered HTLCs_: Zero or more pending payments (*HTLCs*), to pay
the *remote node* in return for a payment preimage.
4. _remote node's offered HTLCs_: Zero or more pending payments (*HTLCs*), to
pay the *local node* in return for a payment preimage.

To incentivize the local and remote nodes to cooperate, an `OP_CHECKSEQUENCEVERIFY`
relative timeout encumbers the *local node's outputs* (in the *local node's
commitment transaction*) and the *remote node's outputs* (in the *remote node's
commitment transaction*). So for example, if the local node publishes its
commitment transaction, it will have to wait to claim its own funds,
whereas the remote node will have immediate access to its own funds. As a
consequence, the two commitment transactions are not identical, but they are
(usually) symmetrical.

See [BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction)
for more details.

> ローカルノードとリモートノードはそれぞれ *commitment transaction* を保持しています。 これらの commitment transactions には、それぞれ4種類の出力があります。 

> 1. _ローカルノードの main output_： *local node's* commitment pubkey に支払うゼロまたは1つのoutput。
> 2. _remoteノードの main output_： *remote node's* commitment pubkey に支払うためのゼロまたは1つのoutput。
> 3. _local node's offered HTLCs_： payment preimage と引き換えに*リモートノード*に支払うためのゼロ以上の保留中の支払（* HTLCs *）。
> 4. _remote node's offered HTLCs_： payment preimage と引き換えに*ローカルノード*に支払うための、ゼロ以上の保留中の支払（* HTLCs）。

> ローカルノードとリモートノードの協力を促すため、 `OP_CHECKSEQUENCEVERIFY` relative timeout は、*local node's outputs* （*local node's
commitment transaction*）と *remote node's outputs* (in the *remote node's commitment transaction*) を妨げます 。 たとえば、ローカルノードがcommitment transaction を発行する場合、リモートノードは自身の資金にすぐにアクセスできるのに対し、ローカルノードは自身の資金を要求するのを待つ必要があります。 結果として、2つの commitment transactions は同一ではありませんが、（通常）対称的です。

> 詳細は [BOLT #3: Commitment Transaction](03-transactions.md#commitment-transaction) を見てください。

# Failing a Channel

Although closing a channel can be accomplished in several ways, the most
efficient is preferred.

Various error cases involve closing a channel. The requirements for sending
error messages to peers are specified in
[BOLT #1: The `error` Message](01-messaging.md#the-error-message).

> チャネルを閉じるにはいくつかの方法がありますが、最も効率的な方法が推奨されます。

> さまざまなエラーの場合には、チャネルを閉じる必要があります。 ピアにエラーメッセージを送信するための要件は、[BOLT＃1：The `error`メッセージ]（01-messaging.md＃the-error-message）で指定されています。

## Requirements

A node:
  - if a *local commitment transaction* has NOT ever contained a `to_local`
  or HTLC output:
    - MAY simply forget the channel.
  - otherwise:
    - if the *current commitment transaction* does NOT contain `to_local` or
    other HTLC outputs:
      - MAY simply wait for the remote node to close the channel.
      - until the remote node closes:
        - MUST NOT forget the channel.
    - otherwise:
      - if it has received a valid `closing_signed` message that includes a
      sufficient fee:
        - SHOULD use this fee to perform a *mutual close*.
      - otherwise:
        - MUST use the *last commitment transaction*, for which it has a
        signature, to perform a *unilateral close*.

> A node：
>   - *local commitment transaction* に `to_local`または HTLC outputが含まれていない場合：
>     - 単にチャンネルを忘れることがあります。
>   - さもないと：
>     - *current commitment transaction* に `to_local`または他の HTLC output が含まれていない場合：
>       - リモートノードがチャネルを閉じるのを単に待つことができます。
>       - リモートノードが閉じるまで：
>         - チャンネルを忘れてはいけません。
>     - さもないと：
>       - `closing_signed`を含む有効なメッセージを受信した場合
>       十分な料金：
>         - この手数料を使用して、 *mutual close* を実行する必要があります。
>       - さもないと：
>         - *last commitment transaction* を使用しなければなりません。
>         署名、*unilateral close*を実行します。

## Rationale

Since `dust_limit_satoshis` is supposed to prevent creation of uneconomic
outputs (which would otherwise remain forever, unspent on the blockchain), all
commitment transaction outputs MUST be spent.

In the early stages of a channel, it's common for one side to have
little or no funds in the channel; in this case, having nothing at stake, a node
need not consume resources monitoring the channel state.

There exists a bias towards preferring mutual closes over unilateral closes,
because outputs of the former are unencumbered by a delay and are directly
spendable by wallets. In addition, mutual close fees tend to be less exaggerated
than those of commitment transactions. So, the only reason not to use the
signature from `closing_signed` would be if the fee offered was too small for
it to be processed.

> `dust_limit_satoshis` は不経済な outputs （そうでなければ永久にブロックチェーンに費やされないままになる）の作成を防ぐことになっているため、すべての commitment transaction outputs を使用する必要があります。

> チャンネルの初期段階では、一方の側がチャンネルにほとんど、またはまったく資金を持たないことが一般的です。 この場合、何も問題がないため、ノードはチャネル状態を監視するリソースを消費する必要はありません。

> 前者の outputs は遅延によって妨げられず、ウォレットによって直接支出されるため、一方的なクローズよりも相互クローズを優先する傾向があります。 さらに、相互成約手数料は、 commitment transactions の手数料よりも誇張されない傾向があります。 したがって、`closing_signed` の署名を使用しない唯一の理由は、提供される料金が少なすぎて処理できない場合です。

# Mutual Close Handling

A closing transaction *resolves* the funding transaction output.

In the case of a mutual close, a node need not do anything else, as it has
already agreed to the output, which is sent to its specified `scriptpubkey` (see
[BOLT #2: Closing initiation: `shutdown`](02-peer-protocol.md#closing-initiation-shutdown)).

> closing transaction は、funding transaction output を `解決` します。

> 相互クローズの場合、ノードは、指定された `scriptpubkey` に送信される output に既に同意しているため、他に何もする必要はありません（[BOLT＃2：終了開始：` shutdown`]（02 -peer-protocol.md＃closing-initiation-shutdown））。

# Unilateral Close Handling: Local Commitment Transaction

This is the first of two cases involving unilateral closes. In this case, a
node discovers its *local commitment transaction*, which *resolves* the funding
transaction output.

However, a node cannot claim funds from the outputs of a unilateral close that
it initiated, until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified
by the remote node's `to_self_delay` field). Where relevant, this situation is
noted below.

> これは、片側閉鎖を伴う2つのケースの最初のケースです。 この場合、ノードはその *local commitment transaction* を発見し、これが funding
transaction output を *解決* します。

> ただし、ノードは、`OP_CHECKSEQUENCEVERIFY` 遅延が経過するまで（リモートノードの `to_self_delay` フィールドで指定される）、開始した一方的な終値の出力から資金を請求することはできません。 関連する場合、この状況を以下に示します。

## Requirements

A node:
  - upon discovering its *local commitment transaction*:
    - SHOULD spend the `to_local` output to a convenient address.
    - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as
    specified by the remote node's `to_self_delay` field) before spending the
    output.
      - Note: if the output is spent (as recommended), the output is *resolved*
      by the spending transaction, otherwise it is considered *resolved* by the
      commitment transaction itself.
    - MAY ignore the `to_remote` output.
      - Note: No action is required by the local node, as `to_remote` is
      considered *resolved* by the commitment transaction itself.
    - MUST handle HTLCs offered by itself as specified in
    [HTLC Output Handling: Local Commitment, Local Offers](#htlc-output-handling-local-commitment-local-offers).
    - MUST handle HTLCs offered by the remote node as
    specified in [HTLC Output Handling: Local Commitment, Remote Offers](#htlc-output-handling-local-commitment-remote-offers).

> A node:
>   - *local commitment transaction* の発見時：
>     - `to_local` output を便利なアドレスに費やす必要があります。
>     - output を消費する前に、（`OP_CHECKSEQUENCEVERIFY` 遅延（リモートノードの `to_self_delay`フィールドで指定）が経過するまで待機する必要があります。
>       - 注： output が（推奨されるように）費やされる場合、output は spending transaction によって*解決*されます。それ以外の場合、 commitment transaction 自体によって*解決*と見なされます。
>     - `to_remote` outpput を無視してもよい。
>       - 注： `to_remote` は commitment transaction 自体によって*解決*されていると見なされるため、ローカルノードによるアクションは不要です。
>     - [HTLC Output Handling：Local Commitment、Local Offers]（＃htlc output-handling-local-commitment-local-offers）で指定されているように、それ自体が提供するHTLCを処理する必要があります。
>     - [HTLC Output Handling：Local Commitment、Remote Offers]（＃htlc-output-handling-local-commitment-remote-offers）で指定されているように、リモートノードによって提供されるHTLCを処理する必要があります。

## Rationale

Spending the `to_local` output avoids having to remember the complicated
witness script, associated with that particular channel, for later
spending.

The `to_remote` output is entirely the business of the remote node, and
can be ignored.

> `to_local` output を使用することで、後で特定のチャネルに関連付けられた複雑な監視スクリプトを覚えておく必要がなくなります。

> `to_remote` output は完全にリモートノードのビジネスであり、無視できます。

## HTLC Output Handling: Local Commitment, Local Offers

Each HTLC output can only be spent by either the *local offerer*, by using the
HTLC-timeout transaction after it's timed out, or the *remote recipient*, if it
has the payment preimage.

There can be HTLCs which are not represented by any outputs: either
because they were trimmed as dust, or because the transaction has only been
partially committed.

The HTLC output has *timed out* once the depth of the latest block is equal to
or greater than the HTLC `cltv_expiry`.

> 各 HTLC output は、タイムアウト後にHTLC-timeout transaction を使用する *local offerer*、または payment preimage がある場合は* remote受信者*によってのみ使用できます。

> output によって表されない HTLC が存在する可能性があります。それらがダストとしてトリミングされたため、またはトランザクションが部分的にのみコミットされたためです。

> HTLC出力は、最新ブロックの深さがHTLC `cltv_expiry`以上になると*タイムアウト*しました。

### Requirements

A node:
  - if the commitment transaction HTLC output is spent using the payment
  preimage, the output is considered *irrevocably resolved*:
    - MUST extract the payment preimage from the transaction input witness.
  - if the commitment transaction HTLC output has *timed out* and hasn't been
  *resolved*:
    - MUST *resolve* the output by spending it using the HTLC-timeout
    transaction.
    - once the resolving transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
      - MUST resolve the output of that HTLC-timeout transaction.
      - SHOULD resolve the HTLC-timeout transaction by spending it to a
      convenient address.
        - Note: if the output is spent (as recommended), the output is
        *resolved* by the spending transaction, otherwise it is considered
        *resolved* by the HTLC-timeout transaction itself.
      - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as
      specified by the remote node's `open_channel` `to_self_delay` field)
      before spending that HTLC-timeout output.
  - for any committed HTLC that does NOT have an output in this commitment
  transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - if no *valid* commitment transaction contains an output corresponding to
    the HTLC.
      - MAY fail the corresponding incoming HTLC sooner.

> A node：
>  - commitment transaction HTLC output が payment preimage を使用して費やされた場合、出力は *irrevocably resolved* と見なされませす：
>    - transaction input witness から payment preimage を抽出する必要があります。
>  - commitment transaction HTLC output が *timed out* で、*resolve* されていない場合：
>    - commitment transaction HTLC output を使用して、output を *resolve* する必要があります。
>    - resolving transaction が妥当な深さに達すると：
>      - 対応する incoming HTLC（存在する場合）に失敗する必要があります。
>      - そのHTLC-timeout transaction の output を解決する必要があります。
>      - HTLC-timeout transaction を convenient address に送信して解決する必要があります。
>        - 注： output が（推奨されるように）消費されると、output は spending transaction によって *resolved* されます。それ以外の場合、 HTLC-timeout transaction 自体によって *resolved* と見なされます。
>      - その HTLC-timeout output を費やす前に、（`OP_CHECKSEQUENCEVERIFY`遅延（リモートノードの `open_channel` `to_self_delay`フィールドで指定された）が経過するまで待機する必要があります。
>  - commitment transactionで output を持たないコミット済み HTLC の場合：
>    - commitment transaction が妥当な深さに達すると：
>      - 対応する incoming HTLC （存在する場合）に失敗する必要があります。
>    - *valid* commitment transaction にHTLCに対応する output が含まれていない場合。
>      - 対応する incoming HTLC により早く失敗する場合があります。

### Rationale

The payment preimage either serves to prove payment (when the offering node
originated the payment) or to redeem the corresponding incoming HTLC from
another peer (when the offering node is forwarding the payment). Once a node has
extracted the payment, it no longer cares about the fate of the HTLC-spending
transaction itself.

In cases where both resolutions are possible (e.g. when a node receives payment
success after timeout), either interpretation is acceptable; it is the
responsibility of the recipient to spend it before this occurs.

The local HTLC-timeout transaction needs to be used to time out the HTLC (to
prevent the remote node fulfilling it and claiming the funds) before the
local node can back-fail any corresponding incoming HTLC, using
`update_fail_htlc` (presumably with reason `permanent_channel_failure`), as
detailed in
[BOLT #2](02-peer-protocol.md#forwarding-htlcs).
If the incoming HTLC is also on-chain, a node must simply wait for it to
timeout: there is no way to signal early failure.

If an HTLC is too small to appear in *any commitment transaction*, it can be
safely failed immediately. Otherwise, if an HTLC isn't in the *local commitment
transaction*, a node needs to make sure that a blockchain reorganization, or
race, does not switch to a commitment transaction that does contain the HTLC
before the node fails it (hence the wait). The requirement that the incoming
HTLC be failed before its own timeout still applies as an upper bound.

> payment preimage は、支払いを証明するために（オファリングノードが支払いを開始したとき）、または別のピアからの対応する incoming HTLC を引き換える（オファリングノードが支払いを転送しているとき）のいずれかに役立ちます。ノードが支払いを抽出すると、HTLCを使用するトランザクション自体の運命を気にしなくなります。

> 両方の解決が可能な場合（例えば、ノードがタイムアウト後に支払い成功を受け取った場合）、どちらの解釈も受け入れられます;これが発生する前にそれを費やすのは受信者の責任です。

> ローカルノードが `update_fail_htlc` を使用して対応する incoming HTLC をバックフェイルする前に、リモートHTLCタイムアウトトランザクションを使用してHTLCをタイムアウトする必要があります（リモートノードがそれを実行して資金を要求するのを防ぐため） `permanent_channel_failure`）、[BOLT＃2]（02-peer-protocol.md＃forwarding-htlcs）で詳しく説明されています。incoming HTLC もチェーン上にある場合、ノードは単純にタイムアウトするまで待機する必要があります。早期障害を通知する方法はありません。

> HTLCが小さすぎて *commitment transaction* に表示されない場合、すぐに安全に失敗する可能性があります。そうでない場合、HTLCが *local commitment
transaction* にない場合、ノードは、ブロックチェーンの再編成、またはレースが、ノードが失敗する前にHTLCを含む commitment transaction に切り替えないことを確認する必要があります（したがって、待つ）。 incoming HTLC がタイムアウトになる前に失敗するという要件は、上限として適用されます。

## HTLC Output Handling: Local Commitment, Remote Offers

Each HTLC output can only be spent by the recipient, using the HTLC-success
transaction, which it can only populate if it has the payment
preimage. If it doesn't have the preimage (and doesn't discover it), it's
the offerer's responsibility to spend the HTLC output once it's timed out.

> 各HTLC outputs は、HTLC-success transaction を使用して受信者のみが使用できます。HTLC-success transaction は、payment preimage がある場合にのみ入力できます。 payment preimage がない場合（および検出されない場合）、タイムアウトになったら HTLC output を使用するのは提供者の責任です。

There are several possible cases for an offered HTLC:

1. The offerer is NOT irrevocably committed to it. The recipient will usually
   not know the preimage, since it will not forward HTLCs until they're fully
   committed. So using the preimage would reveal that this recipient is the
   final hop; thus, in this case, it's best to allow the HTLC to time out.
2. The offerer is irrevocably committed to the offered HTLC, but the recipient
   has not yet committed to an outgoing HTLC. In this case, the recipient can
   either forward or timeout the offered HTLC.
3. The recipient has committed to an outgoing HTLC, in exchange for the offered
   HTLC. In this case, the recipient must use the preimage, once it receives it
   from the outgoing HTLC; otherwise, it will lose funds by sending an outgoing
   payment without redeeming the incoming payment.


> 提供されるHTLCには、いくつかのケースが考えられます。

> 1. 申し出人は、取消不能な形でそれにコミットしません。 受信者は通常、preimage を認識しません。これは、完全にコミットされるまでHTLCを転送しないためです。 したがって、プリイメージを使用すると、この受信者が最終ホップであることがわかります。 したがって、この場合、HTLCがタイムアウトするのを許可するのが最善です。
> 2. 申し出人は、申し出られたHTLCに取り消すことはできませんが、受信者はまだ outgoing HTLC にコミットしていません。 この場合、受信者は提供されたHTLCを転送またはタイムアウトできます。
> 3. 受信者は、offered HTLC と引き換えに、outgoing HTLC にコミットしました。 この場合、受信側は outgoing HTLC から受信した preimage を使用する必要があります。 そうでない場合、入金を引き換えずに出金を送信することで資金を失います。


### Requirements

A local node:
  - if it receives (or already possesses) a payment preimage for an unresolved
  HTLC output that it has been offered AND for which it has committed to an
  outgoing HTLC:
    - MUST *resolve* the output by spending it, using the HTLC-success
    transaction.
    - MUST resolve the output of that HTLC-success transaction.
  - otherwise:
    - if the *remote node* is NOT irrevocably committed to the HTLC:
      - MUST NOT *resolve* the output by spending it.
  - SHOULD resolve that HTLC-success transaction output by spending it to a
  convenient address.
  - MUST wait until the `OP_CHECKSEQUENCEVERIFY` delay has passed (as specified
    by the *remote node's* `open_channel`'s `to_self_delay` field), before
    spending that HTLC-success transaction output.

If the output is spent (as is recommended), the output is *resolved* by
the spending transaction, otherwise it's considered *resolved* by the HTLC-success
transaction itself.

If it's NOT otherwise resolved, once the HTLC output has expired, it is
considered *irrevocably resolved*.

> A local node：
>  - 提供された、および outgoing HTLC にコミットした unresolved HTLC output の payment preimage を受信する（または既に所有している）場合：
>    - HTLC-success transaction を使用して、出力を使用して*解決*する必要があります。
>    - その HTLC-success transaction の出力を解決する必要があります。
>  - さもないと：
>    - *リモートノード*がHTLCに取消不能にコミットされていない場合：
>      - 出力を使用して*解決*してはなりません。
>  - HTLC-success transaction output を便利なアドレスに送信して解決する必要があります。
>  - HTLC-success transaction output を費やす前に、 `OP_CHECKSEQUENCEVERIFY` （*remote node's* `open_channel`の `to_self_delay`フィールドで指定される）が経過するまで待機する必要があります。

> 出力が費やされる場合（推奨）、出力はspending transaction によって*解決*されます。それ以外の場合、HTLC-success transaction 自体によって*解決*と見なされます。

> それ以外の方法で解決されない場合、HTLC output の有効期限が切れると、*取消不能に解決された*と見なされます。

# Unilateral Close Handling: Remote Commitment Transaction

The *remote node's* commitment transaction *resolves* the funding
transaction output.

There are no delays constraining node behavior in this case, so it's simpler for
a node to handle than the case in which it discovers its local commitment
transaction (see [Unilateral Close Handling: Local Commitment Transaction](#unilateral-close-handling-local-commitment-transaction)).

> The *remote node's* commitment transaction *resolves* the funding transaction output.

> この場合、ノードの動作を制限する遅延はないため、local commitment transaction を検出する場合よりもノードの処理が簡単です（[一方的クローズ処理：ローカルコミットトランザクション]（＃unilateral-close-handling-local -commitment-transaction））。

## Requirements

A local node:
  - upon discovering a *valid* commitment transaction broadcast by a
  *remote node*:
    - if possible:
      - MUST handle each output as specified below.
      - MAY take no action in regard to the associated `to_remote`, which is
      simply a P2WPKH output to the *local node*.
        - Note: `to_remote` is considered *resolved* by the commitment transaction
        itself.
      - MAY take no action in regard to the associated `to_local`, which is a
      payment output to the *remote node*.
        - Note: `to_local` is considered *resolved* by the commitment transaction
        itself.
      - MUST handle HTLCs offered by itself as specified in
      [HTLC Output Handling: Remote Commitment, Local Offers](#htlc-output-handling-remote-commitment-local-offers)
      - MUST handle HTLCs offered by the remote node as specified in
      [HTLC Output Handling: Remote Commitment, Remote Offers](#htlc-output-handling-remote-commitment-remote-offers)
    - otherwise (it is NOT able to handle the broadcast for some reason):
      - MUST send a warning regarding lost funds.

> A local node:：
>  - *リモートノード*によってブロードキャストされた *valid* commitment transaction を発見すると：
>    - 可能なら：
>      - 以下に指定されているように各出力を処理する必要があります。
>      - 関連付けられた`to_remote` に関してアクションを実行しない場合があります。これは、*ローカルノード*へのP2WPKH出力です。
>        - 注：`to_remote` は、commitment transaction自体によって`解決`されたと見なされます。
>      - 関連付けられた`to_local`に関してアクションを実行しない場合があります。これは、*リモートノード*への payment output です。
>        - 注：`to_local`はcommitment transaction 自体によって「解決」されたと見なされます。
>      - [HTLC出力処理：リモートコミットメント、ローカルオファー]（＃htlc-output-handling-remote-commitment-local-offers）で指定されているように、自ら提供するHTLCを処理する必要があります
>      - [HTLC出力処理：リモートコミットメント、リモートオファー]（＃htlc-output-handling-remote-commitment-remote-offers）で指定されているリモートノードによって提供されるHTLCを処理する必要があります。
>    - それ以外の場合（何らかの理由でブロードキャストを処理できません）：
>      - 資金の損失に関する警告を送信する必要があります。

## Rationale

There may be more than one valid, *unrevoked* commitment transaction after a
signature has been received via `commitment_signed` and before the corresponding
`revoke_and_ack`. As such, either commitment may serve as the *remote node's*
commitment transaction; hence, the local node is required to handle both.

In the case of data loss, a local node may reach a state where it doesn't
recognize all of the *remote node's* commitment transaction HTLC outputs. It can
detect the data loss state, because it has signed the transaction, and the
commitment number is greater than expected. If both nodes support
`option_data_loss_protect`, the local node will possess the remote's
`per_commitment_point`, and thus can derive its own `remotepubkey` for the
transaction, in order to salvage its own funds. Note: in this scenario, the node
will be unable to salvage the HTLCs.

> 署名が`commitment_signed`を介して受信された後、対応する`revoke_and_ack`の前に、複数の有効な*unrevoked* commitment transaction が存在する場合があります。 そのため、いずれかのコミットメントが*remote node's* commitment transactionとして機能します。 したがって、両方を処理するには local node が必要です。

> データ損失の場合、ローカルノードは、*remote node's* commitment transaction HTLC outputs のすべてを認識しない状態になる可能性があります。 トランザクションに署名し、コミットメント番号が予想よりも大きいため、データ損失状態を検出できます。 両方のノードが `option_data_loss_protect` をサポートしている場合、ローカルノードはリモートの `per_commitment_point` を所有するため、独自の資金を回収するために、トランザクションに対して独自の `remotepubkey` を導出できます。 > 注：このシナリオでは、ノードはHTLCをサルベージできません。

## HTLC Output Handling: Remote Commitment, Local Offers

Each HTLC output can only be spent by either the *local offerer*, after it's
timed out, or by the *remote recipient*, by using the HTLC-success transaction
if it has the payment preimage.

There can be HTLCs which are not represented by any outputs: either
because the outputs were trimmed as dust, or because the remote node has two
*valid* commitment transactions with differing HTLCs.

The HTLC output has *timed out* once the depth of the latest block is equal to
or greater than the HTLC `cltv_expiry`.

> 各HTLC output は、*local offerer*（タイムアウト後）、または*remote受信者*が、payment preimageがある場合に HTLC-success transaction を使用することによってのみ使用できます。

> 出力によって表されないHTLCが存在する可能性があります。出力がダストとしてトリミングされたか、リモートノードに異なるHTLCを持つ2つの有効なcommitment transactions があるためです。

> HTLC出力は、最新のブロックの深さがHTLC `cltv_expiry`以上になると*タイムアウト*しました。

### Requirements

A local node:
  - if the commitment transaction HTLC output is spent using the payment
  preimage:
    - MUST extract the payment preimage from the HTLC-success transaction input
    witness.
      - Note: the output is considered *irrevocably resolved*.
  - if the commitment transaction HTLC output has *timed out* AND NOT been
  *resolved*:
    - MUST *resolve* the output, by spending it to a convenient address.
  - for any committed HTLC that does NOT have an output in this commitment
  transaction:
    - once the commitment transaction has reached reasonable depth:
      - MUST fail the corresponding incoming HTLC (if any).
    - otherwise:
      - if no *valid* commitment transaction contains an output corresponding to
      the HTLC:
        - MAY fail it sooner.

> A local node:
>   - commitment transaction HTLC output がpayment preimage を使用して費やされる場合：
>     - HTLC-success transaction input witness から payment preimage を抽出する必要があります。
>       - 注： output は *irrevocably resolved* と見なされます。
>   - commitment transaction HTLC output が*タイムアウト*で、*解決*されていない場合：
>     - 便利なアドレスに費やすことにより、output を *resolve* しなければなりません。
>   - このcommitment transaction で output を持たないコミット済みHTLCの場合：
>     - commitment transaction が妥当な深さに達すると：
>       - 対応する incoming HTLC（存在する場合）を失敗させる必要があります。
>     - さもないと：
>       - HTLCに対応する出力が *valid* commitment transaction に含まれていない場合：
>         - より早く失敗する場合があります。

### Rationale

If the commitment transaction belongs to the *remote* node, the only way for it
to spend the HTLC output (using a payment preimage) is for it to use the
HTLC-success transaction.

The payment preimage either serves to prove payment (when the offering node is
the originator of the payment) or to redeem the corresponding incoming HTLC from
another peer (when the offering node is forwarding the payment). After a node has
extracted the payment, it no longer need be concerned with the fate of the
HTLC-spending transaction itself.

In cases where both resolutions are possible (e.g. when a node receives payment
success after timeout), either interpretation is acceptable: it's the
responsibility of the recipient to spend it before this occurs.

Once it has timed out, the local node needs to spend the HTLC output (to prevent
the remote node from using the HTLC-success transaction) before it can
back-fail any corresponding incoming HTLC, using `update_fail_htlc`
(presumably with reason `permanent_channel_failure`), as detailed in
[BOLT #2](02-peer-protocol.md#forwarding-htlcs).
If the incoming HTLC is also on-chain, a node simply waits for it to
timeout, as there's no way to signal early failure.

If an HTLC is too small to appear in *any commitment transaction*, it
can be safely failed immediately. Otherwise,
if an HTLC isn't in the *local commitment transaction* a node needs to make sure
that a blockchain reorganization or race does not switch to a
commitment transaction that does contain it before the node fails it: hence
the wait. The requirement that the incoming HTLC be failed before its
own timeout still applies as an upper bound.

> commitment transaction が*remote*ノードに属している場合、HTLC output を（payment preimageを使用して）使用する唯一の方法は、HTLC-success transaction を使用することです。

> payment preimage は、支払いを証明するために（オファリングノードが支払いの発信者である場合）、または別のピアからの対応する incoming HTLC を引き換えるために（オファリングノードが支払いを転送する場合）のいずれかです。ノードが支払いを抽出した後、HTLCを使用するトランザクション自体の運命を気にする必要はなくなりました。

> 両方の解決策が可能な場合（たとえば、ノードがタイムアウト後に支払い成功を受け取った場合）、どちらの解釈も受け入れられます。これが発生する前にそれを費やすのは受信者の責任です。

> タイムアウトになったら、ローカルノードはHTLC出力を費やして（リモートノードがHTLC成功トランザクションを使用するのを防ぐため）、対応する incoming HTLC を`update_fail_htlc` （おそらく`permanent_channel_failure`）、[BOLT＃2]（02-peer-protocol.md＃forwarding-htlcs）で詳しく説明されています。incoming HTLCもチェーン上にある場合、早期の障害を通知する方法がないため、ノードは単純にタイムアウトするまで待機します。

> HTLCが小さすぎて*任意のコミットメントトランザクション*に表示できない場合、すぐに安全に失敗する可能性があります。そうでなければ、HTLCが *local commitment transaction* にない場合、ノードは、ブロックチェーンの再編成または競合が、ノードが失敗する前にそれを含む commitment transaction に切り替えないことを確認する必要があります。 incoming HTLC がタイムアウトになる前に失敗するという要件は、上限として適用されます。

## HTLC Output Handling: Remote Commitment, Remote Offers

The remote HTLC outputs can only be spent by the local node if it has the
payment preimage. If the local node does not have the preimage (and doesn't
discover it), it's the remote node's responsibility to spend the HTLC output
once it's timed out.

> リモートHTLC出力は、ローカルノードが payment preimage を持っている場合にのみ使用できます。 ローカルノードに reimage がない場合（および検出されない場合）、タイムアウトになったらHTLC output を使用するのはリモートノードの責任です。

There are actually several possible cases for an offered HTLC:

1. The offerer is not irrevocably committed to it. In this case, the recipient
   usually won't know the preimage, since it won't forward HTLCs until
   they're fully committed. As using the preimage would reveal that
   this recipient is the final hop, it's best to allow the HTLC to time out.
2. The offerer is irrevocably committed to the offered HTLC, but the recipient
   hasn't yet committed to an outgoing HTLC. In this case, the recipient can
   either forward it or wait for it to timeout.
3. The recipient has committed to an outgoing HTLC in exchange for an offered
   HTLC. In this case, the recipient must use the preimage, if it receives it
   from the outgoing HTLC; otherwise, it will lose funds by sending an outgoing
   payment without redeeming the incoming one.

> 実際に、提供されるHTLCにはいくつかのケースが考えられます。

> 1. 申し出人は、取消不能な形でそれにコミットしません。 この場合、受信者は通常、HTLCを完全にコミットするまで転送しないため、preimage を認識しません。 preimageを使用すると、この受信者が最終ホップであることが明らかになるため、HTLCがタイムアウトするのを許可するのが最善です。
> 2. 申し出人は、offered HTLC に取り消すことはできませんが、受信者はまだ outgoing HTLC にコミットしていません。 この場合、受信者はそれを転送するか、タイムアウトになるのを待つことができます。
> 3. 受信者は、outgoing HTLC と引き換えに outgoing HTLC にコミットしました。 この場合、受信HTLCから受信する場合、受信者はプリイメージを使用する必要があります。 それ以外の場合は、入金を償還せずに出金を送信することで資金を失います。

### Requirements

A local node:
  - if it receives (or already possesses) a payment preimage for an unresolved
  HTLC output that it was offered AND for which it has committed to an
outgoing HTLC:
    - MUST *resolve* the output by spending it to a convenient address.
  - otherwise:
    - if the remote node is NOT irrevocably committed to the HTLC:
      - MUST NOT *resolve* the output by spending it.

If not otherwise resolved, once the HTLC output has expired, it is considered
*irrevocably resolved*.

# Revoked Transaction Close Handling

If any node tries to cheat by broadcasting an outdated commitment transaction
(any previous commitment transaction besides the most current one), the other
node in the channel can use its revocation private key to claim all the funds from the
channel's original funding transaction.

## Requirements

Once a node discovers a commitment transaction for which *it* has a
revocation private key, the funding transaction output is *resolved*.

A local node:
  - MUST NOT broadcast a commitment transaction for which *it* has exposed the
  `per_commitment_secret`.
  - MAY take no action regarding the _local node's main output_, as this is a
  simple P2WPKH output to itself.
    - Note: this output is considered *resolved* by the commitment transaction
      itself.
  - MUST *resolve* the _remote node's main output_ by spending it using the
  revocation private key.
  - MUST *resolve* the _remote node's offered HTLCs_ in one of three ways:
    * spend the *commitment tx* using the payment revocation private key.
    * spend the *commitment tx* using the payment preimage (if known).
    * spend the *HTLC-timeout tx*, if the remote node has published it.
  - MUST *resolve* the _local node's offered HTLCs_ in one of three ways:
    * spend the *commitment tx* using the payment revocation private key.
    * spend the *commitment tx* once the HTLC timeout has passed.
    * spend the *HTLC-success tx*, if the remote node has published it.
  - MUST *resolve* the _remote node's HTLC-timeout transaction_ by spending it
  using the revocation private key.
  - MUST *resolve* the _remote node's HTLC-success transaction_ by spending it
  using the revocation private key.
  - SHOULD extract the payment preimage from the transaction input witness, if
  it's not already known.
  - MAY use a single transaction to *resolve* all the outputs.
  - MUST handle its transactions being invalidated by HTLC transactions.

## Rationale

A single transaction that resolves all the outputs will be under the
standard size limit because of the 483 HTLC-per-party limit (see
[BOLT #2](02-peer-protocol.md#the-open_channel-message)).

Note: if a single transaction is used, it may be invalidated if the remote node
refuses to broadcast the HTLC-timeout and HTLC-success transactions in a timely
manner. Although, the requirement of persistence until all outputs are
irrevocably resolved, should still protect against this happening. [ FIXME: May have to divide and conquer here, since the remote node may be able to delay the local node long enough to avoid a successful penalty spend? ]

## Penalty Transactions Weight Calculation

There are three different scripts for penalty transactions, with the following
witness weights (details of weight computation are in
[Appendix A](#appendix-a-expected-weights)):

    to_local_penalty_witness: 160 bytes
    offered_htlc_penalty_witness: 243 bytes
    accepted_htlc_penalty_witness: 249 bytes

The penalty *txinput* itself takes up 41 bytes and has a weight of 164 bytes,
which results in the following weights for each input:

    to_local_penalty_input_weight: 324 bytes
    offered_htlc_penalty_input_weight: 407 bytes
    accepted_htlc_penalty_input_weight: 413 bytes

The rest of the penalty transaction takes up 4+1+1+8+1+34+4=53 bytes of
non-witness data: assuming it has a pay-to-witness-script-hash (the largest
standard output script), in addition to a 2-byte witness header.

In addition to spending these outputs, a penalty transaction may optionally
spend the commitment transaction's `to_remote` output (e.g. to reduce the total
amount paid in fees). Doing so requires the inclusion of a P2WPKH witness and an
additional *txinput*, resulting in an additional 108 + 164 = 272 bytes.

In the worst case scenario, the node holds only incoming HTLCs, and the
HTLC-timeout transactions are not published, which forces the node to spend from
the commitment transaction.

With a maximum standard weight of 400000 bytes, the maximum number of HTLCs that
can be swept in a single transaction is as follows:

    max_num_htlcs = (400000 - 324 - 272 - (4 * 53) - 2) / 413 = 966

Thus, 483 bidirectional HTLCs (containing both `to_local` and
`to_remote` outputs) can be resolved in a single penalty transaction.
Note: even if the `to_remote` output is not swept, the resulting
`max_num_htlcs` is 967; which yields the same unidirectional limit of 483 HTLCs.

# General Requirements

A node:
  - upon discovering a transaction that spends a funding transaction output
  which does not fall into one of the above categories (mutual close, unilateral
  close, or revoked transaction close):
    - MUST send a warning regarding lost funds.
      - Note: the existence of such a rogue transaction implies that its private
      key has leaked and that its funds may be lost as a result.
  - MAY simply monitor the contents of the most-work chain for transactions.
    - Note: on-chain HTLCs should be sufficiently rare that speed need not be
    considered critical.
  - MAY monitor (valid) broadcast transactions (a.k.a the mempool).
    - Note: watching for mempool transactions should result in lower latency
    HTLC redemptions.

# Appendix A: Expected Weights

## Expected Weight of the `to_local` Penalty Transaction Witness

As described in [BOLT #3](03-transactions.md), the witness for this transaction
is:

    <sig> 1 { OP_IF <revocationpubkey> OP_ELSE to_self_delay OP_CSV OP_DROP <local_delayedpubkey> OP_ENDIF OP_CHECKSIG }

The *expected weight* of the `to_local` penalty transaction witness is
calculated as follows:

    to_local_script: 83 bytes
        - OP_IF: 1 byte
            - OP_DATA: 1 byte (revocationpubkey length)
            - revocationpubkey: 33 bytes
        - OP_ELSE: 1 byte
            - OP_DATA: 1 byte (delay length)
            - delay: 8 bytes
            - OP_CHECKSEQUENCEVERIFY: 1 byte
            - OP_DROP: 1 byte
            - OP_DATA: 1 byte (local_delayedpubkey length)
            - local_delayedpubkey: 33 bytes
        - OP_ENDIF: 1 byte
        - OP_CHECKSIG: 1 byte

    to_local_penalty_witness: 160 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - one_length: 1 byte
        - witness_script_length: 1 byte
        - witness_script (to_local_script)

## Expected Weight of the `offered_htlc` Penalty Transaction Witness

The *expected weight* of the `offered_htlc` penalty transaction witness is
calculated as follows (some calculations have already been made in
[BOLT #3](03-transactions.md)):

    offered_htlc_script: 133 bytes

    offered_htlc_penalty_witness: 243 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocation_key_length: 1 byte
        - revocation_key: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (offered_htlc_script)

## Expected Weight of the `accepted_htlc` Penalty Transaction Witness

The *expected weight*  of the `accepted_htlc` penalty transaction witness is
calculated as follows (some calculations have already been made in
[BOLT #3](03-transactions.md)):

    accepted_htlc_script: 139 bytes

    accepted_htlc_penalty_witness: 249 bytes
        - number_of_witness_elements: 1 byte
        - revocation_sig_length: 1 byte
        - revocation_sig: 73 bytes
        - revocationpubkey_length: 1 byte
        - revocationpubkey: 33 bytes
        - witness_script_length: 1 byte
        - witness_script (accepted_htlc_script)

# Authors

[FIXME:]

![Creative Commons License](https://i.creativecommons.org/l/by/4.0/88x31.png "License CC-BY")
<br>
This work is licensed under a [Creative Commons Attribution 4.0 International License](http://creativecommons.org/licenses/by/4.0/).
