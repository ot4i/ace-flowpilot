# ACE Error Handling Rules

## Purpose
This document defines how to design and review error handling in IBM App Connect Enterprise (ACE) message flows, ESQL and JavaCompute code.

## When to use
Use this document whenever a request creates or reviews a message flow, subflow, Compute node or JavaCompute node that receives messages from MQ or HTTP, calls a database or backend, or explicitly asks for error handling, retries, logging or poison message handling.

## How errors move through a flow
Apply these runtime facts when designing or reviewing a flow. Do not contradict them.

- A node's `failure` terminal receives errors that occur **inside that node**. The message on the failure path is the message as it was before the node ran.
- If a node's `failure` terminal is not wired, the failure is turned into an exception and control returns to the **closest upstream node that can process it**: a Try Catch node, an AggregateReply node or the input node. It does not go to the `failure` terminal of an upstream node.
- A `catch` terminal only receives exceptions thrown by nodes **downstream** of it.
- If an input node's `catch` terminal is wired and the catch flow completes without throwing, the transaction is **not** rolled back. It is committed. Rollback happens only when the catch flow throws.
- If neither `failure` nor `catch` is wired on an input node, the node applies its own default processing, which differs by node type (see the MQ and HTTP sections).
- When an exception returns control to a catch flow, the Root (message) tree and LocalEnvironment are restored to what they were at that node. Environment changes are kept, so Environment is the place to carry error context (for example a business key) into a catch flow. See the Environment guidance in [`skills/shared/esql-guidelines.md`](esql-guidelines.md).
- On a Try Catch node `catch` path, the message tree is the one the Try Catch node originally received. Changes made in the try flow are not present.
- Database updates made in the try flow are committed or rolled back according to the node's transaction setting, not by the Try Catch node. If they were committed, the catch flow must undo them when that matters.
- A Compute node with `Transaction` set to `Commit` (the default is `Automatic`) commits its database work regardless of whether the flow succeeds, so a later rollback cannot undo it. Use `Automatic` unless the user asks otherwise.
- A subflow is inlined into its parent flow and runs in the parent flow's transaction.
- By default, MQ and database work in a flow are not coordinated (two-phase). If the user needs both committed or rolled back together, say that the flow's coordinated transaction setting and a local queue manager are required; do not claim this happens by default.

Source: [Handling errors in message flows](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=flows-handling-errors-in-message), [TryCatch node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-trycatch-node), [Compute node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-compute-node)

## Flow design rules
- Every input node must have a deliberate error strategy. Wire its `catch` terminal to an error handler, or state in the response why the node default is acceptable.
- Prefer **one consistent error handler** (a subflow reused by every flow in the application) over wiring every node's `failure` terminal individually.
- Use a Try Catch node around a sequence of nodes that needs local recovery, such as a backend call that should return a fallback reply instead of failing the whole flow.
- Wire a node's `failure` terminal only when that node needs specific handling.
- Input nodes use `Parse timing` `On Demand` by default, which delays parsing of the message body. A malformed body is then usually detected by the first node that reads it, not by the input node, and it reaches the nearest catch. If unparseable messages must be diverted at the input node, set `Parse timing` to `Complete` (or validate the message) and state this in the response.
- Every catch flow must end in exactly one of these outcomes, and the response must say which one:
  1. **Handle**: log the error, deliver it (error queue, error reply), and let the transaction commit.
  2. **Rethrow**: log the error, then throw (a Throw node, or ESQL `THROW`) so the transaction rolls back and the message is retried or backed out. `RESIGNAL` is valid only inside an ESQL handler, not as a general catch flow statement.
- Never let a catch flow end silently with no log, no reply and no error message. That loses the message.
- Keep the catch flow simple. An exception inside the catch flow causes a rollback, so it must not call the same backend that just failed.
- Use a Throw node (`ComIbmThrow.msgnode`) to raise a business error from flow logic without code. Its default message number is 3001, and 3001 to 3049 are reserved for user messages in the default catalog. Source: [Throw node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-throw-node)

## MQ input flows: retries, backout and poison messages
- The MQ Input node `Transaction mode` values are `Automatic`, `Yes` (default) and `No`. Keep `Yes` when a message must not be lost.
- With `Automatic`, only persistent messages are received under sync point. Non-persistent messages (and every message with `No`) are received outside sync point, so a rollback cannot return them to the queue and backout processing does not apply. Do not use `Automatic` or `No` for business-critical messages unless the user accepts that risk.
- Retries are controlled by the input queue attributes `BOTHRESH` (backout threshold) and `BOQNAME` (backout queue). The MQ Input node has no retry count property. Ask the user for both values, or state the assumption.
- When a transactional flow rolls back, the message returns to the input queue and is retried until it succeeds or the backout threshold is reached. MQ tracks the number of backouts in `MQMD.BackoutCount`, and increments it only for a message that was received under sync point and then rolled back. When the threshold is reached:
  1. If the MQ Input node `failure` terminal is wired, the message is propagated there.
  2. Otherwise it is put to the `BOQNAME` queue, if defined.
  3. Otherwise it is put to the queue manager dead-letter queue, if defined.
  4. Otherwise the message keeps looping through the node until the problem is resolved. This is a **poison message** that blocks the queue.
- If the backout threshold is reached a second time in the fail flow, the node tries the backout queue and then the dead-letter queue. Handle the error in the fail flow; do not rely on this fallback.
- A `BOTHRESH` of 0 behaves as 1, which means no retry. Use 2 for one retry.
- Always define `BOQNAME` (or wire the MQ Input `failure` terminal to an error queue) so a poison message cannot block the queue.
- Separate retryable errors from non-retryable ones. Retry transient errors, such as a database or backend that is unavailable, by rethrowing so MQ backout applies. Handle non-retryable errors, such as a validation or parse failure, the first time: log them, put the message to the error queue and let the transaction commit, because a retry cannot fix them. When the catch flow cannot tell which kind it has, rethrow.
- To keep an error record even when the flow rolls back, set the error queue MQ Output node `Transaction mode` to `No`. A `No` put is committed immediately. Verify this on a test queue, because it changes delivery guarantees.

Source: [MQInput node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-mqinput-node), [MQOutput node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-mqoutput-node), [Handling MQInput node errors](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=flows-handling-mqinput-node-errors)

## HTTP input flows
- The HTTP Input `failure` terminal receives messages that fail validation. The `catch` terminal receives exceptions thrown downstream. The `HTTP Timeout` terminal is used only when the integration server uses the embedded HTTP listener; the nodes wired to it must include an HTTP Reply node.
- If a downstream exception is not caught and returns to the HTTP Input node, the node returns an error reply built from the exception in the format set by its `Fault format` property (SOAP 1.1 by default; SOAP 1.2, HTML and JSON are also available).
- IBM's terminal description also says that if the `failure` or `catch` terminal is not wired, the message is discarded and the client receives an error when the `Maximum client wait time` expires (180 seconds by default). Whenever no reply is sent within that time, the client receives a timeout fault.
- Therefore every catch path in an HTTP flow must end in an HTTP Reply node (`ComIbmWSReply.msgnode`) with an explicit status code and a JSON error body.
- Set the status code in `OutputLocalEnvironment.Destination.HTTP.ReplyStatusCode`, for example 500 for a server error or 400 for a validation error. It overrides any status code in the reply headers, and the Compute node `Compute mode` must include LocalEnvironment.
- If you set `X-Original-HTTP-Status-Code` in `HTTPReplyHeader` instead, create that header before any body folder, as defined in [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md). A body folder created first causes `BIP6062W`.
- Do not return ExceptionList text, stack details or internal host names to the client. Return a short message and a correlation id, and log the details.

Source: [HTTPInput node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-httpinput-node), [HTTPReply node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-httpreply-node)

## ESQL: raising errors
- Use `THROW USER EXCEPTION` for business and validation errors. The clause order is `SEVERITY`, `CATALOG`, `MESSAGE`, `VALUES`.
- Include `SEVERITY 3`. The default is 1, and IBM requires 3 on Windows so the event log reports the error correctly.
- Use a `MESSAGE` number from 2951 to 2999, the range reserved for users in the default catalog. Omit `CATALOG` to use the current version catalog (`BIPmsgs`).
- `VALUES` supplies the message inserts. The supplied messages cater for eight inserts. Use short, readable values, because they appear in logs and in the ExceptionList.
- In the ExceptionList, find your values by content, not by position: do not assume your first `VALUES` entry is `Insert[1]`.
- Do not invent other catalogs or message numbers outside the user range unless the user provides a custom catalog.

```esql
IF COALESCE(FIELDVALUE(InputRoot.JSON.Data.orderId), '') = '' THEN
    THROW USER EXCEPTION SEVERITY 3 MESSAGE 2951 VALUES('Validation failed', 'orderId is missing');
END IF;
```

Source: [THROW statement](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=statements-throw-statement), [Creating message catalogs](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=extension-creating-message-catalogs)

## ESQL: handlers and database errors
- `DECLARE {CONTINUE | EXIT} HANDLER FOR SQLSTATE ...` declarations must be together at the top of their scope, before any other statements. `EXIT` resumes after the scope; `CONTINUE` resumes after the failing statement.
- A handler **absorbs** the exception, so the transaction commits. A handler must therefore log the error and then either recover deliberately or rethrow with `RESIGNAL;` (valid only inside a handler) or `THROW`.
- **Never generate an empty handler** such as `DECLARE CONTINUE HANDLER FOR SQLSTATE LIKE '%' BEGIN END;`. It silently swallows every error.
- `SQLSTATE` starts with `D` for database errors. Integration node errors use fixed values such as `S22012` (divide by zero). For user exceptions the state comes from the `THROW` inserts, so test it before routing a handler on it. Handlers are searched in declaration order, so declare specific handlers first and any `LIKE '%'` handler last.
- Keep the Compute node property `Throw exception on database error` selected (the default). If the user turns it off, the ESQL must check `SQLCODE` and `SQLSTATE` after every database statement and `THROW` on failure; otherwise database errors are ignored.

```esql
CREATE FUNCTION Main() RETURNS BOOLEAN
BEGIN
    DECLARE EXIT HANDLER FOR SQLSTATE LIKE 'D%'
    BEGIN
        -- Add context, then rethrow so the flow's catch strategy applies
        THROW USER EXCEPTION SEVERITY 3 MESSAGE 2952
            VALUES('Database insert failed', SQLSTATE, SQLCODE, SQLERRORTEXT);
    END;

    INSERT INTO Database.DATASOURCE_NAME.SCHEMA.ORDERS (ORDER_ID) VALUES (FIELDVALUE(InputRoot.JSON.Data.orderId));
    CALL CopyEntireMessage();
    RETURN TRUE;
END;
```

Source: [DECLARE HANDLER statement](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=statements-declare-handler-statement), [RESIGNAL statement](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=statements-resignal-statement), [SQLSTATE function](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=functions-sqlstate-function), [Compute node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-compute-node)

## ESQL: reading the ExceptionList in a catch flow
- In a Compute node, read the exception tree through `InputExceptionList`.
- Each exception (for example `RecoverableException`, `ParserException`, `DatabaseException`, `UserException`) can contain `Number`, `Severity`, `Catalog`, `Label` (the node that raised it), an optional `Text` and zero or more `Insert` elements (`Type`, `Text`). A nested exception, if present, is the last child of its parent.
- The **innermost** exception is closest to the root cause. Walk down with `MOVE ... LASTCHILD` until the move fails, and keep the values of the last element that has a `Number`.
- Make routing decisions on `Number` (and `Catalog`). IBM states that `File`, `Line`, `Function` and `Text` must not be used for decision making; log them only.

```esql
DECLARE ex REFERENCE TO InputExceptionList.*[1];
DECLARE errNumber INTEGER 0;
DECLARE errText CHARACTER '';
DECLARE errNode CHARACTER '';
WHILE LASTMOVE(ex) DO
    IF FIELDVALUE(ex.Number) IS NOT NULL THEN
        SET errNumber = FIELDVALUE(ex.Number);
        SET errNode = COALESCE(FIELDVALUE(ex.Label), errNode);
        SET errText = COALESCE(FIELDVALUE(ex.Text), '');
        FOR ins AS ex.Insert[] DO
            SET errText = errText || ' | ' || COALESCE(FIELDVALUE(ins.Text), '');
        END FOR;
    END IF;
    MOVE ex LASTCHILD;
END WHILE;
```

Source: [Exception list tree](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=assembly-exception-list-tree), [Accessing the ExceptionList tree using ESQL](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=tree-accessing-exceptionlist-using-esql)

## JavaCompute exceptions
- `evaluate()` throws `MbException`. Let `MbException` propagate unchanged; do not catch it and continue.
- Catch checked or runtime exceptions only to add context, then rethrow as `MbUserException`, the subclass with a public constructor:

```java
} catch (NumberFormatException e) {
    throw new MbUserException(this, "evaluate()", "", "", e.toString(), null);
}
```

- Never write `catch (Exception e) { }` or log and return without propagating. That bypasses the flow's catch strategy and commits a broken message.
- Read the exception tree with `inAssembly.getExceptionList()`.

Source: [JavaCompute node exception handling](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=java-javacompute-node-exception-handling), [Accessing the ExceptionList tree using Java](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=node-accessing-exceptionlist-tree-using-java)

## Logging the error
- Every handled error must produce one log or error record containing at least: flow name (`MessageFlowLabel`), failing node (`Label`), error `Number`, root cause text with inserts, a correlation id (for example `MQMD.MsgId` or a business key carried in Environment) and a timestamp.
- Do not log full payloads that may contain personal or confidential data unless the user asks for it.
- Keep the original message unchanged on the error queue, or reference it by its correlation id, so it can be replayed after the fix.
- A Trace node with the pattern `${ExceptionList}` is useful while developing. IBM advises removing Trace nodes or switching them off (destination `None`) once the flow is tested.
- For diagnosis, the integration server exception log can be enabled in the `ExceptionLog` settings of `server.conf.yaml`. It records exceptions even when a flow handles them, in a rolling set of 4 files that overwrite the oldest data. Use it as a temporary diagnostic aid, not as the flow's error log.

Source: [Trace node](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=nodes-trace-node), [Enabling the exception log](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=flows-enabling-exception-log), [Viewing the exception log](https://www.ibm.com/docs/en/app-connect/13.0.x?topic=logs-viewing-exception-log)

## Related files
- [`skills/ace-error-handling/SKILL.md`](../ace-error-handling/SKILL.md)
- [`skills/shared/message-flow-rules.md`](message-flow-rules.md)
- [`skills/shared/esql-guidelines.md`](esql-guidelines.md)
- [`skills/shared/java-guidelines.md`](java-guidelines.md)
- [`skills/shared/node-types.md`](node-types.md)
- [`skills/shared/review-checklist.md`](review-checklist.md)
