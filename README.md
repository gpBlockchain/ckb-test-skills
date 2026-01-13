# CKB Test Skills

此项目包含用于 CKB (Common Knowledge Base) 合约测试的辅助 Skills。旨在帮助开发者更高效地进行合约分析与测试用例设计。

## 依赖安装

使用此项目中的 Skills 需要先安装 `openskills` 工具：

```bash
npm i -g openskills
openskills install gpBlockchain/ckb-test-skills
openskills sync 
```

## 可用 Skills

### ckb-vm-contract-test-analysis

该 Skill 专注于分析 CKB 智能合约代码，辅助设计全面的测试方案。

**主要功能：**
*   **合约逻辑分析**: 深入理解合约的验证流程（如 Lock Script 和 Type Script）。
*   **测试场景生成**: 根据代码逻辑自动建议测试场景（正常流程与异常边界）。
*   **数据结构提取**: 识别合约使用的 Args、Witness 和 Cell Data 结构。

#### 使用方法

你可以直接调用此 Skill 对特定合约文件进行分析。

**命令示例：**

```text
使用 ckb-vm-contract-test-analysis 对 type 合约做测试分析 <合约文件路径>
```

#### 实战案例：xUDT 合约分析

以项目中的 `examples/xudt.c` 为例：

```text
使用 ckb-vm-contract-test-analysis 对 type 合约做测试分析 examples/xudt.c

```
*   [xudt OpenSkills 分享链接](https://opncd.ai/share/90bH1Us3)

以项目中的 `examples/commitment-lock.rs ` 为例：

```text
commitment-lock.rs 使用ckb-vm-contract-test-analysis 做合约测试分析

```
*   [commitment-lock OpenSkills 分享链接](https://opncd.ai/share/DxsUidbn)


