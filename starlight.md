# Starlight Payment Channels Protocol

- [Introduction](#introduction)
- [Workflows](#workflows)
  - [Create a channel](#create-a-channel)
  - [Payments](#payments)
  - [Conflict Resolution](#conflict-resolution)
  - [Host top-up](#host-top-up)
  - [Cooperative closing](#cooperative-closing)
  - [Force closing](#force-closing)
  - [Monitoring](#monitoring)
- [Reference](#reference)
  - [State](#state)
  - [Accounts](#accounts)
  - [Computed Values](#computed-values)
  - [Messages](#messages)
  - [Transactions](#transactions)
  - [User commands](#user-commands)
  - [Timers](#timers)
  - [Fees](#fees)
  - [Timing parameters](#timing-parameters)
  - [Stellar protocol background](#stellar-protocol-background)

## Introduction

- two parties: <span style="color:blue">_host_</span> (opens channel) and <span style="color:blue">_guest_</span>
- both run an <span style="color:blue">_agent_</span>, a server which must be highly available
- the guest agent must have a public IP or URL, but not host
- host sends POST and long-polling GET request to guest
- host setups accounts and pays
	- their minimum account balance (fully recovered)
	- on chain transaction fees (partially recoceved)
- `HostAccount` the single account for the host (not multisig)
- `GuestAcount` the single account for the guest (not multisig)
- new accounts (created by host and merged back into HostAccount when channel closes)
	- `EscrowAccount` holds the funds of the channel
	- `HostRatchetAccount`; only holds minimum balance; used as source account for channel transactions
	- `GuestRatchetAccount`; only holds minimum balance; used as source account for channel transactions


## Workflows
### Create a channel
- channel is started in the `Start` state
- host user sends `CreateChannelCmd` to host's agent
- host
	- generates keypairs `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount`
	- choose parameter `Feerate`
	- creates, signs and publishes `SetupAccountTx(HostRatchetAccount)`, `SetupAccountTx(GuestRatchetAccount)` and `SetupAccountTx(EscrowAccount)`
	- goes into `SettingUp` state and waits for the three accounts to be created
	- determine `EscrowAccountSequenceNumber`, `HostRatchetAccountSequenceNumber` and `GuestRatchetAccountSequenceNumber`
	- puts `RoundNumber` = `1`
	- puts `HostBalance = HostAmount`, `GuestBalance = 0`
	- puts `PaymentTime = FundingTime` = most recent ledger timestamp
	- sends a `ChannelProposeMsg` to the guest's agent URL
	- goes into `ChannelProposed` state
- guest
	- receives `ChannelProposedMsg`
	- put `PaymentTime = FundingTime = ChannelProposedMsg.FundingTime`
	- determine `EscrowAccountSequenceNumber`
	- puts `RoundNumber = 1`
	- create a `SettleOnlyWithHostTx(FundingTime)` and a `RatchetTx(HostRatchetAccount, FundingTime)` and sign both
	- sends a `ChannelAcceptMsg` to the host
	- goes into `AwaitingFunding` state
- host
	- receives `ChannelAcceptMsg` and validates it
	- set `CurrentSettlementTxes = { ChannelAcceptMsg.SettleOnlyWithHostTx }` and `CurrentRatchetTx = ChannelAcceptMsg.RatchetTx`
	- creates `FundingTx`, signs with `HostAccount`, `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` and submits
	- goes into `AwaitingFunding` state
- host and guest go into `Open` state once they observe that `FundingTx` is on chain
- guest
	- checks that
		- `EscrowAccountSequenceNumber` and `GuestRatchetAccountSequenceNumber` are still correct
		- balance of `EscrowAccont` is at least `HostAmount + 1.5 + 8 * Feerate`
		- signers of `EscrowAccount` and the two ratchet accounts are as expected
	- if any check fails it transitions to `Closed` state

### Payments
- from <span style="color:blue">_sender_</span> to <span style="color:blue">_recipient_</span>
- sender user send `ChannelPayCmd` to sender's agent
	- agent will reject if not in `Open` state or if `ChannelPayCmd.PaymentAmount` is higher than the sender's balance in channel
- sender
	- put `PendingPaymentTime = PaymentTime` = max(most recent ledger timestamp, `PaymentTime`)
	- `RoundNumber += 1`
	- put `PendingPaymentAmount = ChannelPayCmd.PaymentAmount`
	- create `PaymentSettleWithGuestTx` and `PaymentSettleWithHostTx`
	- send a `PaymentProposeMsg ` to the recipient
	- goes into `PaymentProposed` state
- recipient
	- receives `PaymentProposeMsg` and validates it
	- if in `PaymentProposed` state, then see below (Conflict Resolution)
	- put `RoundNumber = PaymentProposeMsg.RoundNumber` (= `RoundNumber + 1`)
	- put `PendingPaymentAmount = PaymentProposeMsg.PaymentAmount` and `PendingPaymentTime = PaymentProposeMsg.PaymentTime`
	- create `SenderRatchetTx`, `CompletePaymentSettleWithGuestTx`, `CompletePaymentSettleWithHostTx`
	- set `CounterpartyLatestSettlementTxes` to `PaymentSettleWithGuestTx` and `PaymentSettleWithHostTx`
	- send a `PaymentAcceptMsg` to the sender
	- goes into `PaymentAccepted` state
- sender
	- receives `PaymentAcceptMsg` and validate it
	- create `RecipientRatchetTx`
	- send a `PaymentCompleteMsg` to the recipient
	- put `CurrentRatchetTx = SenderRatchetTx`, `GuestBalance = NewGuestBalance`, `HostBalance = NewHostBalance`, `PaymentTime = PendingPaymentTime`, `CurrentSettlementTxes = CounterpartyLatestSettlementTxes = {CompletePaymentSettleWithGuestTx, CompletePaymentSettleWithHostTx}`
	- goes into `Open` state
- recipient
	- receives `PaymentCompleteMsg` and validates it
	- put `CurrentRatchetTx = RecipientRatchetTx`, `GuestBalance = NewGuestBalance`, `HostBalance = NewHostBalance`, `PaymentTime = PendingPaymentTime`, `CurrentSettlementTxes = CounterpartyLatestSettlementTxes = {CompletePaymentSettleWithGuestTx, CompletePaymentSettleWithHostTx}` (???)
	- goes into `Open` state

### Conflict Resolution
- recipient receives `PaymentProposeMsg` in `PaymentProposed` state (so it is also a sender)
	- if `PendingPaymentAmount > PaymentProposeMsg.PaymentAmount` or (`PendingPaymentAmount = PaymentProposeMsg.PaymentAmount` and recipient is host)
		- put `RoundNumber += 1`
		- put `PendingPaymentAmount -= PaymentProposeMsg.PaymentAmount`
		- create `PaymentSettleWithGuestTx` and `PaymentSettleWithHostTx`
		- send a `PaymentProposeMsg ` to the recipient
	- else
		- put `RoundNumber += 1` (NO !!!)
		- put `TheirPaymentAmount = PaymentProposeMsg.PaymentAmount` and `MyPaymentAmount = PendingPaymentAmount`
		- goes into `AwaitingPaymentMerge` state

### Host top-up
- host user sends `TopUpCmd` to host's agent
	- agent rejects if channel with `EscrowAccount` does not exist
	- create and submit `TopUpTx(Amount)`

### Cooperative Closing
- user sends a `CloseChannelCmd` to its agent
	- this agent is the <span style="color:blue">_closer_</span>; the other the <span style="color:blue">_cooperator_</span>
- closer
	- constructs a `CooperativeCloseTx` and signs with `EscrowAccount` (if closer is host) or `GuestAccount` (if closer is guest)
	- sends `CloseMsg` to cooperator
	- goes into `AwaitingClose`
- cooperator
	- receives `CloseMsg` and validates it
	- signs `CloseMsg.CooperativeCloseTx` with `GuestAccount` (if closer is host) or `EscrowAccount` (if closer is guest) and submits to stellar
	- if submission successful
		- goes into `Closed` state
	- otherwise
		- submit `CurrentRatchetTx`
		- goes into `AwaitingRatchet` state

### Force closing
- user sends a `ForceCloseCmd` to its agent
	- submit `CurrentRatchetTx`
	- goes into `AwaitingRatchet` state
- agent is in state `AwaitingRatchet` and `CurrentRatchetTx` gets on chain
	- goes to `AwaitingSettlementMintime` state
- agent is in state `AwaitingSettlement` and `CurrentSettlementTxes` get on chain
	- goes to `Closed` state

### Monitoring
The following conditions need to be constantly monitored by both agents and reacted accordingly
- host and guest monitor `EscrowAccount` for outside payments to the account
	- if there is such a payment, they increase `HostBalance` by that value
- host and guest monitor a valid `CooperativeCloseTx` to get on chain
	- if this happens, then go to `Closed` state
- counterparty publishes a ratchet transaction
	- if that bumps the sequence number of `EscrowAccount` to a lower number than `CurrentRatchetTx` does
		- submit `CurrentRatchetTx`
		- goes into `AwaitingRatchet` state
	- if that bump the sequence number of `EscrowAccount` to the same number as `CurrentRatchetTx` does
		- goes to `AwaitingSettlementMintime` state
	- if that bumps the sequence number of `EscrowAccount` to a higher number than `CurrentRatchetTx` does
		- put `CurrentSettlementTxes` to `CounterpartyLatestSettlementTxes`
		- if in state `AwaitingSettlement` go to state `AwaitingSettlementMintime`

## Reference

### State
#### Immutable Channel Information
Is stored as long as channel is not in `Start` or `Closed` state
- `EscrowAccount`
- `EscrowAccountSequenceNumber`: initial sequence number of `EscrowAccount`
- `MaxRoundDuration`
- `FinalityDelay`
- `CounterpartyURL` (starlight URL of counterparty)
- `HostRatchetAccount` and `HostRatchetAccountSequenceNumber` (if account already created)
- `GuestRatchetAccount` and `GuestRatchetAccountSequenceNumber` (if account already created)
- `GuestAccount`
- `Role` (either "host" or "guest")

#### Shared State
Only exists if channel is in an `Open state`, `Payment state`, `Cooperative closing state` or `Force closing state`
- `RoundNumber`: number of current round
- `HostBalance`: current channel balance of host
- `GuestBalance`: current channel balance of guest
- `CurrentRatchetTx`: latest `RatchetTx` including counterparties signature
- `CurrentSettlementTxes`: latest settlement transaction including counterparties signature; contains only a `SettleOnlyWithHostTx` if `GuestBalance` is 0; otherwise contains a `SettleWithGuestTx` and a `SettleWithHostTx`
- `CounterpartyLatestSettlementTxes`: latest settlement transactions for which counterparty has a valid `RatchetTx`, inluding counterparties signature
- `PaymentTime`: time of latest completed payment (or `FundingTime` if no payment yet)

#### Channel States
##### `Setup State`
- `Start`: channel neither created nor proposed
- `SettingUp` (host only): wait for three `SetupAccountTx` to get on chain
- `ChannelProposed` (host only): after sending `ChannelProposeMsg`; wait for guest's `ChannelAcceptMsg`
- `AwaitingFunding`: wait for `FundingTx` to get on chain
- `AwaitingCleanup` (host only): wait for `CleanupTx` to get on chain
##### `Open state`
- `Open`: standard open state; channel is open, round not timed out yet, no pending payments or closed attempts
##### `Payment states`
- `PaymentProposed` (sender only): after sending `PaymentProposeMsg`; wait for `PaymentAcceptMsg`
- `PaymentAccepted` (recipient only): after sending `PaymentAcceptMsg`; wait for `PaymentCompleteMsg`
##### `Conflict resolution states`
- `AwaitingPaymentMerge`: a sender received a `PaymentProposeMsg` with a higher `PaymentAmount` while in state `PaymentProposed`
	- wait for another `PaymentProposeMsg`
	- additional state: `MyPaymentAmount` and `TheirPaymentAmount`
##### `Force closing states`
- `AwaitingRatchet`: wait for `CurrentRatchetTx` to get on chain
- `AwaitingSettlementMintime`: wait for minimum time of `CurrentSettlementTxes`
- `AwaitingSettlement`: wait for `CurrentSettlementTxes` to go on chain
- `Closed`: channel has been closed
##### `Cooperative closing states`
- `AwaitingClose`: wait for `CooperativeCloseTx` to get on chain

### Accounts
- id `EscrowAccount`
	- holds the payment channel funds
	- created by `SetupAccountTx`
	- two signers: `EscrowAccount` and `GuestAccount`
	- minimum balance: 1.5 XLM
- id `HostRatchetAccount`
	- created by `SetupAccountTx`
	- sole signer: `EscrowAccount`
	- minimum balance: 1.5 XLM
- id `GuestRatchetAccount`
	- created by `SetupAccountTx`
	- two signers: `EscrowAccount` and `GuestAccount`
	- minimum balance: 2 XLM
- id `HostAccount`
	- the account of the host; can be used for all channels of the host (also as guest for another channel)
- id `GuestAccount`
	- the account of the guest; can be used for all channels of the guest (also as host for another channel)

### Computed Values
are values defined per payment round
- `RoundSequenceNumber` = `EscrowAccountSequenceNumber + RoundNumber * 4`
- `PendingPaymentTime`: the `PaymentTime` from the `PaymentProposeMsg`
- `PendingPaymentAmount`: the `PaymentAmount` from the `PaymentProposeMsg`
- `SenderRatchetAccount`: the ratchet account of the sender
- `RecipientRatchetAccount`: the ratchet account of the recipient
- `SenderEscrowPubKey`: is `EscrowAccount` if sender is host, otherwise `GuestAccount`
- `RecipientEscrowPubKey`: is `EscrowAccount` if recipient is host, otherwise `GuestAccount`
- `NewGuestBalance`: `GuestBalance + PendingPaymentAmount` if sender is host and `GuestBalance - PendingPaymentAmount` otherwise
- `NewHostBalance`: `HostBalance - PendingPaymentAmount` if sender is host and `HostBalance + PendingPaymentAmount` otherwise
- `PaymentSettleWithGuestTx`: if `NewGuestBalance === 0`, then empty; otherwise it is `SettleWithGuestTx(PendingPaymentTime, NewGuestBalance)` signed with `SenderEscrowPubKey`
	- `CompletePaymentSettleWithGuestTx`: `PaymentSettleWithGuestTx` also signed with `RecipientEscrowPubKey`
- `PaymentSettleWithHostTx` if `NewGuestBalance === 0`, then it is `SettleOnlyWithHostTx(PendingPaymentTime)`; otherwise it is `SettleWithHostTx(PendingPaymentTime)` signed with `SenderEscrowPubKey`
	- `CompletePaymentSettleWithHostTx`: `PaymentSettleWithHostTx` also signed with `RecipientEscrowPubKey`
- `SenderRatchetTx`: `RatchetTx(SenderRatchetAccount, PendingPaymentTime)` signed with `RecipientEscrowPubKey`
- `RecipientRatchetTx`: `RatchetTx(RecipientRatchetAccount, PendingPaymentTime)` signed with `SenderEscrowPubKey`


### Messages
- every message contains
	- `Version` (currently 2)
	- `Body`: specific to message; contains a `EscrowAccount`
	- `MessageSignature`: a signature of `Version` and `Body` either signed by `HostAccount` or `GuestAccount`, depending on who is sender
- signatue can be verified by identifying what public key to use (using the `EscrowAccount` in the message or from entry `HostAccount` for `ChannelProposeMsg`)
#### `ChannelProposeMsg`
- sent by host
- field `EscrowAccount`, `GuestAccount`, `HostRatchetAccount`, `GuestRatchetAccount`, `MaxRoundDuration`, `FinalityDelay`, `Feerate`, `HostAmount`, `FundingTime`, `HostAccount`
- validation (by guest):
	- channel with `EscrowAccount` does not exist yet
	- guest does not have a channel with `HostAccount` as counterparty
	- `GuestAccount` is correct
	- the accounts `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` are created
	- values `MaxRoundDuration`, `FinalityDelay`, and `Feerate` are within bounds accepted by guest
	- `HostAmount` > 0
	- latest ledger timestamp between `FundingTime - MaxRoundDuration` and `FundingTime + MaxRoundDuration`
#### `ChannelAcceptMsg`
- sent by guest
- fields: `EscrowAccount`, `RatchetTx`, `SettleOnlyWithHostTx`
- validation (by host)
	- channel with `EscrowAccount` exists
	- channel is in `ChannelProposed` state
	- `RatchetTx` is valid and correctly signed
	- `SettleOnlyWithHostTx` is valid and signed
#### `PaymentProposeMsg`
- sent by sender
- fields: `EscrowAccount`, `RoundNumber`, `PaymentTime`, `PaymentAmount`, `PaymentSettleWithGuestTx`, `PaymentSettleWithHostTx`
- validation (by recipient):
	- channel with `EscrowAccount` exists
	- state in `Open`, `PaymentProposed` or `AwaitingPaymentMerge`
	- `PaymentAmount > 0` and less than counterparties channel balance
	- if state in `PaymentProposed`, then `RoundNumber === PaymentProposeMsg.RoundNumber`
	- if state is `Open` or `AwaitingPaymentMerge`, then
		- `RoundNumber + 1 === PaymentProposeMsg.RoundNumber`
		- `PaymentProposeMsg.PaymentTime` is within `MaxRoundDuration` difference of current ledger timestamp
		- `PaymentProposeMsg.PaymentTime` >= `PaymentTime`
	- `NewGuestBalance >= 0`
	- `PaymentSettleWithGuestTx` and `PaymentSettleWithHostTx` are valid and correctly signed
	- if state is `AwaitingPaymentMerge`, then `PaymentProposeMsg.PaymentAmount = TheirPaymentAmount - MyPaymentAmount`
#### `PaymentAcceptMsg`
- sent by recipient
- fields: `EscrowAccount`, `RoundNumber`, `SenderRatchetTx`, `CompletePaymentSettleWithGuestTx`, `CompletePaymentSettleWithHostTx`
- validation (by sender):
	- channel with `EscrowAccount` exists
	- channel is in state `PaymentProposed`
	- `PaymentAcceptMsg.RoundNumber === RoundNumber`
	- `SenderRatchetTx`, `CompletePaymentSettleWithGuestTx`, `CompletePaymentSettleWithHostTx` are valid and correctly signed
#### `PaymentCompleteMsg`
- sent by sender
- fields: `EscrowAccount`, `RoundNumber`, `RecipientRatchetTx`
- validation (by recipient):
	- channel with `EscrowAccount` exists
	- channel is in state `PaymentAccepted`
	- `PaymentCompleteMsg.RoundNumber === RoundNumber`
	- `RecipientRatchetTx` is valid and correctly signed
#### `CloseMsg`
- sent by closer
- fields: `EscrowAccount`, `CooperativeCloseTx`
- validation (by cooperator):
	- channel with `EscrowAccount` exists
	- channel is in state `Open`, `PaymentProposed` or `AwaitingClose`x
	- `CooperativeCloseTx` is valid and correctly signed (where _valid_ in case of state `PaymentProposed` refers to pre-payment balances)

### Transactions
#### `SetupAccountTx(AccountId)`
- source: `HostAccount`
- sequence number: `HostAccount.SequenceNumber + 1`
- operations: Create account `AccountId` with starting balance 1 XLM
#### `FundingTx`
- source: `HostAccount`
- sequence number: `HostAccount.SequenceNumber + 1`
- maxtime: `FundingTime + FinalityDelay + MaxRoundDuration`
- operations:
	- `EscrowAccount`: pay `HostAmount + 0.5 + 8 * Feerate`, add `GuestAccount` as cosigner
	- `GuestRatchetAccount`: pay `1 + Feerate`, remove master key as signer, add `EscrowAccount` and `GuestAccount` as signers
	- `HostRatchetAccount`: pay `0.5 + Feerate`, remove master key as signer, add `EscrowAccount` as signer
#### `CleanupTx`
- source: `HostAccount`
- sequence number: `HostAccount.SequenceNumber + 1`
- operations: merge `EscrowAccount`, `GuestRatchetAccount` and `HostRatchetAccount` into `HostAccount`
#### `RatchetTx(RatchetAccount, PaymentTime)`
- source: `RatchetAccount`
- sequence number: `RatchetAccount.SequenceNumber + 1`
- maxtime: `PaymentTime + FinalityDelay + MaxRoundDuration`
- operations: bump sequence of `EscrowAccount` to `RoundSequenceNumber + 1`
#### `SettleOnlyWithHostTx(PaymentTime)`
- source: `EscrowAccount`
- sequence number: `RoundSequenceNumber + 2`
- mintime: `PaymentTime + 2 * FinalityDelay + MaxRoundDuration`
- operations: merge `EscrowAccount`, `GuestRatchetAccount` and `HostRatchetAccount` into `HostAccount`
#### `SettleWithGuestTx(PaymentTime, GuestAmount)`
- source: `EscrowAccount`
- sequence number: `RoundSequenceNumber + 2`
- mintime: `PaymentTime + 2 * FinalityDelay + MaxRoundDuration`
- operations: pay `GuestAmount` from `EscrowAccount` to `GuestAccount`
#### `SettleWithHostTx(PaymentTime)`
- source: `EscrowAccount`
- sequence number: `RoundSequenceNumber + 3`
- mintime: `PaymentTime + 2 * FinalityDelay + MaxRoundDuration`
- operations: merge `EscrowAccount`, `GuestRatchetAccount` and `HostRatchetAccount` into `HostAccount`
#### `CooperativeCloseTx`
- source: `EscrowAccount`
- sequence number: `HostAccount.SequenceNumber + 1`
- operations:
	- (only if `GuestBalance > 0`): pay `GuestBalance` from `EscrowAccount` to `GuestAccount`
	- merge `EscrowAccount`, `GuestRatchetAccount` and `HostRatchetAccount` into `HostAccount`
#### `TopUpTx(TopupAmount)`
- source: `HostAccount`
- sequence number: `HostAccount.SequenceNumber + 1`
- operations: pay `TopupAmount` to `EscrowAccount`

### User Commands
#### `CreateChannelCmd`
- fields
	- `Counterparty` (= stellar federation address or stellar account)
	- `MaxRoundDuration`
	- `FinalityDelay`
	- `HostAmount` (the amount for funding the channel)
#### `ChannelPayCmd`
- fields
	- `EscrowAccount`
	- `PaymentAmount`
#### `TopUpCmd`
- only possible for host in `Open` state
- fields
	- `EscrowAccount`
	- `Amount`
#### `CloseChannelCmd`
- only possible for agent in `Open` state
- fields
	- `EscrowAccount`
#### `ForceCloseCmd`
- only possible for agent not in any `Force closing state`
- fields
	- `EscrowAccount`
#### `CleanUpCmd`
- only possible for host in `ChannelProposed` state
- fields: `EscrowAccount`
- host does
	- create, sign and submit `CleanupTx` and transition to `AwaitingCleanup` state
	- if transaction is on chain, transition to `Closed` state

### Timers
- these timers trigger events when the ledger times reaches their respecive values
- for every channel there is always at most one timer active
#### `PreFundTimeout`
- only active in state `AwaitingFunding`
- expires at `FundingTime + FinalityDelay + MaxRoundDuration`
- on expiry
	- host
		- create, sign and submit `CleanupTx` and transition to `AwaitingCleanup` state
		- if transaction is on chain, transition to `Closed` state
	- guest: transition to `Closed` state
#### `ChannelProposedTimeout`
- only active in state `ChannelProposed`
- expired at `FundingTime + MaxRoundDuration`
- on expiry (host only)
	- create, sign and submit `CleanupTx` and transition to `AwaitingCleanup` state
	- if transaction is on chain, transition to `Closed` state
#### `RoundTimeout`
- only active in state `Open`, `PaymentProposed`, `PaymentAccepted` or `AwaitingClose`
- expires at `PaymentTime + MaxRoundDuration`
- on expiry
	- submit `CurrentRatchetTx`
	- goes into `AwaitingRatchet` state
#### `SettlementMintimeTimeout`
- only active in state `AwaitingSettlementMintime`
- expires at mintime of `CurrentSettlementTxes`
- on expiry
	- submit `CurrentSettlementTxes`
	- goes to `AwaitingSettlement` state

### Fees
- the `Feerate` parameter of a channel determines the fees for transactions with source `EscrowAccount`, `HostRatchetAccount` or `GuestRatchetAccount` (i.e, `Feerate` times number of transactions)
- should be higher than 100 stroops to make sure that these transactions get included promptly in the ledger (for safety reasons)

### Timing Parameters
if set too low, then liveness and safety set at risk; if set too high, then unnecessary delays
- `MaxRoundDuration`
	- if one party initiates round `R` when ledger timestamp is `T`, then parties should by able to complete round `R+1` before ledger timestamp > `T + MaxRoundDuration`
	- if set too low, threatens liveness of channel
- `FinalityDelay`
	- if any event happens on ledger at timestamp `T`, then a party should be able to observe that, create and submit a new transaction and have it included on ledger before `T + FinalityDelay`


### Stellar Protocol Background
- each accounts starts with a single <span style="color:blue">_signer_</span>, the <span style="color:blue">_master key_</span> (= <span style="color:blue">_account ID_</span>) and minimum balance 1 XLM
- every additional signer requires 0.5 XLM additional minimum balance
- every account has a <span style="color:blue">_sequence number_</span>
- every transaction has a <span style="color:blue">_source account_</span>: it pays transaction fees and its sequence number will be incremented
- a transaction can have a <span style="color:blue">_maxtime_</span> and/or <span style="color:blue">_mintime_</span>
- a transaction must be authorized by its own source account and the source account of every operation
- a transaction is <span style="color:blue">_invalid_</span> if not sufficiently authorized, sequence number mismatches, insufficient balances for fees or mintime/maxtime mismatch
- a transaction <span style="color:blue">_fails_</span> if any of the operations fails
	- (!!!) *no operations are executed but the transaction is still included in the ledger, sequence number incremented and transaction fees paid*

## Used Synonyms
The following notions are (almost) used synonymously in the original protocol definition. In this document we only ever use the first name on every line.
- `HostAccount` = `HostAccountKey`
- `GuestAccount` = `GuestAccountKey` = `GuestEscrowPubKey`
- `EscrowAccount` = `HostEscrowPubKey` = `ChannelID`
- `HostRatchetAccount` = `FirstThrowawayPubKey`
- `GuestRatchetAccount` = `SecondThrowawayPubKey`
- `EscrowAccountSequenceNumber` = `BaseSequenceNumber`
- `PaymentSettleWithGuestTx` (= `SenderSettleWithGuestSig`)
- `PaymentSettleWithHostTx` (= `SenderSettleWithHostSig`)
- `SenderRatchetTx` (= `RecipientRatchetSig`)
- `RecipientRatchetTx` (= `SenderRatchetSig`)