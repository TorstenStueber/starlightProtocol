Initially both parties are in the `Start` state

# Preface
- each party has a **user**, an **agent** and an account
  - `HostAccount`, created account, not multisig
  - `GuestAcount`, created account, not multisig
- the user can send **commands** to its own agent; the agents can send **messages** to each other
  - there are 6 commands: `CreateChannelCmd`, `ChannelPayCmd`, `TopUpCmd`, `CloseChannelCmd`, `ForceCloseCmd`, `CleanUpCmd`
  - there are 6 messages: `ChannelProposeMsg`, `ChannelAcceptMsg`, `PaymentProposeMsg`, `PaymentAcceptMsg`, `PaymentCompleteMsg`, `CloseMsg`
- each agent stores the following values for every channel (where a channel is identified by the `EscrowAccountId`):
  - `const MaxRoundDuration: Duration`
  - `const FinalityDelay: Duration`
  - `const Feerate: Account`
  - `const EscrowAccount: Account`
  - `const HostRatchetAccount: Account`
  - `const GuestRatchetAccount: Account`
  - `const CosignerSecret: SecretKey`
  - `const CounterCosignerPublic: PublicKey`
  - `const HostAccountId: string`
  - `const GuestAccountId: string`
  - `const Role: "Role"` or `"Guest"`
  - `RoundNumber: uint`
  - `HostBalance: uint`
  - `GuestBalance: uint`
  - `CurrentRatchetEnvelope: TransactionEnvelope` (optional)
  - `CurrentSettlementEnvelopes: Array<TransactionEnvelope>` (optional)
  - `CounterpartyLatestSettlementEnvelopes: Array<TransactionEnvelope>` (optional)
  - `PaymentTime: Date`
  - `PendingPaymentTime: Date` (optional, only used during payment)
  - `PendingPaymentAmount: uint` (optional, only used during payment)
  - `SettleWithGuestTx: Transaction` (optional, only used during payment)
  - `SettleWithHostTx: Transaction` (optional, only used during payment)
  - `TheirPaymentAmount: uint` (optional, only used in state `AwaitingPaymentMerge`)
  - `MyPaymentAmount: uint` (optional, only used in state `AwaitingPaymentMerge`)
- in addition to that it maintains a **state** which can be either one of the values:
  - `Start, SettingUp, ChannelProposed, AwaitingFunding, AwaitingCleanup,
  Open, PaymentProposed, PaymentAccepted, AwaitingPaymentMerge, AwaitingRatchet,
  AwaitingSettlementMintime, AwaitingSettlement, Closed, AwaitingClose`

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
  - determine `GuestAccountId` from `Counterparty` (e.g., using federation server)
  - put
    - `CosignerSecret = secretKey(EscrowKeypair)`
    - `CounterCosignerPublic = GuestAccountId`
    - `HostAccountId = id(hostAccount)`
    - `Role = "Host"`
    - `RoundNumber = 1`
    - `HostBalance = HostAmount`
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
  - `MaxRoundDuration`
  - `FinalityDelay`
  - `Feerate`
  - `HostAmount`
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
  - let `SenderBalance = HostBalance` if agent is host, otherwise `SenderBalance = GuestBalance`
  - validate `ChannelPayCmd`
    - `ChannelPayCmd.PaymentAmount <= SenderBalance`
  - put
    - `PendingPaymentTime` = max(most recent ledger timestamp, `PaymentTime`)
    - `PendingPaymentAmount = ChannelPayCmd.PaymentAmount`
    - `RoundNumber = RoundNumber + 1`
  - label `sendPaymentProposeMsg` (jump destination from somewhere else)
  - let `NewGuestBalance = GuestBalance + PendingPaymentAmount` if agent is host, otherwise `NewGuestBalance = GuestBalance - PendingPaymentAmount`
  - create `SettleWithGuestTx(NewGuestBalance, EscrowAccount, GuestAccountId, RoundNumber, PendingPaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - create `SettleWithHostTx(NewGuestBalance, EscrowAccount, GuestRatchetAccount, HostRatchetAccount, RoundNumber, PendingPaymentTime + 2 * FinalityDelay + MaxRoundDuration)`
  - let
    -`SenderSettleWithGuestSig = sign(SettleWithGuestTx, CosignerSecret)`
    -`SenderSettleWithHostSig = sign(SettleWithHostTx, CosignerSecret)`
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
    - agent is host
  - create `TopUpTx(HostAccount, EscrowAccount, TopUpCmd.Amount)`
  - sign `TopUpTx` with `secretKey(HostAccount)`
  - submit `TopUpTx`

### Agent in `Open`, `PaymentProposed` or `AwaitingPaymentMerge` state
- if receives `PaymentProposeMsg`
  - let `SenderBalance = HostBalance` if agent is guest, otherwise `SenderBalance = GuestBalance`
  - validate
    - `SenderBalance >= PaymentProposeMsg.PaymentAmount > 0`
  - if state is `PaymentProposed`
    - validate
      - `RoundNumber === PaymentProposeMsg.RoundNumber`
    - if `PendingPaymentAmount > PaymentProposeMsg.PaymentAmount` or (`PendingPaymentAmount === PaymentProposeMsg.PaymentAmount` and agent is host)
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
    - let `NewGuestBalance = GuestBalance + PaymentProposeMsg.PaymentAmount` if agent is guest, otherwise `NewGuestBalance = GuestBalance - PaymentProposeMsg.PaymentAmount`
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
    - let `SenderRatchetAccount = HostRatchetAccount` if agent is guest, `GuestRatchetAccount` otherwise
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
  - let `SenderRatchetAccount = HostRatchetAccount` if agent is host, `GuestRatchetAccount` otherwise
  - create `SenderRatchetTx = RatchetTx(SenderRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - validate `PaymentAcceptMsg`
    - `PaymentAcceptMsg.RoundNumber === RoundNumber`
    - `ChannelAcceptMsg.RecipientRatchetSig` is a valid signature of `SenderRatchetTx` for `CounterCosignerPublic`
    - `ChannelAcceptMsg.RecipientSettleWithGuestSig` is a valid signature of `SettleWithGuestTx` for `CounterCosignerPublic`
    - `ChannelAcceptMsg.RecipientSettleWithHostSig` is a valid signature of `SettleWithHostTx` for `CounterCosignerPublic`
  - let `RecipientRatchetAccount = HostRatchetAccount` if agent is guest, `GuestRatchetAccount` otherwise
  - create `RecipientRatchetTx = RatchetTx(RecipientRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - let
    - `SenderRecipientRatchetSig = sign(RecipientRatchetTx, CosignerSecret)`
  - put
    - `CurrentRatchetEnvelope = (SenderRatchetTx, ChannelAcceptMsg.RecipientRatchetSig, sign(SenderRatchetTx, CosignerSecret))`
    - `GuestBalance = GuestBalance + PendingPaymentAmount` if agent is host, otherwise `GuestBalance = GuestBalance - PendingPaymentAmount`
    - `HostBalance = HostBalance + PendingPaymentAmount` if agent is guest, otherwise `HostBalance = HostBalance - PendingPaymentAmount`
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
  - let `RecipientRatchetAccount = HostRatchetAccount` if agent is host, `GuestRatchetAccount` otherwise
  - create `RatchetTx(RecipientRatchetAccount, EscrowAccount, sequenceNumber(EscrowAccount) + RoundNumber * 4 + 1, PendingPaymentTime + FinalityDelay + MaxRoundDuration)`
  - validate `PaymentCompleteMsg`
    - `PaymentCompleteMsg.RoundNumber === RoundNumber`
    - `ChannelAcceptMsg.SenderRatchetSig` is a valid signature of `RatchetTx` for `CounterCosignerPublic`
  - put
    - `CurrentRatchetEnvelope = (RatchetTx, ChannelAcceptMsg.SenderRatchetSig, sign(RatchetTx, CosignerSecret))`
    - `GuestBalance = GuestBalance + PendingPaymentAmount` if agent is guest, otherwise `GuestBalance = GuestBalance - PendingPaymentAmount`
    - `HostBalance = HostBalance + PendingPaymentAmount` if agent is host, otherwise `HostBalance = HostBalance - PendingPaymentAmount`
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
      - _why multiplication by 4; wouldn't 2 be enough?_
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
