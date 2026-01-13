# CKB Test Skills

This project contains auxiliary Skills for CKB (Common Knowledge Base) contract testing. It aims to help developers perform contract analysis and test case design more efficiently.

## Installation

To use the Skills in this project, you need to install the `openskills` tool first:

```bash
npm i -g openskills
openskills install gpBlockchain/ckb-test-skills
openskills sync
```

## Available Skills

### ckb-vm-contract-test-analysis

This Skill focuses on analyzing CKB smart contract code to assist in designing comprehensive test plans.

**Key Features:**
*   **Contract Logic Analysis**: Deeply understand the contract verification flow (e.g., Lock Script and Type Script).
*   **Test Scenario Generation**: Automatically suggest test scenarios (normal flows and edge cases) based on code logic.
*   **Data Structure Extraction**: Identify Args, Witness, and Cell Data structures used by the contract.

#### Usage

You can directly invoke this Skill to analyze specific contract files.

**Command Example:**

```text
Use ckb-vm-contract-test-analysis to perform test analysis on type contract <contract_file_path>
```

#### Practical Example: xUDT Contract Analysis

Taking `examples/xudt.c` in the project as an example:

```text
Use ckb-vm-contract-test-analysis to perform test analysis on type contract examples/xudt.c
```
*   [xudt OpenSkills Share Link](https://opncd.ai/share/90bH1Us3)

#### Practical Example: commitment-lock Contract Analysis

Taking `examples/commitment-lock.rs` in the project as an example:

```text
Use ckb-vm-contract-test-analysis to perform test analysis on commitment-lock.rs
```
*   [commitment-lock OpenSkills Share Link](https://opncd.ai/share/DxsUidbn)
