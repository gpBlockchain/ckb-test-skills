---
name: ckb-vm-contract-test-analysis
description: Analyzes and designs test cases for CKB (Common Knowledge Base) smart contracts. Use this skill when working on CKB contract testing to understand transaction structures, verification rules, and test coverage requirements for Lock and Type scripts.
---
# CKB Contract Test Analysis

This skill provides guidelines for testing CKB (Common Knowledge Base) smart contracts. CKB uses a UTXO model, differing from account-based models like Ethereum. Testing focuses on transaction structure, grouping logic, and comprehensive test case design for Lock and Type scripts.

## 0. Upstream Knowledge Sources — Agent MUST Fetch

Before analyzing or designing tests, fetch the latest information from these authoritative sources:

### §0.1 RFC 0022 — CKB Transaction Structure
- **URL**: https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md
- **Extract**: Full transaction fields, capacity rules, Dep Group semantics, Type ID pattern, header_deps preconditions, `hash_type` (data vs type), `outputs_data` as parallel array.

### §0.2 ckb-std — Rust Contract Library
- **Repo**: https://github.com/nervosnetwork/ckb-std
- **Key files**:
  - `src/high_level.rs` — all high-level Rust API functions and `QueryIter` pattern
  - `src/ckb_constants.rs` — `Source`, `CellField`, `HeaderField`, `InputField` enums and syscall numbers
  - `src/syscalls/native.rs` — raw syscall wrappers
- **Extract**: Complete function signatures, iterator patterns, error handling idioms.

### §0.3 ckb-c-stdlib — C Contract Library
- **Repo**: https://github.com/nervosnetwork/ckb-c-stdlib
- **Key files**:
  - `ckb_syscall_apis.h` — all C syscall signatures including VM v2 (spawn/pipe/read/write)
  - `ckb_consts.h` — all source, field, and error code constants
- **Extract**: Complete C API signatures, constant values, VM version groupings.

### §0.4 ckb-script-templates — Contract Project Templates
- **Repo**: https://github.com/cryptape/ckb-script-templates
- **Extract**: Standard workspace structure, template types, `make build`/`make test` flow, Native Simulator support.

## 1. CKB Transaction Structure

A CKB transaction includes:

- **Inputs**: Cells consumed in the transaction.
  - `previous_output`: Reference to the prior output Cell.
  - `since`: Time-lock field for transaction validity (see [RFC 0017](https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0017-tx-valid-since/0017-tx-valid-since.md)).
- **Outputs**: New Cells created by the transaction.
- **outputs_data**: Parallel array to `outputs`; each element is the data for the corresponding output Cell.
- **CellDeps**: Dependencies for validation code or other Cells.
  - Can be `DepType::Code` (single cell) or `DepType::DepGroup` (expands to multiple cells).
- **HeaderDeps**: Block header hashes referenced by the transaction; must be within a recent epoch window.
- **Witnesses**: Additional data (e.g., signatures) associated with inputs.

### 1.1 Cell Structure

Each Cell comprises:

- **Lock Script**: Ownership verification logic.
- **Type Script** (Optional): Data validation logic.
- **Data**: Stored content (referenced in parallel via `outputs_data`).
- **Capacity**: Storage space allocated (in Shannons; 1 CKB = 10^8 Shannons).

### 1.2 Script Structure

Scripts contain:
- `code_hash`: Hash of the executable code.
- `hash_type`:
  - `data` — `code_hash` is the hash of the cell's data field (exact binary match).
  - `type` — `code_hash` is the hash of the cell's type script (enables upgradable scripts).
- `args`: Parameters passed to the script at execution time.

### 1.3 Capacity Rules

- A cell must satisfy: `occupied(cell) ≤ cell.capacity` (cell's used bytes ≤ allocated capacity).
- A transaction must satisfy: `sum(inputs.capacity) ≥ sum(outputs.capacity)` (no capacity creation).

### 1.4 Dep Group

A `DepGroup` cell's data is a list of `OutPoint`s. When used as a cell dep, CKB expands it transparently into all referenced cells. This allows bundling multiple code cells under one dep reference.

### 1.5 Type ID

A Type ID script enables singleton cells (unique by construction):
- **Creation**: `inputs: [], outputs: [TypeIDCell]` — type_args = hash(first_input_outpoint ++ output_index).
- **Transfer**: `inputs: [TypeIDCell], outputs: [TypeIDCell]` — exactly one input and one output in the group.
- **Deletion**: `inputs: [TypeIDCell], outputs: []` — type ID destroyed.

### 1.6 Header Deps

`header_deps` allows contracts to read block header data (epoch, timestamp, etc.).
- Headers must be confirmed and within a recent epoch window at transaction submission time.
- Two loading methods: by index into `header_deps` array, or by matching `input.since` (time-locked inputs).

## 2. Verification Rules

### 2.1 Grouping Logic

Cells are grouped by `Hash(Script)` for execution:
- **Inputs**: Both Lock and Type scripts are grouped and executed.
- **Outputs**: Only Type scripts are executed; Lock scripts are not.

### 2.2 Source Semantics

| Source | Value | Meaning |
|--------|-------|---------|
| `Input` | 1 | All transaction inputs (absolute index) |
| `Output` | 2 | All transaction outputs (absolute index) |
| `CellDep` | 3 | Cell dependencies (absolute index) |
| `HeaderDep` | 4 | Header dependencies (absolute index) |
| `GroupInput` | 0x0100000000000001 | Inputs sharing the current script hash (relative index) |
| `GroupOutput` | 0x0100000000000002 | Outputs sharing the current script hash (relative index) |

### 2.3 Execution Flow

- Scripts run in the CKB-VM by group.
- Cycles (execution costs) accumulate per group, with a limit of ~1,000M cycles.
- Execution order: Lock scripts on inputs, then Type scripts on inputs, then Type scripts on outputs.
- A script returning non-zero causes the entire transaction to fail.

## 3. Contract API Reference

### §3.1 C Syscall Signatures

```c
/* VM version 0 — Core APIs */
int ckb_exit(int8_t code);
int ckb_load_tx_hash(void* addr, uint64_t* len, size_t offset);
int ckb_load_transaction(void* addr, uint64_t* len, size_t offset);
int ckb_load_script_hash(void* addr, uint64_t* len, size_t offset);
int ckb_load_script(void* addr, uint64_t* len, size_t offset);
int ckb_load_cell(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);
int ckb_load_input(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);
int ckb_load_header(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);
int ckb_load_witness(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);
int ckb_load_cell_by_field(void* addr, uint64_t* len, size_t offset, size_t index, size_t source, size_t field);
int ckb_load_header_by_field(void* addr, uint64_t* len, size_t offset, size_t index, size_t source, size_t field);
int ckb_load_input_by_field(void* addr, uint64_t* len, size_t offset, size_t index, size_t source, size_t field);
int ckb_load_cell_data(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);

/* VM version 1 */
int ckb_vm_version();
uint64_t ckb_current_cycles();
int ckb_exec(size_t index, size_t source, size_t place, size_t bounds, int argc, const char* argv[]);

/* VM version 2 — Spawn/IPC APIs */
typedef struct spawn_args_t {
  size_t argc;
  const char** argv;
  uint64_t* process_id;
  const uint64_t* inherited_fds;  /* 0-terminated list */
} spawn_args_t;

int ckb_spawn(size_t index, size_t source, size_t place, size_t bounds, spawn_args_t* spawn_args);
int ckb_wait(uint64_t pid, int8_t* exit_code);
uint64_t ckb_process_id();
int ckb_pipe(uint64_t fds[2]);
int ckb_read(uint64_t fd, void* buffer, size_t* length);
int ckb_write(uint64_t fd, const void* buffer, size_t* length);
int ckb_inherited_fds(uint64_t* fds, size_t* length);
int ckb_close(uint64_t fd);
int ckb_load_block_extension(void* addr, uint64_t* len, size_t offset, size_t index, size_t source);
```

### §3.2 Field Constants

```c
/* Source */
#define CKB_SOURCE_INPUT                 1
#define CKB_SOURCE_OUTPUT                2
#define CKB_SOURCE_CELL_DEP              3
#define CKB_SOURCE_HEADER_DEP            4
#define CKB_SOURCE_GROUP_INPUT           0x0100000000000001
#define CKB_SOURCE_GROUP_OUTPUT          0x0100000000000002

/* CellField */
#define CKB_CELL_FIELD_CAPACITY          0
#define CKB_CELL_FIELD_DATA_HASH         1
#define CKB_CELL_FIELD_LOCK              2
#define CKB_CELL_FIELD_LOCK_HASH         3
#define CKB_CELL_FIELD_TYPE              4
#define CKB_CELL_FIELD_TYPE_HASH         5
#define CKB_CELL_FIELD_OCCUPIED_CAPACITY 6

/* HeaderField */
#define CKB_HEADER_FIELD_EPOCH_NUMBER              0
#define CKB_HEADER_FIELD_EPOCH_START_BLOCK_NUMBER  1
#define CKB_HEADER_FIELD_EPOCH_LENGTH              2

/* InputField */
#define CKB_INPUT_FIELD_OUT_POINT        0
#define CKB_INPUT_FIELD_SINCE            1
```

### §3.3 Rust High-Level API (ckb-std)

```rust
// Transaction-level
load_tx_hash() -> Result<[u8; 32], SysError>
load_script_hash() -> Result<[u8; 32], SysError>
load_script() -> Result<Script, SysError>
load_transaction() -> Result<Transaction, SysError>

// Cell
load_cell(index, source) -> Result<CellOutput, SysError>
load_cell_capacity(index, source) -> Result<u64, SysError>
load_cell_occupied_capacity(index, source) -> Result<u64, SysError>
load_cell_data(index, source) -> Result<Vec<u8>, SysError>
load_cell_data_hash(index, source) -> Result<[u8; 32], SysError>
load_cell_lock(index, source) -> Result<Script, SysError>
load_cell_lock_hash(index, source) -> Result<[u8; 32], SysError>
load_cell_type(index, source) -> Result<Option<Script>, SysError>
load_cell_type_hash(index, source) -> Result<Option<[u8; 32]>, SysError>

// Input
load_input(index, source) -> Result<CellInput, SysError>
load_input_since(index, source) -> Result<u64, SysError>
load_input_out_point(index, source) -> Result<OutPoint, SysError>

// Witness
load_witness(index, source) -> Result<Vec<u8>, SysError>
load_witness_args(index, source) -> Result<WitnessArgs, SysError>

// Header
load_header(index, source) -> Result<Header, SysError>
load_header_epoch_number(index, source) -> Result<u64, SysError>
load_header_epoch_start_block_number(index, source) -> Result<u64, SysError>
load_header_epoch_length(index, source) -> Result<u64, SysError>

// Utilities
find_cell_by_data_hash(data_hash: &[u8], source) -> Result<Option<usize>, SysError>
exec_cell(code_hash, hash_type, argv) -> Result<Infallible, SysError>
spawn_cell(code_hash, hash_type, argv, inherited_fds) -> Result<u64, SysError>
inherited_fds() -> Vec<u64>

// Iterator
QueryIter::new(load_fn, source)   // iterates until IndexOutOfBound; panics on other errors
```

### §3.4 Source Enum (Rust)

```rust
pub enum Source {
    Input          = 1,
    Output         = 2,
    CellDep        = 3,
    HeaderDep      = 4,
    GroupInput     = 0x0100000000000001,
    GroupOutput    = 0x0100000000000002,
}
```

### §3.5 Error Codes

| Code | Name | Value | Meaning |
|------|------|-------|---------|
| `CKB_SUCCESS` | Success | 0 | Operation succeeded |
| `CKB_INDEX_OUT_OF_BOUND` | IndexOutOfBound | 1 | No more items at given index |
| `CKB_ITEM_MISSING` | ItemMissing | 2 | Optional field absent (e.g., no type script) |
| `CKB_LENGTH_NOT_ENOUGH` | LengthNotEnough | 3 | Buffer too small; retry with actual size |
| `CKB_INVALID_DATA` | InvalidData | 4 | Data format invalid |
| `CKB_WAIT_FAILURE` | WaitFailure | 5 | `ckb_wait` failed (VM v2) |
| `CKB_INVALID_FD` | InvalidFd | 6 | Invalid file descriptor (VM v2) |
| `CKB_OTHER_END_CLOSED` | OtherEndClosed | 7 | Pipe other end closed (VM v2) |
| `CKB_MAX_VMS_SPAWNED` | MaxVmsSpawned | 8 | Too many spawned VMs (VM v2) |
| `CKB_MAX_FDS_CREATED` | MaxFdsCreated | 9 | Too many file descriptors (VM v2) |

## 4. Contract API Analysis Pattern

Analyze contracts by identifying which syscalls they invoke and what data they access. Treat transaction metadata as contract parameters.

### §4.1 Analysis Forms

- **Script args–driven**: `load_script` → `args()` used for ownership or parameter checks.
- **Cell data–driven**: `load_cell_data` validates structure, sums, ranges, and invariants.
- **Witness–driven**: `load_witness` / `load_witness_args` carries signatures/proofs for authorization and multi-sig.
- **CellDeps–driven**: read config/oracle/code via `Source::CellDep` for external parameters.
- **Header–dependent**: use `load_header` / `load_header_by_field` for time/epoch/environmental constraints.
- **Transaction–wide**: access `load_transaction` / `load_tx_hash` for global or anti-replay constraints.
- **Field-level access**: `load_*_by_field` for precise field validation without full object parsing.
- **Group aggregation**: iterate `GroupInput` / `GroupOutput` via `QueryIter` to compute totals and cross-cell consistency.
- **Index/source driven**: select indices across Input/Output/CellDep sources to target specific cells.
- **Since-driven**: read `load_input_since` to enforce time-lock or relative epoch constraints.
- **Spawn/Exec-driven**: use `spawn_cell` / `exec_cell` for multi-process contract composition (VM v2).
- **Mixed strategy**: combine the above patterns to build robust validation pipelines.

### §4.2 Cell Structure Abstraction Template

Define input/output Cell structures as:

```typescript
type MyCellArgs {
    args: { field1: TypeA, field2: TypeB },
    data: { field3: TypeC },
    witness: { field4: TypeD },   // optional
}
```

## 5. Test Case Design

Design tests based on input/output combinations to cover validation, transformation, and edge cases.

### Expected Output Format

All test cases must be presented as a Markdown table with the following headers:

| Inputs | Outputs | Scenario | Description |
| --- | --- | --- | --- |

### 5.1 Lock Script Testing

Lock scripts execute only on inputs (outputs are not executed). Key scenarios:

| Inputs | Outputs | Scenario | Description |
|--------|---------|----------|-------------|
| Single | None | `inputs: [N], outputs: []` | Single-cell validation (e.g., burn). |
| Multi  | None | `inputs: [N..N], outputs: []` | Multi-cell validation with same lock. |

### 5.2 Type Script Testing

Type scripts execute on both inputs and outputs. Cover creation, destruction, and transformation:

| Inputs | Outputs | Scenario | Description |
|--------|---------|----------|-------------|
| Single | None   | `inputs: [N], outputs: []` | Input validation (burn). |
| Multi  | None   | `inputs: [N..N], outputs: []` | Multi-input validation. |
| Single | Single | `inputs: [N], outputs: [N]` | 1-to-1 transformation. |
| Single | Multi  | `inputs: [N], outputs: [N..N]` | Split (1-to-N). |
| Multi  | Single | `inputs: [N..N], outputs: [N]` | Merge (N-to-1). |
| Multi  | Multi  | `inputs: [N..N], outputs: [N..N]` | Complex N-to-N. |
| None   | Single | `inputs: [], outputs: [N]` | Creation (mint). |
| None   | Multi  | `inputs: [], outputs: [N..N]` | Batch creation. |

## 6. Testing Considerations

- **Grouping**: Validate correct grouping in multi-Cell scenarios.
- **Robustness**: Include invalid cases (e.g., non-zero return codes for failures).
- **Cycle Limits**: Monitor and optimize for the ~1,000M cycle cap.
- **Data Boundaries**: Test extremes for `args`, `witness`, and `data` sizes/values.
- **Edge Cases**: Cover zero-balance, max-capacity, and dependency failures.
- **Dep Group**: Test that contracts work when code cell is loaded via a DepGroup.
- **Type ID**: Test singleton creation, transfer (exactly 1-in/1-out), and deletion edge cases.
- **Since / Time-lock**: Test relative and absolute time-lock values via `load_input_since`.

## 7. Error Scenarios and Return Codes

Use negative test cases to cover each error path exposed by the contract. Design inputs/witnesses to deterministically trigger the failure and assert non-zero return codes.

| Code | Cause | How to Trigger | Expected Behavior | Notes |
|------|-------|----------------|-------------------|-------|
