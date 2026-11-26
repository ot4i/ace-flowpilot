---
name: ace-error-handling
description: Use when the user wants to add, fix or review error handling in IBM App Connect Enterprise message flows, ESQL or JavaCompute code, including Try Catch design, MQ backout and poison messages, HTTP error replies and exception logging.
---

# ACE Error Handling Skill

## Purpose
- Use this skill when the user asks to add, fix or review error handling, exception handling, retries, backout, poison message handling or error logging in ACE artifacts.
- Also apply it when a new flow receives messages from MQ or HTTP, calls a database or calls a backend. In that case apply only the minimum strategy: one catch handler for each input node, plus MQ backout settings or an HTTP error reply. Do not add Try Catch nodes, Throw nodes or extra `failure` wiring unless the user asks for them.

## When not to use this skill
- If the request is only about message transformation and the user explicitly asks for no error handling, use [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md) or [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) alone.

## Required reading order
1. [`skills/shared/skill-composition.md`](../shared/skill-composition.md)
2. [`skills/shared/error-handling-rules.md`](../shared/error-handling-rules.md)
3. [`skills/shared/node-types.md`](../shared/node-types.md)
4. [`skills/shared/review-checklist.md`](../shared/review-checklist.md)

## Critical rules
- [`skills/shared/error-handling-rules.md`](../shared/error-handling-rules.md) is the single source of runtime facts. Do not contradict it, especially that a wired `catch` flow commits the transaction unless it throws.
- Every catch flow must either handle the error (log, deliver, commit) or rethrow it (log, throw, roll back). State which one in the response.
- Never generate empty ESQL handlers or Java `catch` blocks that swallow exceptions.
- For MQ input flows, ask for or state the `BOTHRESH` and `BOQNAME` values of the input queue.
- For HTTP input flows, every catch path must end in an HTTP Reply node with an explicit status code and must not expose internal error details.
- User message numbers are 2951 to 2999 for ESQL `THROW` and 3001 to 3049 for the Throw node. Do not invent message catalogs, other message numbers, node properties or terminal names.
- Use [`skills/ace-message-flow/SKILL.md`](../ace-message-flow/SKILL.md) or [`skills/ace-subflow/SKILL.md`](../ace-subflow/SKILL.md) for the `.msgflow` or `.subflow` structure, and [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md) or [`skills/ace-java-compute/SKILL.md`](../ace-java-compute/SKILL.md) for the code.

## Output requirements
- Create or update the requested artifacts with the error handling applied.
- Summarize the error strategy in four points:
  1. Where errors are caught: `catch` or `failure` terminals, Try Catch nodes, ESQL handlers.
  2. The transaction outcome of each catch flow: handle and commit, or rethrow and roll back.
  3. Where failed messages go: HTTP status code, error queue, backout queue or dead-letter queue.
  4. What is logged: the fields from the logging rules in [`skills/shared/error-handling-rules.md`](../shared/error-handling-rules.md).
- State every assumption the user must configure, such as `BOTHRESH`, `BOQNAME` and transaction modes.
