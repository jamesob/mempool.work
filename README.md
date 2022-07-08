This document gives an overview of some of the ongoing challenges and work
in mempool design. It documents the existing design, failures, and vulnerabilities
of the mempool as well as some proposals that exist to remedy the shortcomings.

- [Assumptions](#assumptions)
- [Mempool acceptance constraints](#mempool-acceptance-constraints)
- [Established concepts](#established-concepts)
  - Important guiding concepts within mempool design
- [Failures](#failures)
- [Attacks](#attacks)
- [Proposals](#proposals)

First we describe the existing state of the mempool by way of existing
contraints, failure modes, and attacks. Then in [Proposals](#proposals) we 
discuss various measures that are yet to be implemented.


## Assumptions

This document assumes familiarity with the concepts of [ancestors, descendants,](https://github.com/bitcoin/bitcoin/blob/master/doc/policy/mempool-limits.md#definitions)
[CPFP](https://bitcoinops.org/en/topics/cpfp/), [RBF](https://bitcoinops.org/en/topics/replace-by-fee/).


## Mempool acceptance constraints

Here are some notable constraints that affect whether a transaction will be accepted to
the mempool or propagated throughout the network. 


| shortname       | description            | code           | notes                 |
| --------------- | ---------------------- | -------------- | -------------------- | 
| `ancestor-count`   | Max number of in-mempool ancestors (default: 25) | [validation.cpp:461](https://github.com/jamesob/bitcoin/blob/fcf6c8f4eb217763545ede1766831a6b93f583bd/src/validation.cpp#L461) | - |
| `ancestor-size`    | Max kilobytes of tx + all in-mempool ancestors (default: 101kb) | [validation.cpp:462](https://github.com/jamesob/bitcoin/blob/fcf6c8f4eb217763545ede1766831a6b93f583bd/src/validation.cpp#L462) | - | 
| `descendant-count` | Max number of in-mempool descendants (default: 25) | [validation.cpp:463](https://github.com/jamesob/bitcoin/blob/fcf6c8f4eb217763545ede1766831a6b93f583bd/src/validation.cpp#L463) | - |
| `descendant-size`  | Max kilobytes of tx + all in-memory descendants (default: 101kb) | [validation.cpp:464](https://github.com/jamesob/bitcoin/blob/fcf6c8f4eb217763545ede1766831a6b93f583bd/src/validation.cpp#L464) | - | 
| `min-relay-fee`     | A fee rate smaller than this is considered zero fee (for relaying, mining and transaction creation) (default: 1000 sats/kvB or 1 sat/vB) | [validation.cpp:642](https://github.com/jamesob/bitcoin/blob/fcf6c8f4eb217763545ede1766831a6b93f583bd/src/validation.cpp#L642) | - |
| `fee-filter`  | FEEFILTER messages are sent to peers to communicate the minimum fee required for admission to the mempool. It is based on `mempoolMinFee`. (See below) | [net_processing.cpp](https://github.com/jamesob/bitcoin/blob/75a227e39e37d475d6088209f24f32c070071219/src/net_processing.cpp#L4476-L4517) | - | 
| `mempool-min-fee`  | Minimum feerate for admission to the mempool (`CTxMemPool::GetMinFee`). It is updated to a track the sats/kvB feerate of transactions evicted from the mempool, and it decays exponentially. | [src/txmempool.cpp](https://github.com/jamesob/bitcoin/blob/75a227e39e37d475d6088209f24f32c070071219/src/txmempool.cpp#L1096-L1118) | - | 
| `rbf1`     | If RBFing, replaced txn must signal with nSequence | - | [BIP 125](https://github.com/bitcoin/bips/blob/master/bip-0125.mediawiki) |
| `rbf2`     | If RBFing, replacement cannot have new unconfirmed inputs | - | - |
| `rbf3`     | If RBFing, replacement must have absolute fee > replaced tx | - | - |
| `rbf4`     | If RBFing, replacement must pay for its own relay per `min-relay-fee` | - | - |
| `rbf5`     | If RBFing, transaction to be evicted + descendants cannot total more than 100| - | - |
| `cpfp-carveout`     | Allow a single-ancestor transaction through if it is no larger than 10_000 vbytes and its parent has hit its descendant limits based on another descendant | [validation.cpp](https://github.com/jamesob/bitcoin/blob/be6d4315c150646cf672778e9232f086403e95df/src/validation.cpp#L897-L911) | - |

## Established concepts

Various concepts have governed mempool design so far, and should be taken
into consideration when understanding attacks and assessing proposals.

### `TANSTAAGM`

"There ain't no such thing as a global mempool." The network's view of the
current contents of the mempool is not guaranteed to be consistent and can
remain divergent indefinitely. [See
here](https://github.com/t-bast/lightning-docs/blob/master/pinning-attacks.md#threat-model)
for more.

Various techniques can be used to intentionally "split" regions of the network
into having separate mempool content, including broadcasting conflicting
transactions simultaneously [as outlined
here](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html).

### Mempool design: `no-free-relay`

Gossiping transactions over the P2P network costs bandwidth. This bandwidth 
should not be wasted on spam.


### Mempool design: `incentive-compatibility`

The mempool should be designed in such a way that it is maximally useful for
miners when choosing transactions to include in blocks. If it is not, miners'
mempool implementations will at some point likely differ from non-mining nodes
(to be revenue maximizing) and bitcoind users will not have an accurate
impression of miner mempool contents, impairing their ability to estimate
market feerates and get transactions confirmed.

Note that in the limit, miner mempool design will likely diverge from non-mining
nodes because of the computationally complexity of optimally knapsack-solving
for the most profitable block (NP-hard). This divergence likely won't happen
until miner revenue from fees surpasses revenue from the block subsidy.


## Failures

"Failures" are considered undesirable behaviors that result from shortcomings
in the current design of the mempool; they are distinguished from attacks in
that they need not be consciously exploited by an attacker to result in the
undesirable behavior.


### Failure: `presigned-feerate-too-low`

In certain applications, a key which later becomes inaccessible may be used to
presign transactions that will at some point be broadcast, essentially turning
those presigned transactions into "bearer assets." This is the case with [many
vault designs](https://github.com/jamesob/simple-ctv-vault#fee-management).  A
similar case exists with counterparties who presign a certain transaction, but
can't be relied upon to resign for fee renegotiation.

In such cases, the successful broadcast of the pre-signed transaction is
critical to the contracting application. However, the pre-negotiated feerate
associated with the transaction may fall beneath the `mempoolMinFee` at time of
broadcast if the fee market has moved significantly since the txn was prepared.

The txn may not be able to propagate on the basis of a too-low feerate.
Normally, `rbf` would be used to dynamically adjust the feerate, but because
the keys used to sign are no longer available, RBF is not an option.

Currently, these cases rely on CPFP (child pays for parent) to adjust the feerate. 
But because the original transaction's feerate may be too low to even be admitted to the
mempool, there is no parent for the child to descend off of.

For this reason, `pacakgeRelay` is a necessary change to avoid failure in such
a situation.


## Attacks

Attacks are behaviors in the current mempool design that could be used by
malicious actors to exploit applications built on top of Bitcoin.

### Attack: `miner-harvesting`

- [Mentioned in @naumenkogs ML post](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-February/002569.html)
- [Mentioned in @ariard ML post](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2021-November/019615.html)

A subset of the miners could drop on the floor transaction inclusions to provoke
artificial congestion during period P. This congestion would be overreacted by the
fee-bumping algorithms of the Bitcoin applications, especially with time-sensitive
requirements in a way that the loss of fees during period P compensates for the gain
during period P+1. It is left as subject of research if future second-layers fee-bumping
algorithms could be vulnerable to those concerns.

### Attack: `mempool-partitioning`

Opt-in RBF enables to segregate network mempools in different arbitrary subsets.
From those subsets, a well-provisioned in UTXO value attacker could broadcast
branch of high-fees child transactions in some target subsets. The targeted
nodes would observe different mempool feerates from the non-partitioned ones. Other
policy discrepancies could be leveraged to partition mempools.

It is left as subject of research how it could be a building block in more
sophisticated attack.

### Attack: `miner-mapping`

- [Mentioned in @ariard ML post](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html)

Regions of the network that feed into major miners' mempools could be identified
by the following technique: `n` conflicting variations of a given transaction
could be created and then broadcast to different peers. Each time one of the variants
is mined, that identifies a peer which affects the mempool composition of a
miner. 

In this way, P2P network topology can be discovered and leveraged during a
mempool-based attack.

### Attack: `fee-pinning`


Fee pinning is an attack in which a malicious actor is able to "pin" a
transaction to the bottom of the mempool (in terms of mining attractiveness)
with the intent of preventing or delaying the confirmation of that transaction.

This has particular relevance in the case of time-sensitive second layer
contracting protocols like Lightning channels, where the timely confirmation
of, say, a justice transaction is critical to enforcing the assumed behavior of
the contract.

Pinning prevents the dynamic adjustment of fees via `rbf` or `cpfp`.

#### Examples

##### `pin-by-anyonecanpay`

- [Described in @glozow's RBF Improvements post](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff?permalink_comment_id=4058140#sighash_anyonecanpay-pinning)

`SIGHASH_ANYONECANPAY`: the `SIGHASH_ANYONECANPAY` flag allows any user to
reuse signatures unlocking inputs while adding additional inputs to a new
transaction (which reuses the original inputs). 

This can be exploited by a malicious user Mal if Mal takes her counterparty's
transaction which has not fully propagated through the network yet and appends
on a large number of input transactions which make its feerate mediocre. Mal
then would create a "split brain" mempool that has nodes feeding miner block
assemblies contain this low feerate version of the transaction, pinning it,
even in the presence of the "honest" higher feerate transaction elsewhere on
the network.

##### `pin-by-descendants`

- Fixed for two-party contracts by `cpfp-carveout`
- [Described in CPFP carveout post by BlueMatt](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2018-November/016518.html)

Because there are various limits on how many descendants a transaction can
have, as well as the total size of those descendants, participants in a
two-party contract can potentially pin the feerate of a transaction by hanging,
e.g., 25 children at a low feerate off of an output they control. Their
counterparty will not be able to bump fees using CPFP - though this has now
been fixed by [the CPFP carveout
change](https://bitcoinops.org/en/topics/cpfp-carve-out/).

Note that the CPFP-carveout only resolves this pinning vector for two-party
contracts. More outputs than two could reopen the ability to pin, since the
carveout only allows for one extra child feebump.


##### Lightning attacks
     
- Various pinning attacks are described by @ariard [in this
post](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html).



### Attack: `feerate-mempool-siphon`

- Hypothetical (contingent on removal of RBF rule 3)
- [Described here](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff?permalink_comment_id=4081417#gistcomment-4081417)

If RBF rule #3 were removed, an attacker could broadcast, say, 300MB of large
transactions which have a higher feerate than anything in the mempool.
Subsequently, all transactions currently in the mempool would be evicted in
nodes operating under default mempool settings. The attacker could then replace
these needlessly large transactions which much smaller transactions that pay a
higher feerate (but lower absolute fees) via RBF. 

So, if [rule 3 were removed](#removing-rbf-rule-3), an attacker would be able
to exhaust the mempools of the network for a known (and probably relatively
low) cost.


## Proposals

### Package relay

- [Mailing-list post (2022 May)](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-May/020493.html)

Currently, the P2P network relays individual transactions, creating a situation
in which transactions which do not surpass the `mempool-min-fee` cannot be broadcast, 
even if bolstered by CPFP.

Package relay would allow connected DAGs of dependent transactions to be
relayed together.  This would ensure that transactions which require CPFP to
bump their effective feerate are able to enter mempools to begin with, else
they suffer the `presigned-feerate-too-low` failure.

There are not any conceptual objections to this proposal, and indeed it is widely
recognized that package relay is needed. Remaining considerations are P2P protocol
considerations like

- how to structure the protocol to avoid DoS vectors (e.g. if a node misreports
  the existence of transactions or aggregate feerate),
- whether to commit to the current tip hash within PKG messages,
- whether packages should necessarily be topologically sorted.

Related pull requests:
- [Pending] [BIP125-based Package RBF](https://github.com/bitcoin/bitcoin/pull/25038)
- [Merged] [-regtest-only RPC for testing package acceptance](https://github.com/bitcoin/bitcoin/pull/24836)
- [Merged] [CPFP fee bumping within packages](https://github.com/bitcoin/bitcoin/pull/24152)


### BlueMatt/sdaftuar voluntary child-size limit

- [Mentioned here](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff?permalink_comment_id=4058140#gistcomment-4058140)
- [Implementation here](https://github.com/glozow/bitcoin/commit/b22afa034135f41e33b075e4698d35ec33fe2585)

> An alternative idea, from talking to @TheBlueMatt today: we could set a bit
> in a transaction that means "do not accept more than X vbytes worth of
> descendants into the mempool that depend on this one, because this may be
> RBF'ed in the future".
> 
> Values of X could be something like twice the original transaction's vsize
> (maybe with a floor -- 1000 vbytes? 5000? ), or some other smallish value.
> 
> The point of this would be to bound the RBF cost to the original transactor,
> as the issue with RBF pinning is that a package's size can be 1000 times
> bigger than a typical transaction, so if we can bound that to 2x the size of
> the original transaction (or something similarly small), the RBF cost when
> you're paying for descendants seems much more reasonable.
> 
> This approach would not require making any feerate predictions (which is
> nice), but also may suffer from the problem of wallets not knowing when to
> use it... But if this at least makes it easier for some use cases, maybe it's
> still forward progress?
>
> *sdaftuar*

This proposal would potentially fix some pinning vectors associated with
2-party contracting applications, but would not address fee-bumping needs for
applications where RBF use is not in scope, e.g. vaults.


#### Alternative: transaction size limit enforced in script

- [Original ML post by Greg Sanders](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-May/020458.html)
- [OpTech summary of proposal](https://bitcoinops.org/en/newsletters/2022/05/18/#using-transaction-introspection-to-prevent-rbf-pinning)

@instagibbs posted an alternative to the idea above, which is similar but instead
involves adding an opcode to push the size of the current transaction to the stack,
allowing script writers to limit the size of the transaction.


### Removing RBF rule 3

Naively, rule #3 of RBF (higher absolute fee required for a replacment) seems
undesirable. If a surrogate replacement transaction can pay a higher feerate
but lower absolute fee, it indicates that a *smaller* equivalent transaction is
suitable to the transactor(s), consuming less chainstate, which is a common
good for the network. Though note that under certain unusual circumstances,
this may not be `incentive-compatible` for miners - namely if the surrogate
transactions which fill in the extra block space yielded by the replacement
transaction do not pay as much as the original absolute fee.

In any case, it is tempting to remove rule #3, since in the average case
replacement of a higher feerate transaction benefits everyone. However, the
relaxation of this rule would allow the siphoning attack mentioned above.

  - Add delay propogation to avoid bandwidth DoS: https://lists.linuxfoundation.org/pipermail/lightning-dev/2018-June/001316.html

### Transaction sponsors

- [ML post by Jeremy Rubin](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2020-September/018168.html)
- [Implementation](https://github.com/bitcoin/bitcoin/compare/master...JeremyRubin:subsidy-tx)

Transaction sponsors, in concept, allow any user to increase the feerate of any
transaction by marking a *sponsor transaction* as only being consensus valid if
mined alongside a certain transaction (i.e. the sponsored). The sponsor txn
marks itself as a sponsor by setting its last output, probably dust, as having
the scriptPubKey `[sponsored-txid] OP_VER`.

Objections to the original proposal include:

- introducing a new kind of dependency between transactions,
- unclear how to avoid pinning of sponsors with suboptimal feerates,
  - (a sponsor replacement policy would need to be devised to avoid this)
- requires a consensus change (soft fork).


### [conceptual] Unauthenticated feerate-bumps always possible

- [ML post by @jamesob](https://lists.linuxfoundation.org/pipermail/bitcoin-dev/2022-February/019879.html)

Ideally, any user should be able to additively bump the feerate of any
transaction, regardless of whether or not the transaction anticipated feebumps
structurally, e.g. being marked RBFable. 

It doesn't make sense to require authentication for feebumps, as the owner of
the inputs in the txn-to-be-bumped has already authenticated their spend via
signature. However, the ideal of monotonic, unauthenticated feerate bumps is
hampered by the real-world issues of
- avoiding additional bandwidth DoS vectors associated with doing miniscule
  feerate bumps, causing re-relay and needless eviction of txns, and
- avoiding the introduction of pinning vectors.

One possible high-level sketch for achieving this is a combination of
- SIGHASH_GROUP
  - with the modification that GROUP allows a given input to commit optionally
    to the last output (see "sponsors" below)
- transaction sponsors
  - with the modification that each sponsor input entry must be SIGHASH_GROUP,
    committing to at least the last output (i.e. the sponsor vector). This allows
    input/output groups within the sponsor to be arbitrarily conjoined in order
    to create a highest-feerate replacement transaction.
  - allow sponsor replacement on the basis of at least an x% feerate bump. `x`
    might be required to increase to limit relay.

This would ensure that sponsors are not pinnable, and relay is limited.

A similar proposal could be done without sponsors, simply by having
transactions make use of SIGHASH_GROUP, but that would require structural
anticipation on the part of transactions.

## Links
- [Bitcoin Core docs on mempool policy](https://github.com/bitcoin/bitcoin/tree/master/doc/policy)
- [@t-bast's write-up on pinning attacks](https://github.com/t-bast/lightning-docs/blob/master/pinning-attacks.md)
- [@ariard's Pinning: The Good, The Bad, The Ugly](https://lists.linuxfoundation.org/pipermail/lightning-dev/2020-June/002758.html)
- [@glozow's proposed removal of RBF rule 2](https://github.com/bitcoin/bitcoin/pull/23121)
- [@glozow's proposed RBF improvements](https://gist.github.com/glozow/25d9662c52453bd08b4b4b1d3783b9ff)
- [@glozow's pinning zoo](https://github.com/glozow/bitcoin-notes/blob/master/pinning.md)
- [Revault write-up on why they have abandoned dynamic fee bumping](https://github.com/revault/practical-revault/pull/119)
