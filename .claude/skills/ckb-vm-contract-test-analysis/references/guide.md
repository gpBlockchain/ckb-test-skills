# CKB 合约测试补充参考

## 关键上游参考链接

| 资源 | URL | 说明 |
|------|-----|------|
| RFC 0022 — 交易结构 | https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0022-transaction-structure/0022-transaction-structure.md | CKB 完整交易字段、容量规则、Dep Group、Type ID |
| RFC 0017 — since 字段 | https://github.com/nervosnetwork/rfcs/blob/master/rfcs/0017-tx-valid-since/0017-tx-valid-since.md | 时间锁语义 |
| ckb-std (Rust) | https://github.com/nervosnetwork/ckb-std | `high_level.rs`、`ckb_constants.rs`、`syscalls/native.rs` |
| ckb-c-stdlib (C) | https://github.com/nervosnetwork/ckb-c-stdlib | `ckb_syscall_apis.h`、`ckb_consts.h` |
| ckb-script-templates | https://github.com/cryptape/ckb-script-templates | 工作区模板、`make build`、`make test`、Native Simulator |
| sUDT 测试示例 | https://github.com/sunchengzhu/ckb-contract-minitest/blob/main/tests/sudt.rs | 完整 sUDT 测试代码参考 |

---

## 分组逻辑详解（来自 RFC 0022）

CKB 按 `Hash(Script)` 将交易中的 Cell 分组执行。**Outputs 中的 Lock 脚本不执行**，仅 Type 脚本执行。

**示例交易**：

```solidity
Transaction {
    Inputs: [
        Cell_1 {lock: A, type: B},
        Cell_2 {lock: A, type: B},
        Cell_3 {lock: C, type: None}
    ]
    Outputs: [
        Cell_4 {lock: D, type: B},
        Cell_5 {lock: C, type: B},
        Cell_6 {lock: G, type: None},
        Cell_7 {lock: A, type: F}
    ]
}
```

**分组结果**：

```solidity
[
    Lock A: inputs:[0, 1], outputs:[],
    Lock C: inputs:[2],    outputs:[],
    Type B: inputs:[0, 1], outputs:[0, 1],
    Type F: inputs:[],     outputs:[3]
]
```

**执行过程**：CKB-VM 依次运行脚本 A、C、B、F，计算总 Cycle 消耗。Outputs 中的 Lock 脚本（D、C、G、A）不执行。

---

## sUDT 完整测试用例参考

### sUDT 合约核心代码（Rust）

```rust
use ckb_std::{
    entry,
    default_alloc,
    high_level::{load_script, load_cell_lock_hash, load_cell_data, QueryIter},
    ckb_constants::Source,
    error::SysError,
    ckb_types::{bytes::Bytes, prelude::*},
};

entry!(entry);
default_alloc!();

#[repr(i8)]
enum Error {
    IndexOutOfBound = 1,
    ItemMissing,
    LengthNotEnough,
    Encoding,
    Amount,
}

const UDT_LEN: usize = 16;

fn check_owner_mode(args: &Bytes) -> Result<bool, Error> {
    let is_owner_mode = QueryIter::new(load_cell_lock_hash, Source::Input)
        .find(|lock_hash| args[..] == lock_hash[..]).is_some();
    Ok(is_owner_mode)
}

fn collect_inputs_amount() -> Result<u128, Error> {
    let mut buf = [0u8; UDT_LEN];
    let udt_list = QueryIter::new(load_cell_data, Source::GroupInput)
        .map(|data| {
            if data.len() == UDT_LEN {
                buf.copy_from_slice(&data);
                Ok(u128::from_le_bytes(buf))
            } else {
                Err(Error::Encoding)
            }
        }).collect::<Result<Vec<_>, Error>>()?;
    Ok(udt_list.into_iter().sum::<u128>())
}

fn collect_outputs_amount() -> Result<u128, Error> {
    let mut buf = [0u8; UDT_LEN];
    let udt_list = QueryIter::new(load_cell_data, Source::GroupOutput)
        .map(|data| {
            if data.len() == UDT_LEN {
                buf.copy_from_slice(&data);
                Ok(u128::from_le_bytes(buf))
            } else {
                Err(Error::Encoding)
            }
        }).collect::<Result<Vec<_>, Error>>()?;
    Ok(udt_list.into_iter().sum::<u128>())
}

fn main() -> Result<(), Error> {
    let script = load_script()?;
    let args: Bytes = script.args().unpack();

    if check_owner_mode(&args)? {
        return Ok(());
    }

    let inputs_amount = collect_inputs_amount()?;
    let outputs_amount = collect_outputs_amount()?;

    if inputs_amount < outputs_amount {
        return Err(Error::Amount);
    }

    Ok(())
}
```

### API 分析

```rust
// args
let script = load_script()?;
let args: Bytes = script.args().unpack();

// data
QueryIter::new(load_cell_data, Source::GroupInput)
QueryIter::new(load_cell_data, Source::GroupOutput)
```

### SudtCell 抽象

```solidity
type SudtCell {
    args: { owner: bytes32 }
    data: { balance: u128 }
}
```

### 测试用例矩阵

| Inputs | Outputs | Scenario | Description |
| --- | --- | --- | --- |
| `[]` | `[SudtCell]` | 管理员铸造单个 sUDT | 1. 管理员可以创建一个 sUDT<br>2. 非管理员创建 sUDT，预期报错<br>3. 管理员可以创建 data 不为 16 字节的 cell |
| `[]` | `[SudtCell..N]` | 管理员批量铸造 | 1. 管理员批量创建 sUDT<br>2. 管理员可以创建多个 cell 总量超过 u128::MAX |
| `[SudtCell]` | `[]` | 销毁单个 sUDT | 1. 销毁一个 sUDT |
| `[SudtCell]` | `[SudtCell]` | 单转单（转账） | 1. 非管理员转移，input.balance >= output.balance<br>2. 非管理员转移，input.balance < output.balance，预期报错<br>3. 非管理员无法转移 data 不为 16 字节的 cell |
| `[SudtCell]` | `[SudtCell..N]` | 单转多（拆分） | 1. 非管理员转移，input.balance 总和 >= output.balance 总和 |
| `[SudtCell..N]` | `[]` | 批量销毁 | 1. 批量销毁 sUDT<br>2. 无法批量销毁 input.balance 总和 > u128::MAX |
| `[SudtCell..N]` | `[SudtCell]` | 多转单（合并） | 1. 非管理员转移 N 个 sUDT，input 总和 > output.balance<br>2. 非管理员转移，input 总和 < output.balance，预期报错 |
| `[SudtCell..N]` | `[SudtCell..N]` | 多转多（复杂转账） | 1. 非管理员转移 N 个 sUDT，input 总和 > output 总和<br>2. 非管理员转移，input 总和 < output 总和，预期报错 |

### sUDT 错误码

| Code | Value | Cause | How to Trigger |
|------|-------|-------|----------------|
| `Error::IndexOutOfBound` | 1 | 访问超出索引范围 | 越界读取 cell |
| `Error::ItemMissing` | 2 | 可选字段缺失 | 读取无 type script 的 cell 的 type hash |
| `Error::LengthNotEnough` | 3 | 缓冲区过小 | 提供小于实际数据的缓冲区 |
| `Error::Encoding` | 4 | data 长度不为 16 字节 | 构造 data 非 16 字节的 sUDT cell |
| `Error::Amount` | 5 | inputs 总量 < outputs 总量 | 构造 output.balance 超出 input.balance 的交易 |
