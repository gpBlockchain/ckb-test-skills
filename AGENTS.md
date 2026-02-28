

<skills_system priority="1">

## Available Skills

<!-- SKILLS_TABLE_START -->
<usage>
When users ask you to perform tasks, check if any of the available skills below can help complete the task more effectively. Skills provide specialized capabilities and domain knowledge.

How to use skills:
- Invoke: Bash("openskills read <skill-name>")
- The skill content will load with detailed instructions on how to complete the task
- Base directory provided in output for resolving bundled resources (references/, scripts/, assets/)

Usage notes:
- Only use skills listed in <available_skills> below
- Do not invoke a skill that is already loaded in your context
- Each skill invocation is stateless
</usage>

<available_skills>

<skill>
<name>ckb-vm-contract-test-analysis</name>
<description>Analyzes and designs test cases for CKB (Common Knowledge Base) smart contracts. Use this skill when working on CKB contract testing to understand transaction structures, verification rules, and test coverage requirements for Lock and Type scripts.</description>
<location>project</location>
</skill>

<skill>
<name>security-audit</name>
<description>AI-driven security audit skill for any software project. Use this skill when performing security audits to systematically analyze code for vulnerabilities across multiple dimensions including input validation, cryptography, authentication, business logic, memory safety, and more. Supports phased execution with TODO document tracking.</description>
<location>project</location>
</skill>

</available_skills>
<!-- SKILLS_TABLE_END -->

</skills_system>
