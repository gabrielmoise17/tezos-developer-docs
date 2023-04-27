---
title: Token transfers and balance updates
---

This page describes the reporting of tokens transfers in block metadata,
as a sequence of balance updates. They serve among others to the Octez
client to derive and print the [Balance updates]{.title-ref} sections of
the receipts reported when operations are included in blocks.

# Overview

Minting, transferring or burning tokens is handled by the
`Token <tezos-protocol-016-PtMumbai/Tezos_raw_protocol_016_PtMumbai/Token/index.html>`{.interpreted-text
role="package-api"} module. The module provides functions
(`Token.transfer` and `Token.transfer_n`) to transfer tokens from one,
respectively from more accounts, to another account. Balance updates
found in block metadata are generated by these functions as a trace of
the token movements having taken place.

Balance updates indicate that either: tokens have been minted and
deposited into an account, transferred from or to another account, or
taken from an account and burned. In the Json format, a balance update
consists of three parts:

> -   an account identification indicated by a combination of fields
>     such as: `kind`, `category`, `contract`, \...
> -   the amount transferred (in mutez) indicated by the field `change`.
>     A positive amount indicates that the account has been credited,
>     and a negative amount indicates that the account has been debited.
> -   the cause of the update given by the field `origin` which may have
>     the following values:
>     -   `"block"` means that the balance update originates from the
>         application of a block
>     -   `"migration"` means that the balance update originates from
>         migration
>     -   `"subsidy"` means that the balance update originates from
>         subsidies for liquidity baking

A transfer of tokens is represented by a continuous and ordered sequence
of (balance) updates. That sequence starts with a series of debits, and
ends with a credit matching those debits. In block metadata, the field
`"balance updates"` contains one or more transfers, represented by their
sequences of updates, concatenated in a flat list. Consider for example
the following list:

    [ {"kind": "...", ..., "change": "-100", "origin": "block"},
      {"kind": "...", ..., "change": "100", "origin": "block"},
      {"kind": "...", ..., "change": "-125", "origin": "block"},
      {"kind": "...", ..., "change": "-75", "origin": "block"},
      {"kind": "...", ..., "change": "200", "origin": "block"} ]

This list reports that two transfers have occurred: a transfer of `100`
mutez from one account to another, and a transfer of `200` mutez from
two accounts to a third. There is one exception to this though:
migration balance updates are a bit compressed to save space, so the
list of balance updates in the corresponding metadata does not
necessarily allow to correlate debits to a credit by analyzing the
sequence. However, for those balance updates, the `delegate` and
`contract` fields can be used to establish correlations between debits
and credits.

# Source, container, and sink accounts

There are three kinds of accounts: source accounts, container accounts,
and sink accounts. Tokens can be transferred from source or container
accounts, to container or sink accounts. All balance updates contain the
field `kind` which allows to determine the kind of account it refers to.
The possible values of this field are described in the following
sections (they are not simply `"source"`, `"container"`, and `"sink"`).
Depending on the kind of account, more fields such as the `category`
field may be used to identify accounts more specifically.

## Source accounts

Source accounts are debited whenever new tokens are minted. A balance
update refers to a source account if and only if the field `kind` has
the value `"minted"`. The value of the additional field `category`
designates one of the following fictitious accounts, each containing a
virtually unlimited number of tokens:

-   `"nonce revelation rewards"` is the source of tokens minted to
    reward delegates for revealing their nonces
-   `"double signing evidence rewards"` is the source of tokens minted
    to reward delegates for injecting a double signing evidence
-   `"endorsing rewards"` is the source of tokens minted to reward
    delegates for endorsing blocks
-   `"baking rewards"` is the source of tokens minted to reward
    delegates for creating blocks
-   `"baking bonuses"` is the source of tokens minted to reward
    delegates for validating blocks and including extra endorsements
-   `"subsidy"` is the source of tokens minted to subsidize the
    liquidity baking CPMM contract
-   `"invoice"` is the source of tokens minted to compensate some users
    who have contributed to the betterment of the chain
-   `"commitment"` is the source of tokens minted to match commitments
    made by some users to supply funds for the chain
-   `"Tx_rollup_rejection_rewards"` is the source of tokens minted to
    reward an account for injecting a transaction rollup rejection
    operation
-   `"Sc_rollup_refutation_rewards"` is the source of tokens minted to
    reward an account for winning a smart-contract rollup refutation
    game
-   `"bootstrap"` is analogous to `"commitment"` but is for internal use
    or testing. It will not be used during normal operation on mainnet,
    but may be used on test networks or in sandboxed mode
-   `"minted"` is only for internal use and may be used to mint tokens
    for testing. It will not be used during normal operation on mainnet,
    but may appear on test networks or in sandboxed mode.

## Container accounts

Container accounts are regular (user and smart contract) accounts, or
convenience accounts that hold tokens temporarily (e.g. when parts of a
delegate\'s funds are frozen). The field `kind` allows to identify the
type of container account, it can have one of the following values:

-   `"contract"` represents implicit or originated accounts, and comes
    with the additional field (also called) `contract` whose value is
    the public key hash of the implicit or originated account.
-   `"freezer"` represents frozen accounts, and comes with the
    additional field `category` that can have one of the following
    values:
    -   `"legacy_deposits"`, `"legacy_fees"`, or `"legacy_rewards"`
        represent the accounts of frozen deposits, frozen fees or frozen
        rewards up to protocol HANGZHOU. Accounts in this category are
        further identified by the following additional fields:
        -   the field `delegate` contains the public key hash of the
            delegate who owns the frozen funds
        -   the field `cycle` contains the cycle at which the funds have
            been deposited or granted.
    -   `"deposits"` represents the accounts of frozen deposits in
        subsequent protocols (replacing the legacy container account
        `"legacy_deposits"` above). Accounts in this category are
        further identified by the additional field `delegate` whose
        value is the public key hash of the delegate who owns the frozen
        funds.
    -   `"bonds"` represents the accounts of frozen bonds. Bonds are
        like deposits. However, they can be associated to implicit or
        originated accounts, unlike deposits that only apply to implicit
        accounts that are also delegates. Accounts in this category are
        further identified by the following additional fields:
        -   the field `contract` contains the public key hash of the
            implicit account, or the contract hash of the originated
            account
        -   the field `bond_id` contains the identifier of the bond
            (e.g. a rollup hash if the bond is associated to a
            transaction or a smart contract rollup).
-   `"accumulator"` represents accounts used to store tokens for some
    short period of time. This type of account is further identified by
    the additional field `category` whose (only possible) value
    `"block fees"` designates the container account used to collect
    manager operation fees while block\'s operations are being applied.
    Other categories may be added in the future.
-   `"commitment"` represents the accounts of commitments awaiting
    activation. This type of account is further identified by the
    additional field `committer` whose value is the encrypted public key
    hash of the user who has committed to provide funds.

## Sink accounts

Sink accounts are credited whenever tokens are burned. A balance update
refers to a sink account if and only if the field `kind` has the value
`"burned"`. The value of the additional field `category` allows to
identify more specifically a fictitious account able to receive a
virtually unlimited number of tokens. The field `category` of a sink
account may have one of the following values:

-   `"storage fees"` is the destination of storage fees burned for
    consuming storage space on the chain
-   `"punishments"` is the destination of tokens burned as punishment
    for a delegate that has double baked or double endorsed
-   `"lost endorsing rewards"` is the destination of rewards that were
    not distributed to a delegate. This category comes with the
    following additional fields:
    -   the field `delegate` contains the public key hash of the
        delegate
    -   the field `participation` has the value `"true"` if
        participation was not sufficient and has the value `"false"`
        otherwise
    -   the field `revelation` has the value `"true"` if the delegate
        has not revealed his nonce and has the value `"false"`
        otherwise.
-   `"Tx_rollup_rejection_punishments"` is the destination of tokens
    burned as punishment for submitting erroneous commitments
-   `"Sc_rollup_refutation_punishments"` is the destination of tokens
    burned as punishment for submitting bad commitments that have been
    refuted
-   `"burned"` is only for internal use and testing. It will not appear
    on mainnet, but may appear on test networks or in sandboxed mode.

# Token transfers and metadata

Balance updates in block metadata give a complete account of all token
transfers that have occurred when a block is applied. A few cases of
token transfers and the associated metadata are illustrated below. All
other cases of token transfers in the protocol follow the same pattern.
The only differences are the accounts involved.

## Origination and transaction

When an origination or transaction operation is applied, tokens are
transferred from one contract to another. Depending on whether or not
storage space has been allocated on the chain by the application of the
operation, storage fees may also be burned. For example, a transaction
of `100` mutez from address `tz1a...` to address `KT1b...` that
allocates storage space for a cost of `10` mutez produces the following
list of balance updates:

    [ {"kind": "contract", "contract": "tz1a...", "change": "-100", "origin": "block"},
      {"kind": "contract", "contract": "KT1b...", "change": "100", "origin": "block"}
      {"kind": "contract", "contract": "tz1a...", "change": "-10", "origin": "block"}
      {"kind": "burned", "category": "storage fees", "change": "10", "origin": "block"} ]

## Baking fees, rewards and bonuses

When a contract pays the baking fees associated to an operation it has
emitted, those fees are temporarily collected (during the processing of
the block) into the container account `"block fees"`. For example, when
a manager operation is applied, the account of the payer contract is
debited with the amount of fees and the `"block fees"` account is
credited with the same amount. Hence, for `100` mutez in fees, the
following balance updates are generated :

    [ {"kind": "contract", "contract": "tz1x...", "change": "-100", ...},
      {"kind": "accumulator", "category": "block fees", "change": "100", ...} ]

When all operations of a block have been applied baking fees rewards and
bonuses are distributed. The total amount of fees collected and the
baking rewards are transferred from the container account `"block fees"`
and the source account `"baking rewards"`, respectively, to the contract
of the payload producer that selected the transactions to be included in
the block. So, for a total amount of `1000` mutez in fees collected and
an amount of `500` mutez in baking rewards, the following balance
updates are generated:

    [ {"kind": "accumulator", "category": "block fees", "change": "-1000", ...},
      {"kind": "minted", "category": "baking rewards", "change": "-500", ...},
      {"kind": "contract", "contract": "tz1a...", "change": "1500", ...} ]

The baking bonus go to the block proposer that signed and injected the
block. Hence the amount of the bonus is transferred from the source
account `"baking bonuses"` to the contract of the block producer. For
example, the balance updates generated for an amount of `100` mutez in
baking bonus are:

    [ {"kind": "minted", "category": "baking bonus", "change": "-100", ...},
      {"kind": "contract", "contract": "tz1b...", "change": "100", ...} ]

## Endorsing, double signing evidence, and nonce revelation rewards

Endorsing rewards are reflected in balance updates as a transfer of
tokens from the `"endorsing rewards"` source account to the account of
the delegate that receives the reward. Hence, for a reward of `100`
mutez, the following two balance updates are generated:

    [ {"kind": "minted", "category": "endorsing rewards", "change": "-100", ...},
      {"kind": "contract", "contract": "tz1...", "change": "100", ...} ]

When endorsing rewards are not distributed to the delegate due to
insufficient participation or for not revealing nonces, they are
transferred instead to the sink account identified by the quadruple
`("lost endorsing rewards", delegate, participation, revelation)`. For
example, for an amount of `100` mutez in rewards not distributed due to
insufficient participation, the following balance updates are generated:

    [ {"kind": "minted", "category": "endorsing rewards", "change": "-100", ...},
      {"kind": "burned",
       "category": "lost endorsing rewards",
       "delegate": "tz1...",
       "participation": "true",
       "revelation": "false",
       "change": "100", ...} ]

Double signing evidence rewards and nonce revelation rewards are
analogous to endorsing rewards, except that the source accounts used are
`"double signing evidence rewards"` and `"nonce revelation rewards"`.