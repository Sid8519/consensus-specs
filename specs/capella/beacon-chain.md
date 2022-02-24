# Capella -- The Beacon Chain

## Table of contents

<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

## Introduction

Capella is a consensus-layer upgrade containing a number of features related
to validator withdrawals. Including:
* Automatic withdrawals of `withdrawable` validators
* Partial withdrawals during block proposal
* Operation to change from `BLS_WITHDRAWAL_PREFIX` to
  `ETH1_ADDRESS_WITHDRAWAL_PREFIX` versioned withdrawal credentials to enable withdrawals for a validator

## Custom types

## Constants

We define the following Python custom types for type hinting and readability:

| Name | SSZ equivalent | Description |
| - | - | - |
| `TransactionType` | `Bytes1` | an EIP-2718 type |

## Preset

### State list lengths

| Name | Value | Unit | Duration |
| - | - | :-: | :-: |
| `WITHDRAWAL_TRANSACTION_LIMIT` | `uint64(2**40)` (= 1,099,511,627,776) | withdrawal transactions enqueued in state|

### Execution

| Name | Value | Description |
| - | - | - |
| `TX_TYPE_WITHDRAWAL` | `TransactionType('0x05')` | EIP-2718 TX Type |
| `MAX_WITHDRAWAL_TRANSACTIONS_PER_PAYLOAD` | `uint64(2**4)` (= 16) | Maximum amount of withdrawal transactions allowed in each payload |

## Configuration

## Containers

### Extended Containers

#### `Validator`

```python
class Validator(Container):
    pubkey: BLSPubkey
    withdrawal_credentials: Bytes32  # Commitment to pubkey for withdrawals
    effective_balance: Gwei  # Balance at stake
    slashed: boolean
    # Status epochs
    activation_eligibility_epoch: Epoch  # When criteria for activation were met
    activation_epoch: Epoch
    exit_epoch: Epoch
    withdrawable_epoch: Epoch  # When validator can withdraw funds
    withdrawn_epoch: Epoch  # [New in Capella]
```

#### `BeaconState`

```python
class BeaconState(Container):
    # Versioning
    genesis_time: uint64
    genesis_validators_root: Root
    slot: Slot
    fork: Fork
    # History
    latest_block_header: BeaconBlockHeader
    block_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    state_roots: Vector[Root, SLOTS_PER_HISTORICAL_ROOT]
    historical_roots: List[Root, HISTORICAL_ROOTS_LIMIT]
    # Eth1
    eth1_data: Eth1Data
    eth1_data_votes: List[Eth1Data, EPOCHS_PER_ETH1_VOTING_PERIOD * SLOTS_PER_EPOCH]
    eth1_deposit_index: uint64
    # Registry
    validators: List[Validator, VALIDATOR_REGISTRY_LIMIT]
    balances: List[Gwei, VALIDATOR_REGISTRY_LIMIT]
    # Randomness
    randao_mixes: Vector[Bytes32, EPOCHS_PER_HISTORICAL_VECTOR]
    # Slashings
    slashings: Vector[Gwei, EPOCHS_PER_SLASHINGS_VECTOR]  # Per-epoch sums of slashed effective balances
    # Participation
    previous_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    current_epoch_participation: List[ParticipationFlags, VALIDATOR_REGISTRY_LIMIT]
    # Finality
    justification_bits: Bitvector[JUSTIFICATION_BITS_LENGTH]  # Bit set for every recent justified epoch
    previous_justified_checkpoint: Checkpoint
    current_justified_checkpoint: Checkpoint
    finalized_checkpoint: Checkpoint
    # Inactivity
    inactivity_scores: List[uint64, VALIDATOR_REGISTRY_LIMIT]
    # Sync
    current_sync_committee: SyncCommittee
    next_sync_committee: SyncCommittee
    # Execution
    latest_execution_payload_header: ExecutionPayloadHeader
    # Withdrawals
    withdrawal_receipts: List[WithdrawalTransaction, WITHDRAWAL_TRANSACTION_LIMIT]  # [New in Capella]
```

#### `ExecutionPayload`

```python
class ExecutionPayload(Container):
    # Execution block header fields
    parent_hash: Hash32
    fee_recipient: ExecutionAddress  # 'beneficiary' in the yellow paper
    state_root: Bytes32
    receipt_root: Bytes32  # 'receipts root' in the yellow paper
    logs_bloom: ByteVector[BYTES_PER_LOGS_BLOOM]
    random: Bytes32  # 'difficulty' in the yellow paper
    block_number: uint64  # 'number' in the yellow paper
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[MAX_EXTRA_DATA_BYTES]
    base_fee_per_gas: uint256
    # Extra payload fields
    block_hash: Hash32  # Hash of execution block
    transactions: List[Transaction, MAX_TRANSACTIONS_PER_PAYLOAD]
    withdrawal_transactions: List[WithdrawalTransaction, MAX_WITHDRAWAL_TRANSACTIONS_PER_PAYLOAD]  # [New in Capella]
```

#### `ExecutionPayloadHeader`

```python
class ExecutionPayloadHeader(Container):
    # Execution block header fields
    parent_hash: Hash32
    fee_recipient: ExecutionAddress
    state_root: Bytes32
    receipt_root: Bytes32
    logs_bloom: ByteVector[BYTES_PER_LOGS_BLOOM]
    random: Bytes32
    block_number: uint64
    gas_limit: uint64
    gas_used: uint64
    timestamp: uint64
    extra_data: ByteList[MAX_EXTRA_DATA_BYTES]
    base_fee_per_gas: uint256
    # Extra payload fields
    block_hash: Hash32  # Hash of execution block
    transactions_root: Root
    withdrawal_transactions_root: Root  # [New in Capella]
```

### New containers

#### `WithdrawalTransaction`

New EIP-2718 transaction type, with the format being the single byte `TX_TYPE_WITHDRAWAL`
followed by an SSZ encoding of the `WithdrawalTransaction` container comprising the transaction contents.

```python
class WithdrawalTransaction(Container):
    address: ExecutionAddress
    value: Gwei
```

## Helpers

### Beacon state mutators

#### `withdraw`

```python
def withdraw(state: BeaconState, index: ValidatorIndex, amount: Gwei) -> None:
    # Decrease the validator's balance
    decrease_balance(state, index, amount)
    # Create a corresponding withdrawal receipt
    receipt = WithdrawalTransaction(
        address=state.validators[index].withdrawal_credentials[12:],
        amount=amount,
    )
    state.withdrawal_receipts.append(receipt)
```

### Predicates

#### `is_withdrawable_validator`

```python
def is_withdrawable_validator(validator: Validator, epoch: Epoch) -> bool:
    """
    Check if ``validator`` is withdrawable.
    """
    is_eth1_withdrawal_prefix = validator.withdrawal_credentials[0:1] == ETH1_ADDRESS_WITHDRAWAL_PREFIX
    return is_eth1_withdrawal_prefix and validator.withdrawable_epoch <= epoch < validator.withdrawn_epoch
```

## Beacon chain state transition function

### Epoch processing

```python
def process_epoch(state: BeaconState) -> None:
    process_justification_and_finalization(state)
    process_inactivity_updates(state)
    process_rewards_and_penalties(state)
    process_registry_updates(state)
    process_slashings(state)
    process_eth1_data_reset(state)
    process_effective_balance_updates(state)
    process_slashings_reset(state)
    process_randao_mixes_reset(state)
    process_historical_roots_update(state)
    process_participation_flag_updates(state)
    process_sync_committee_updates(state)
    process_full_withdrawals(state)  # [New in Capella]
```

#### Withdrawals

*Note*: The function `process_full_withdrawals` is new.

```python
def process_full_withdrawals(state: BeaconState) -> None:
    current_epoch = get_current_epoch(state)
    for index, validator in enumerate(state.validators):
        if is_withdrawable_validator(validator, current_epoch):
            # TODO, consider the zero-balance case
            withdraw(state, ValidatorIndex(index), state.balances[index])
            validator.withdrawn_epoch = current_epoch
```

### Block processing

```python
def process_block(state: BeaconState, block: BeaconBlock) -> None:
    process_block_header(state, block)
    if is_execution_enabled(state, block.body):
        process_withdrawal_transactions(state, block.body.execution_payload)  # [New in Capella]
        process_execution_payload(state, block.body.execution_payload, EXECUTION_ENGINE)  # [Modified in Capella]
    process_randao(state, block.body)
    process_eth1_data(state, block.body)
    process_operations(state, block.body)
    process_sync_aggregate(state, block.body.sync_aggregate)
```

#### New `process_withdrawal_transactions`

```python
def process_withdrawal_transactions(state: BeaconState, payload: ExecutionPayload) -> None:
    num_withdrawal_transactions = min(MAX_WITHDRAWAL_TRANSACTIONS_PER_PAYLOAD, len(state.withdrawal_receipts))
    dequeued_withdrawal_receipts = state.withdrawal_receipts[:num_withdrawal_transactions]

    assert len(dequeued_withdrawal_receipts) == len(payload.withdrawal_transactions)
    for dequeued_receipt, withdrawal_transaction in zip(dequeued_withdrawal_receipts, payload.withdrawal_transactions):
        assert dequeued_receipt == withdrawal_transaction

    # Ensure no withdrawal type transactions in the normal payload transactions
    # assert no_withdrawal_type_transactions_in(payload.transactions)

    # Remove dequeued receipts from state
    state.withdrawal_receipts = state.withdrawal_receipts[num_withdrawal_transactions:]
```

#### Modified `process_execution_payload`

*Note*: The function `process_execution_payload` is modified to use the new `ExecutionPayloadHeader` type.

```python
def process_execution_payload(state: BeaconState, payload: ExecutionPayload, execution_engine: ExecutionEngine) -> None:
    # Verify consistency of the parent hash with respect to the previous execution payload header
    if is_merge_transition_complete(state):
        assert payload.parent_hash == state.latest_execution_payload_header.block_hash
    # Verify prev_randao
    assert payload.prev_randao == get_randao_mix(state, get_current_epoch(state))
    # Verify timestamp
    assert payload.timestamp == compute_timestamp_at_slot(state, state.slot)
    # Verify the execution payload is valid
    assert execution_engine.notify_new_payload(payload)
    # Cache execution payload header
    state.latest_execution_payload_header = ExecutionPayloadHeader(
        parent_hash=payload.parent_hash,
        fee_recipient=payload.fee_recipient,
        state_root=payload.state_root,
        receipts_root=payload.receipts_root,
        logs_bloom=payload.logs_bloom,
        prev_randao=payload.prev_randao,
        block_number=payload.block_number,
        gas_limit=payload.gas_limit,
        gas_used=payload.gas_used,
        timestamp=payload.timestamp,
        extra_data=payload.extra_data,
        base_fee_per_gas=payload.base_fee_per_gas,
        block_hash=payload.block_hash,
        transactions_root=hash_tree_root(payload.transactions),
        withdrawal_transactions=hash_tree_root(payload.withdrawal_transactions),
    )
```