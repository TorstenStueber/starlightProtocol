Initially both parties are in the `Start` state

# Preface
- each party has a **user**, an **agent** and an account
  - `HostAccount`, created account, not multisig
  - `GuestAcount`, created account, not multisig
- the user can send **commands** to its own agent; the agents can send **messages** to each other
  - there are 6 commands: `CreateChannelCmd`, `ChannelPayCmd`, `TopUpCmd`, `CloseChannelCmd`, `ForceCloseCmd`, `CleanUpCmd`
  - there are 6 messages: `ChannelProposeMsg`, `ChannelAcceptMsg`, `PaymentProposeMsg`, `PaymentAcceptMsg`, `PaymentCompleteMsg`, `CloseMsg`
- each agent stores the following values for every channel (where a channel is identified by the `EscrowAccountId`):
  - `MaxRoundDuration`
    - a duration, constant throughout the lifetime of one channel
    - defines the maximal time the channel can stay open after the last payment
  - `FinalityDelay`
    - a duration, constant throughout the lifetime of one channel
    - defines the maximal reaction time for a party to submit the latest ratchet transaction upon observing that the counterparty submitted an out of date ratchet transaction
  - `Feerate`
    - an unsigned integer, constant throughout the lifetime of one channel
    - defines the transaction fees per operation the parties are willing to spend
  - `EscrowAccount`
    - an account, constant throughout the lifetime of one channel
    - the used escrow account, containing the account id and a sequence number
  - `HostRatchetAccount`
    - an account, constant throughout the lifetime of one channel
    - the used host ratchet account, containing the account id and a sequence number
  - `GuestRatchetAccount`
    - an account, constant throughout the lifetime of one channel
    - the used guest ratchet account, containing the account id and a sequence number
  - `CosignerSecret`
    - a secret key, constant throughout the lifetime of one channel
    - the parties secret key for cosigning ratchet and settlement transactions
  - `CounterCosignerPublic`
    - a public key, constant throughout the lifetime of one channel
    - the public key of the counterparties `CosignerSecret` – for validating signatures
  - `HostAccountId`
    - a account id, constant throughout the lifetime of one channel
    - the account id of the host account
  - `GuestAccountId`
    - a account id, constant throughout the lifetime of one channel
    - the account id of the guest account
  - `Role`
    - either `"Host"` or `"Guest"`, constant throughout the lifetime of one channel
    - defines whether this is the host agent or the guest agent
  - `RoundNumber`
    - an unsigned integer
    - the current number of the payment round, will increase by one for every payment inside the channel
  - `HostBalance`
    - an unsigned integer
    - the current balance of host in the channel
  - `GuestBalance`
    - an unsigned integer
    - the current balance of guest in the channel
  - `CurrentRatchetEnvelope`
    - a transaction envelope
    - optional: guest only has this value after the first payment
    - envelope of the latest ratchet transaction, signed by both parties
  - `CurrentSettlementEnvelopes`
    - an array of transaction envelopes
    - optional: guest only has this value after the first payment
    - envelopes of the settlement transactions belonging to `CurrentRatchetEnvelope`, signed by both parties
  - `CounterpartyLatestSettlementEnvelopes`
    - an array of transaction envelopes, optional
    - optional: guest only has this value after the first payment
    - the latest settlement transactions, signed by both parties
    - usually `CurrentSettlementEnvelopes` and `CounterpartyLatestSettlementEnvelopes` are equal; however, while a recipient of a payment is in state `PaymentAcceptMsg`, this will already be the settlement transaction of the next round (while `CurrentRatchetEnvelope` is still the ratchet of the current roud)
  - `PaymentTime`
    - a date
    - either the date of the channel creation (when there was no payment yet) or the date of the first payment
  - `PendingPaymentTime`
    - a date
    - optional, only used during payment
    - the time of the currently negotiated payment
  - `PendingPaymentAmount`
    - an unsinged integer
    - optional, only used during payment
    - the payment amount of the currently negotiated payment
  - `SettleWithGuestTx`
    - a transaction
    - optional, only used during payment
    - the next settlement transaction for guest
  - `SettleWithHostTx`
    - a transaction
    - optional, only used during payment
    - the next settlement transaction for host
  - `TheirPaymentAmount`
    - an unsinged integer
    - optional, only used in state `AwaitingPaymentMerge`
    - the amount paid by the counterparty
  - `MyPaymentAmount`
    - an unsinged integer
    - optional, only used in state `AwaitingPaymentMerge`
    - the amount paid by this party
- in addition to that it maintains a **state** for each channel, which can be either one of the values:
  - `Start`, `SettingUp`, `ChannelProposed`, `AwaitingFunding`, `AwaitingCleanup`,
  `Open`, `PaymentProposed`, `PaymentAccepted`, `AwaitingPaymentMerge`, `AwaitingRatchet`,
  `AwaitingSettlementMintime`, `AwaitingSettlement`, `Closed`, `AwaitingClose`

# Workflows
## Create a channel
Both agents start in the `Start` state for a new channel.

### Host in `Start` state
- if receives `CreateChannelCmd` from user
  - command contains
    - `Counterparty` (= stellar federation address or stellar account),
    - `MaxRoundDuration`,
    - `FinalityDelay`,
    - `HostAmount` (the amount for funding the channel)
  - generate keypairs `EscrowKeypair`, `HostRatchetKeypair`, `GuestRatchetKeypair`
  - choose parameter `Feerate`
  - determine `GuestAccountId` from `CreateChannelCmd.Counterparty` (e.g., using federation server)
  - put
    - `CosignerSecret = secretKey(EscrowKeypair)`
    - `CounterCosignerPublic = GuestAccountId`
    - `HostAccountId = id(HostAccount)`
    - `Role = "Host"`
    - `RoundNumber = 1`
    - `HostBalance = CreateChannelCmd.HostAmount`
    - `GuestBalance = 0`
  - create `SetupAccounTxes` consisting of
    - `SetupAccountTx(HostAccount, 1, hostRatchetKeypair)`
    - `SetupAccountTx(HostAccount, 2, guestRatchetKeypair)`
    - `SetupAccountTx(HostAccount, 3, escrowKeypair)`
  - sign `SetupAccountTxes` with `secretKey(HostAccount)`
  - submit `SetupAccountTxes`
  - go to `SettingUp` state

### Host in `SettingUp` state
- wait for each `SetupAccountTxes` to either succeed or fail
- load `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` from horizon
- put `PaymentTime` = most recent ledger timestamp
- send `ChannelProposeMsg` to guest with fields
  - `EscrowAccountId = publicKey(EscrowKeypair)`
  - `HostRatchetAccountId = publicKey(HostRatchetKeypair)`
  - `GuestRatchetAccountId = publicKey(GuestRatchetKeypair)`
  - `MaxRoundDuration: CreateChannelCmd.MaxRoundDuration`
  - `FinalityDelay: CreateChannelCmd.FinalityDelay`
  - `Feerate`
  - `HostAmount: CreateChannelCmd.HostAmount`
  - `PaymentTime`
  - `HostAccountId = id(HostAccount)`
  - `GuestAccountId`
- go to `ChannelProposed` state

### Guest in `Start` state
- if receives `ChannelProposedMsg`
  - load `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` from horizon using `ChannelProposedMsg.EscrowAccountId`, `ChannelProposedMsg.HostRatchetAccountId`, `ChannelProposedMsg.GuestRatchetAccountId`
  - validate `ChannelProposedMsg`
    - guest does not have a channel with `HostAccount` as counterparty
    - `ChannelProposedMsg.GuestAccountId == id(GuestAccount)` is correct
    - `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` exist
    - values `ChannelProposedMsg.MaxRoundDuration`, `ChannelProposedMsg.FinalityDelay`, and `ChannelProposedMsg.Feerate` are within bounds accepted by guest
    - `ChannelProposedMsg.HostAmount > 0`
    - difference between `ChannelProposedMsg.PaymentTime` and latest ledger timestamp less than `ChannelProposedMsg.MaxRoundDuration`
  - put
    - `MaxRoundDuration = ChannelProposedMsg.MaxRoundDuration`
    - `FinalityDelay = ChannelProposedMsg.FinalityDelay`
    - `Feerate = ChannelProposedMsg.Feerate`
    - `CosignerSecret = secretKey(GuestAcount)`
    - `CounterCosignerPublic = ChannelProposedMsg.EscrowAccountId`
    - `HostAccountId = ChannelProposedMsg.HostAccountId`
    - `GuestAccountId = ChannelProposedMsg.GuestAccountId`
    - `Role = "Guest"`
    - `RoundNumber = 1`
    - `HostBalance = ChannelProposedMsg.HostAmount`
    - `GuestBalance = 0`
    - `PaymentTime = ChannelProposedMsg.PaymentTime`
  - create `SettleWithHostTx(GuestBalance, HostAccountId, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, 1, PaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - create `RatchetTx(HostRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + 5, PaymentTime + FinalityDelay + MaxRoundDuration)`
  - let
    - `GuestSettleWithHostTxSig = sign(SettleWithHostTx, CosignerSecret)`
    - `GuestRatchetSig = sign(RatchetTx, CosignerSecret)`
    - _This is an attack vector; guest blindly sings a transaction that merges some unknown accounts into `HostAccount` without proving that `HostAccount` has any rights to them; our one-way scheme instead lets host create these transactions and already signs them before sending them to guest, which proves to guest that this is legit_
  - send a `ChannelAcceptMsg` to host with fields
    - `EscrowAccountId: id(EscrowAccount)`
    - `GuestSettleWithHostTxSig`
    - `GuestRatchetSig`
  - go to `AwaitingFunding` state

### Host in `ChannelProposed` state
- if time `PaymentTime + MaxRoundDuration` expires
  - go to `AwaitingCleanup` state
- if receive `CleanUpCmd`
  - command contains
    - `EscrowAccountId`
  - go to `AwaitingCleanup` state
- if receives `ChannelAcceptMsg`
  - create `SettleWithHostTx(GuestBalance, HostAccountId, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, 1, PaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - create `RatchetTx(HostRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + 5, PaymentTime + FinalityDelay + MaxRoundDuration)`
  - validate `ChannelAcceptMsg`
    - `ChannelAcceptMsg.GuestSettleWithHostTxSig` is a valid signature of `SettleWithHostTx` for `CounterCosignerPublic`
    - `ChannelAcceptMsg.GuestRatchetSig` is a valid signature of `RatchetTx` for `CounterCosignerPublic`
  - put
    - `CurrentSettlementEnvelopes = [ (SettleWithHostTx, ChannelAcceptMsg.GuestSettleWithHostTxSig, sign(SettleWithHostTx, Cosigner) ]`
    - `CurrentRatchetEnvelope = (RatchetTx, ChannelAcceptMsg.GuestRatchetSig, sign(RatchetTx, Cosigner))`
  - create `FundingTx(HostAccount, GuestAccountId, EscrowAccount, HostRatchetAccount, GuestRatchetAccount, PaymentTime + FinalityDelay + MaxRoundDuration, HostBalance, Feerate)`
  - sign `FundingTx` with `secretKey(HostAccount)`, `secretKey(EscrowKeypair)`, `secretKey(HostRatchetKeypair)`, `secretKey(GuestRatchetKeypair)`
  - submit `FundingTx`
  - go to `AwaitingFunding` state

### Host and guest in `AwaitingFunding` state
- once they observe that `FundingTx` is on chain:
  - host
    - go to `Open` state
  - guest
    - let `SequenceNumber = sequenceNumber(EscrowAccount)`
    - reload `EscrowAccount`, `HostRatchetAccount`, `GuestRatchetAccount` from horizon
    - check
      - `sequenceNumber(EscrowAccount) === SequenceNumber`
        - _I removed the sequence number check of GuestRatchetAccount; it was pointless_
      - balance of `EscrowAccount` is at least `HostAmount + 1.5 + 8 * Feerate`
      - signers of `EscrowAccount`, `GuestRatchetAccount` are as expected
        - _I removed the check of HostRatchetAccount; it was pointless_
    - if any check fails, then go to `Closed` state, otherwise go to `Open` state
- if `PaymentTime + FinalityDelay + MaxRoundDuration` expires, then
  - host
    - go to `AwaitingCleanup` state
  - guest
    - go to `Closed` state

### Host in `AwaitingCleanup` state
- create `CleanupTx(HostAccount, EscrowAccount, GuestRatchetAccount, HostRatchetAccount)`
- sign `CleanupTx` with `secretKey(HostAccount)`, `secretKey(EscrowKeypair)`, `secretKey(HostRatchetKeypair)`, `secretKey(GuestRatchetKeypair)`
- submit `CleanupTx` and wait for success
- go to `Closed` state

## Payments and Top-Up
From <span style="color:blue">_sender_</span> to <span style="color:blue">_recipient_</span>

### Agent in `Open` state
- if receives `ChannelPayCmd` from user
  - command contains
    - `EscrowAccountId`
    - `PaymentAmount`
  - let `SenderBalance = HostBalance` if `Role === "Host"`, otherwise `SenderBalance = GuestBalance`
  - validate `ChannelPayCmd`
    - `ChannelPayCmd.PaymentAmount <= SenderBalance`
  - put
    - `PendingPaymentTime` = max(most recent ledger timestamp, `PaymentTime`)
    - `PendingPaymentAmount = ChannelPayCmd.PaymentAmount`
    - `RoundNumber = RoundNumber + 1`
  - label `sendPaymentProposeMsg` (jump destination from somewhere else)
  - let `NewGuestBalance = GuestBalance + PendingPaymentAmount` if `Role === "Host"`, otherwise `NewGuestBalance = GuestBalance - PendingPaymentAmount`
  - create `SettleWithGuestTx(NewGuestBalance, EscrowAccount, GuestAccountId, RoundNumber, PendingPaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - create `SettleWithHostTx(NewGuestBalance, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, RoundNumber, PendingPaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - let
    - `SenderSettleWithGuestSig = sign(SettleWithGuestTx, CosignerSecret)`
    - `SenderSettleWithHostSig = sign(SettleWithHostTx, CosignerSecret)`
  - send a `PaymentProposeMsg` to recipient with fields
    - `EscrowAccountId: id(EscrowAccount)`
    - `RoundNumber`
    - `PaymentTime: PendingPaymentTime`
    - `PaymentAmount: PendingPaymentAmount`
    - `SenderSettleWithGuestSig`
    - `SenderSettleWithHostSig`
  - go to `PaymentProposed` state
- if receives `TopUpCmd` from user
  - command contains
    - `EscrowAccountId`
    - `Amount`
  - validate
    - `Role === "Host"`
  - create `TopUpTx(HostAccount, EscrowAccount, TopUpCmd.Amount)`
  - sign `TopUpTx` with `secretKey(HostAccount)`
  - submit `TopUpTx`

### Agent in `Open`, `PaymentProposed` or `AwaitingPaymentMerge` state
- if receives `PaymentProposeMsg`
  - let `SenderBalance = HostBalance` if `Role === "Guest"`, otherwise `SenderBalance = GuestBalance`
  - validate
    - `SenderBalance >= PaymentProposeMsg.PaymentAmount > 0`
  - if state is `PaymentProposed`
    - validate
      - `RoundNumber === PaymentProposeMsg.RoundNumber`
    - if `PendingPaymentAmount > PaymentProposeMsg.PaymentAmount` or (`PendingPaymentAmount === PaymentProposeMsg.PaymentAmount` and `Role === "Host"`)
      - put
        - `RoundNumber = RoundNumber + 1`
        - `PendingPaymentAmount = PendingPaymentAmount - PaymentProposeMsg.PaymentAmount`
      - jump to label `sendPaymentProposeMsg`
    - else
      - put
        - `RoundNumber = RoundNumber + 1`
          - _This seems to be wrong from my persective; the RoundNumber will be incremented when the next_
        - `TheirPaymentAmount = PaymentProposeMsg.PaymentAmount`
        - `MyPaymentAmount = PendingPaymentAmount`
      - go to `AwaitingPaymentMerge` state
  - else
    - let `NewGuestBalance = GuestBalance + PaymentProposeMsg.PaymentAmount` if `Role === "Guest"`, otherwise `NewGuestBalance = GuestBalance - PaymentProposeMsg.PaymentAmount`
    - create `SettleWithGuestTx(NewGuestBalance, EscrowAccount, GuestAccountId, PaymentProposeMsg.RoundNumber, PaymentProposeMsg.PaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
    - create `SettleWithHostTx(NewGuestBalance, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, PaymentProposeMsg.RoundNumber, PaymentProposeMsg.PaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
    - validate
      - `RoundNumber + 1 === PaymentProposeMsg.RoundNumber`
      - `PaymentProposeMsg.PaymentTime` is within `MaxRoundDuration` difference of current ledger timestamp
      - `PaymentProposeMsg.PaymentTime` >= `PaymentTime`
      - `NewGuestBalance >= 0`
        - _This check is also not needed_
      - `ChannelAcceptMsg.SenderSettleWithGuestSig` is a valid signature of `SettleWithGuestTx` for `CounterCosignerPublic`
      - `ChannelAcceptMsg.SenderSettleWithHostSig` is a valid signature of `SettleWithHostTx` for `CounterCosignerPublic`
      - if state is `AwaitingPaymentMerge`, then `PaymentProposeMsg.PaymentAmount = TheirPaymentAmount - MyPaymentAmount`
    - put
      - `RoundNumber = PaymentProposeMsg.RoundNumber`
      - `PendingPaymentAmount = PaymentProposeMsg.PaymentAmount`
      - `PendingPaymentTime = PaymentProposeMsg.PaymentTime`
    - let `SenderRatchetAccount = HostRatchetAccount` if `Role === "Guest"`, `GuestRatchetAccount` otherwise
    - create `RatchetTx(SenderRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
    - let
      - `RecipientRatchetSig = sign(RatchetTx, CosignerSecret)`
      - `RecipientSettleWithGuestSig = sign(SettleWithGuestTx, CosignerSecret)`
      - `RecipientSettleWithHostSig = sign(SettleWithHostTx, CosignerSecret)`
    - put `CounterpartyLatestSettlementEnvelopes` to the array with entries
      - `(SettleWithGuestTx, ChannelAcceptMsg.SenderSettleWithGuestSig, RecipientSettleWithGuestSig)`
      - `(SettleWithHostTx, ChannelAcceptMsg.SenderSettleWithHostSig, RecipientSettleWithHostSig)`
    - send a `PaymentAcceptMsg` with fields
      - `EscrowAccount: id(EscrowAccount)`
      - `RoundNumber`
      - `RecipientRatchetSig`
      - `RecipientSettleWithGuestSig`
      - `RecipientSettleWithHostSig`
    - go to `PaymentAccepted` state

### Agent in `PaymentProposed` state
- if receives `PaymentAcceptMsg`
  - let `SenderRatchetAccount = HostRatchetAccount` if `Role === "Host"`, `GuestRatchetAccount` otherwise
  - create `SenderRatchetTx = RatchetTx(SenderRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - validate `PaymentAcceptMsg`
    - `PaymentAcceptMsg.RoundNumber === RoundNumber`
    - `ChannelAcceptMsg.RecipientRatchetSig` is a valid signature of `SenderRatchetTx` for `CounterCosignerPublic`
    - `ChannelAcceptMsg.RecipientSettleWithGuestSig` is a valid signature of `SettleWithGuestTx` for `CounterCosignerPublic`
    - `ChannelAcceptMsg.RecipientSettleWithHostSig` is a valid signature of `SettleWithHostTx` for `CounterCosignerPublic`
  - let `RecipientRatchetAccount = HostRatchetAccount` if `Role === "Guest"`, `GuestRatchetAccount` otherwise
  - create `RecipientRatchetTx = RatchetTx(RecipientRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - let
    - `SenderRecipientRatchetSig = sign(RecipientRatchetTx, CosignerSecret)`
  - put
    - `CurrentRatchetEnvelope = (SenderRatchetTx, ChannelAcceptMsg.RecipientRatchetSig, sign(SenderRatchetTx, CosignerSecret))`
    - `GuestBalance = GuestBalance + PendingPaymentAmount` if `Role === "Host"`, otherwise `GuestBalance = GuestBalance - PendingPaymentAmount`
    - `HostBalance = HostBalance + PendingPaymentAmount` if `Role === "Guest"`, otherwise `HostBalance = HostBalance - PendingPaymentAmount`
    - `PaymentTime = PendingPaymentTime`
    - `CurrentSettlementEnvelopes = CounterpartyLatestSettlementEnvelopes` to the array with entries
      - `(SettleWithGuestTx, ChannelAcceptMsg.RecipientSettleWithGuestSig, sign(SettleWithGuestTx, CosignerSecret))`
      - `(SettleWithHostTx, ChannelAcceptMsg.RecipientSettleWithHostSig, sign(SettleWithHostTx, CosignerSecret))`
  - send a `PaymentCompleteMsg` with fields
      - `EscrowAccount: id(EscrowAccount)`
      - `RoundNumber`
      - `SenderRatchetSig: SenderRecipientRatchetSig`
  - go to `Open` state

### Agent in `PaymentAccepted` state
- if receives `PaymentCompleteMsg`
  - let `RecipientRatchetAccount = HostRatchetAccount` if `Role === "Host"`, `GuestRatchetAccount` otherwise
  - create `RatchetTx(RecipientRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - validate `PaymentCompleteMsg`
    - `PaymentCompleteMsg.RoundNumber === RoundNumber`
    - `ChannelAcceptMsg.SenderRatchetSig` is a valid signature of `RatchetTx` for `CounterCosignerPublic`
  - put
    - `CurrentRatchetEnvelope = (RatchetTx, ChannelAcceptMsg.SenderRatchetSig, sign(RatchetTx, CosignerSecret))`
    - `GuestBalance = GuestBalance + PendingPaymentAmount` if `Role === "Guest"`, otherwise `GuestBalance = GuestBalance - PendingPaymentAmount`
    - `HostBalance = HostBalance + PendingPaymentAmount` if `Role === "Host"`, otherwise `HostBalance = HostBalance - PendingPaymentAmount`
    - `PaymentTime = PendingPaymentTime`
    - `CurrentSettlementEnvelopes = CounterpartyLatestSettlementEnvelopes`
      - _This is a little optimization_
  - go to `Open` state

## Cooperative Closing
### Agent in `Open` state
- if receives `CloseChannelCmd`
  - command contains
    - `EscrowAccountId`
  - load `HostAccount` from horizon using `HostAccountId`
  - construct a `CooperativeCloseTx(HostAccount, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, GuestBalance, GuestAccountId)`
  - let `CooperativeCloseSig = sign(CooperativeCloseTx, CosignerSecret)`
  - send a `CloseMsg` with fields
      - `EscrowAccount: id(EscrowAccount)`
      - `CooperativeCloseSig`
  - go to `AwaitingClose` state

### Agent in `Open`, `PaymentProposed` or `AwaitingClose` state
- if receives `CloseMsg`
  - load `HostAccount` from horizon using `HostAccountId`
  - construct a `CooperativeCloseTx(HostAccount, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, GuestBalance, GuestAccountId)`
  - validate `CloseMsg`
    - `ChannelAcceptMsg.SenderRatchetSig` is a valid signature of `RatchetTx` for `CounterCosignerPublic`
  - let `CooperativeCloseSig = sign(CooperativeCloseTx, CosignerSecret)`
  - submit `(CooperativeCloseTx, CooperativeCloseSig, CloseMsg.CooperativeCloseSig)` to horizon
  - if submission successful or `CurrentRatchetEnvelope` does not exist
    - go to `Closed` state
  - otherwise
    - submit `CurrentRatchetEnvelope` to horizon
    - go to `AwaitingRatchet` state

## Force closing
### Agent in `Open`, `PaymentProposed`, `PaymentAccepted`, `AwaitingPaymentMerge` or `AwaitingClose` state
- if time `PaymentTime + MaxRoundDuration` expires or if receives `ForceCloseCmd`
  - if `CurrentRatchetEnvelope` does not exist
    - go to `Closed` state
  - submit `CurrentRatchetEnvelope` to horizon
  - go to `AwaitingRatchet` state

### Agent in `AwaitingRatchet` state
- once it observes that `CurrentRatchetEnvelope` is on chain
  - go to `AwaitingSettlementMintime` state

### Agent in `AwaitingSettlementMintime` state
- if mintime of `CurrentSettlementEnvelopes` expires
  - submit `CurrentSettlementEnvelopes`
  - go to `AwaitingSettlement` state

### Agent in `AwaitingSettlement` state
- once it observes that `CurrentSettlementEnvelopes` is on chain
  - go to `Closed` state

# Monitoring
The following conditions need to be constantly monitored by both agents and reacted accordingly
- if `EscrowAccount` receives payments
  - increase `HostBalance` by that value
- if `CooperativeCloseTx` gets on chain
  - go to `Closed` state
- if counterparty publishes some `RatchetTx`
  - if that bumps the sequence number of `EscrowAccount` to a lower number than `CurrentRatchetEnvelope` does
    - submit `CurrentRatchetEnvelope` to horizon
    - go to `AwaitingRatchet` state
  - if that bump the sequence number of `EscrowAccount` to the same number as `CurrentRatchetEnvelope` does
    - go to `AwaitingSettlementMintime` state
  - if that bumps the sequence number of `EscrowAccount` to a higher number than `CurrentRatchetEnvelope` does
    - put `CurrentSettlementEnvelopes = CounterpartyLatestSettlementEnvelopes`
    - if in state `AwaitingSettlement`
      - go to state `AwaitingSettlementMintime`


# Transactions
`SetupAccountTx(HostAccount, Increment, Keypair)`
- source: `id(HostAccount)`
- sequence number: `sequenceNumber(HostAccount) + Increment`
- operations: Create account `publicKey(Account)` with starting balance 1 XLM

`FundingTx(HostAccount, GuestAccountId, EscrowAccount, HostRatchetAccount, GuestRatchetAccount, Maxtime, HostBalance, Feerate)`
- source: `id(HostAccount)`
- sequence number: `sequenceNumber(HostAccount) + 1`
- maxtime: `Maxtime`
- operations:
  - `id(EscrowAccount)`: pay `HostBalance + 0.5 + 8 * Feerate`, add `GuestAccountId` as cosigner
  - `id(GuestRatchetAccount)`: pay `1 + Feerate`, remove master key as signer, add `id(EscrowAccount)` and `GuestAccountId` as signers
  - `id(HostRatchetAccount)`: pay `0.5 + Feerate`, remove master key as signer, add `id(EscrowAccount)` as signer

`CleanupTx(HostAccount, EscrowAccount, GuestRatchetAccount, HostRatchetAccount)`
- source: `id(HostAccount)`
- sequence number: `sequenceNumber(HostAccount) + 1`
- operations: merge `id(EscrowAccount)`, `id(GuestRatchetAccount)` and `id(HostRatchetAccount)` into `id(HostAccount)`

`RatchetTx(RatchetAccount, EscrowAccount, SequenceNumber, Maxtime)`
- source: `id(RatchetAccount)`
- sequence number: `sequenceNumber(RatchetAccount) + 1`
- maxtime: `Maxtime`
- operations: bump sequence of `id(EscrowAccount)` to `SequenceNumber`

`SettleWithGuestTx(NewGuestBalance, EscrowAccount, GuestAccountId, RoundNumber, Mintime)`
  - if `NewGuestBalance === 0`
    - undefined
  - otherwise
    - source: `id(EscrowAccount)`
    - sequence number: `sequenceNumber(EscrowAccount) + RoundNumber * 4 + 2`
      - _why multiplication by 4; wouldn't 3 be enough?_
    - mintime: `Mintime`
    - operations: pay `NewGuestBalance` from `id(EscrowAccount)` to `GuestAccountId`

`SettleWithHostTx(NewGuestBalance, HostAccountId, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, RoundNumber, Mintime)`
  - if `NewGuestBalance === 0`
    - source: `id(EscrowAccount)`
    - sequence number: `sequenceNumber(EscrowAccount) + RoundNumber * 4 + 2`
    - mintime: `Mintime`
    - operations: merge `id(EscrowAccount)`, `id(GuestRatchetAccount)` and `id(HostRatchetAccount)` into `HostAccountId`
  - otherwise
    - source: `id(EscrowAccount)`
    - sequence number: `sequenceNumber(EscrowAccount) + RoundNumber * 4 + 3`
    - mintime: `Mintime`
    - operations: merge `id(EscrowAccount)`, `id(GuestRatchetAccount)` and `id(HostRatchetAccount)` into `HostAccountId`

`CooperativeCloseTx(HostAccount, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, GuestBalance, GuestAccountId)`
- source: `id(EscrowAccount)`
- sequence number: `sequenceNumber(HostAccount) + 1`
- operations:
  - (only if `GuestBalance > 0`): pay `GuestBalance` from `id(EscrowAccount)` to `GuestAccountId`
  - merge `id(EscrowAccount)`, `id(GuestRatchetAccount)` and `id(HostRatchetAccount)` into `id(HostAccount)`

`TopUpTx(HostAccount, EscrowAccount, TopupAmount)`
- source: `id(HostAccount)`
- sequence number: `sequenceNumber(HostAccount) + 1`
- operations: pay `TopupAmount` to `id(EscrowAccount)`
