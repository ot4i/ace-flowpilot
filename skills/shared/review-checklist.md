# ACE Skill Review Checklist

## Purpose
Use this checklist before returning generated ACE artifacts.

## When to use
Use this checklist after generating or updating ACE artifacts and before finalizing a response.

## Checklist

### Artifact correctness
- The correct artifact type was created.
- The generated files were placed in the correct project type.

### Project metadata — Application projects
When a new Application project is created, **all four of the following files must be present in the output** — not described in prose, actually created:
- `.project` — contains **exactly 2 natures**: `com.ibm.etools.msgbroker.tooling.applicationNature` and `com.ibm.etools.msgbroker.tooling.messageBrokerProjectNature`. Do not add extra natures.
- `.settings/org.eclipse.core.resources.prefs` — **must be created as a file and contain the following exact content verbatim:**
```properties
eclipse.preferences.version=1
encoding/<project>=UTF-8
```
- The `.msgflow` file(s) requested by the user.

### Project metadata — Policy projects
- Any new Policy project contains the required metadata files.

### Message flow nodes
- Every message flow node uses a validated `xmi:type`.
- HTTP Input and HTTP Reply nodes use `ComIbmWSInput.msgnode` and `ComIbmWSReply.msgnode` — **not** `ComIbmHTTPInput` or `ComIbmHTTPReply`.
- If any Compute node reads `InputRoot.XMLNSC.*`, confirm the upstream Input node has `messageDomainProperty` of `XMLNSC`. If it reads `InputRoot.JSON.Data`, confirm `messageDomainProperty` of `JSON`. A missing or mismatched messageDomainProperty causes silent empty-tree failures at runtime.

### Connector-specific
- Connector-specific rules were applied where relevant.
- Required schema files were created where relevant.
- Required policy files were created where relevant.

### ESQL correctness
- No direct cross-domain tree assignment (`SET OutputRoot.JSON.Data = InputRoot.XMLNSC.*`) — this produces duplicate keys for repeated elements.
- A repeatability table was built for every element at every nesting depth before writing code. Every element that can repeat is mapped to a `JSON.Array`, not a scalar.
- Every repeating element is iterated with `DECLARE ... REFERENCE TO [1]` + `WHILE LASTMOVE` + `MOVE ... NEXTSIBLING NAME`, with a `NAMESPACE` clause when the element is namespace-qualified. An existing `FOR ... AS ...[] DO` loop is valid and is not a defect.
- Every JSON array is declared with `CREATE FIELD ... IDENTITY (JSON.Array)` before being populated.
- Every JSON array object is created with `CREATE LASTCHILD ... TYPE Name` before its fields are assigned.
- Every scalar read from the XMLNSC tree uses `FIELDVALUE()`.
- Output domain is initialised with `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` + `CREATE FIELD OutputRoot.JSON.Data` — not `SET OutputRoot.JSON.Data = NULL`.
- `CREATE OUTPUTROOT DOMAIN(...)` does not appear anywhere — it is not valid ACE ESQL syntax.
- There is no such built-in ESQL function named `RANDOM` although there is a built-in function named `RAND`

### Error handling
- Every input node has a deliberate error strategy: its `catch` terminal is wired to an error handler, or the response states why the node default is acceptable.
- Every catch flow either handles the error (log, deliver, commit) or rethrows it (log, throw, roll back). No catch flow ends silently.
- No empty ESQL handler (`DECLARE ... HANDLER ... BEGIN END;`) and no Java `catch` block that swallows an exception.
- `THROW USER EXCEPTION` uses `SEVERITY 3` and a message number in the user range 2951 to 2999 unless the user provided a custom catalog. A Throw node uses a message number in the range 3001 to 3049.
- MQ input flows state the `BOTHRESH` and `BOQNAME` assumptions, or wire the MQ Input `failure` terminal, so a poison message cannot block the queue.
- MQ input flows that must not lose messages use `Transaction mode` `Yes`, not `Automatic` or `No`.
- A Compute node `Transaction` property is `Commit` only when the user asked for database work to survive a flow rollback.
- Every catch path in an HTTP flow ends in an HTTP Reply node with an explicit status code and does not expose internal error details. The status is set in `OutputLocalEnvironment.Destination.HTTP.ReplyStatusCode`, or in `HTTPReplyHeader` created before any body folder (`BIP6062W`).
- Every handled error is logged with flow name, failing node, error number, root cause text and a correlation id.

### Simplicity
- The flow topology is the minimum required to satisfy the request. Do not add Compute nodes, ESQL files, or transformation logic unless the user explicitly asked for transformation. A direct wire from Input to Reply is correct for a pass-through or echo pattern.

### Final response
- Version-specific assumptions were either confirmed or clearly stated.
- The final response includes a concise summary of what was created.
- If a new Application project was created, include the ACE Toolkit import guidance.

## Related files
- [`skills/shared/ace-projects.md`](ace-projects.md)
- [`skills/shared/policy-projects.md`](policy-projects.md)
- [`skills/shared/message-flow-rules.md`](message-flow-rules.md)
- [`skills/shared/error-handling-rules.md`](error-handling-rules.md)
