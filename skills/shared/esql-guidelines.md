# ACE ESQL Guidelines

## Purpose
This document defines ACE-focused guidance for creating and refining ESQL used by Compute nodes.

## When to use
Use this document when generating or reviewing `.esql` files for ACE Compute nodes.

## BROKER SCHEMA usage

**Default  behavior: Do not use BROKER SCHEMA declarations unless explicitly required.**

### When NOT to use BROKER SCHEMA
- When the `.esql` file is placed in the project root directory
- When the user has not explicitly requested a specific schema organization
- For simple projects with a small number of ESQL files

### When to use BROKER SCHEMA
- Only when the user explicitly requests it
- When placing ESQL files in subdirectories for organizational purposes
- When the `.esql` file is in a Shared Library project with subdirectory structure

### File placement rules
- **No BROKER SCHEMA**: Place `.esql` file in project root
  - File: `ProjectName/ModuleName.esql`
  - ESQL: No `BROKER SCHEMA` declaration
  - Compute node reference: `esql://routine/#ModuleName.FunctionName`
  
- **With BROKER SCHEMA**: Place `.esql` file in subdirectory matching schema
  - File `ProjectName/com/ibm/example/ModuleName.esql`
  - ESQL: `BROKER SCHEMA com.ibm.example`
  - Compute node reference: `esql://routine/com.ibm.example#ModuleName.FunctionName`

### Example without BROKER SCHEMA (default)
CREATE COMPUTE MODULE MyCompute
    CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
	    -- implementation here
	    RETURN TRUE;
	END;
END MODULE;

Compute node property: `computeExpression="esql://routine/#MyCompute.Main"

## Compute node expression formats
The `computeExpression` property in a Compute node must match the ESQL file structure:

| ESQL Structure | computeExpression Format | Example |
|---|---|---|
| No BROKER SCHEMA | `esql://routine/#Module.Function` | `esql://routine/#EmployeeAPI_Compute.CreateEmployee` |
| With BROKER SCHEMA | `esql://routine/schema.path#Module.Function` | `esql://routine/com.ibm.example#EmployeeAPI_Compute.CreateEmployee` |

## Always DECLARE REFERENCE variables
- When using a CREATE statement with an AS clause, be sure to have declared the REFERENCE before the CREATE statement, as shown in the example below:
```esql
DECLARE myRef REFERENCE TO OutputRoot.JSON.Data.SaleList;
CREATE LASTCHILD OF OutputRoot.JSON.Data.SaleList AS myRef TYPE Name NAME 'Field1';
```

## Performance and tree navigation
- Declare `REFERENCE` variables when navigating heavily nested logical trees to reduce repeated navigation cost.
- Avoid unnecessary full tree copying when a change is limited to a small part of the message tree.
- Avoid multiple consecutive Compute nodes where a simpler design would achieve the same result.
- Avoid using `CARDINALITY` inside loops when the value can be determined once outside the loop.
- Minimize expensive string manipulation where a simpler approach is available.
- Minimize the number of statements when doing so improves clarity and efficiency.
- If an output field is known to be mandatory, it is important not to assign NULL using a SET statement (or if you do, ensure that SET is using the VALUE clause). This can be avoided using the ESQL COALESCE function which evaluates its parameters in order and returns the first one that is not NULL. The result is NULL if, and only if, all the arguments are NULL. The parameters can be of any scalar type, but they need not all be of the same type. The COALESCE function is often used to provide a default value for a field, which might not exist in a message. When using the COALESCE function, order your arguments by likelihood and cost: Place values that are frequently non-NULL first, and if you can put expensive expressions (such as subqueries or heavy functions) later in the argument list since COALESCE short-circuits as soon as it finds a match.

## When to use ESQL EXTERNAL variables
When declaring ESQL variables:
- Use EXTERNAL if the variable is to be used for defining configuration that is set at the time of deployment.
- An EXTERNAL variable has the same lifetime as the message flow which uses it.
- An EXTERNAL variable MUST be declared at the scope of the ESQL Module or Schema. It cannot be declared inside a Function or Procedure.
- An EXTERNAL variable carries a constant fixed value. It cannit be changed and is read-only.
- An EXTERNAL variable declaration is associated with a User-Defined Property (UDP) of the message flow. The value of a UDP which has been set in the Message Flow editor (which is then stored in the .msgflow file using the `eStructuralFeatures` element) takes precedence over a value which is assigned in a DECLARE statement in ESQL. UDPs are accessible in ESQL within the Compute, Database, DatabaseInput and Filter nodes. UDPs have global scope within a message flow, including subflows. This means that if you reuse the same subflow (with a UDP) more than once within a message flow each UDP must carry the same value.

## When to use ESQL SHARED variables
When declaring ESQL variables:
- A SHARED variable should be used when defining variables that will be changed at runtime AND which need to be kept across messages.
- A SHARED variable works well when you want an in-memory cache for example containing reference or lookup data that is expensive to fetch for every message. For example, a common pattern is to populate a SHARED ROW variable once from a database query when the first message arrives. Note that the data is not persisted (it is lost on restart or redeploy and is not transactional). A rollback will not undo changes.
- A SHARED variable MUST be declared at the scope of the ESQL Module or Schema. It cannot be declared inside a Function or Procedure.
- A SHARED variable is shared across all additional instances (multiple threads) of a message flow. When writing code using shared variables you must consider this, so typically when adopting the variable you should be within an ATOMIC section in order to avoid race conditions.
- A SHARED variable is NOT visible beyond a message flow, so you can't share it across integration servers. If you need that capability, it would be better to use the ACE Embedded Cache capability.
- A SHARED variable is held in memory, so if being used to hold a large cache this will be held in memory for the life of the flow, so size SHARED variables carefully.
- A SHARED variable is not automatically refreshed. If the source data changes, you will need a strategy to accommodate this such as a timestamp-based reload, a restart or a redeploy.
- Use neither EXTERNAL nor SHARED for defining variables that will be changed at runtime within the scope of a Compute node but will not be kept across messages.

## How to use the Environment tree from ESQL
- The Environment tree is always created with the in-memory logical tree, but the message flow itself (as opposed to the user's logic within the message flow) neither populates the Environment tree nor uses its contents. 
- The Environment tree (and any data you choose to add to it as a message flow developer) is propagated to subsequent nodes in the message flow.   
- You can use the Environment tree for your own purposes, as a scratchpad or working area to pass information from one node to another. 
- If you create a new output message in a Compute node, the Environment tree is also copied from the input message to the new output message. (In contrast to the LocalEnvironment tree, which is only included in the output message if you explicitly request that it is).
- There is only one Environment tree per message thread local storage (ie one message travelling through a flow does not see the Environment tree belonging to a different message travelling through the flow). Changes made in any message flow node are visible to all later nodes. They are also visible to earlier nodes if the message flows back, for example to a TryCatch or FlowOrder node. Changes to the Environment are retained when that happens.
- By convention, the best place to write your own data to the Environment tree is to a subtree Environment.Variables
- Use the Environment.Variables tree when you need to pass data between nodes that should survive loops, error paths, TryCatch handling and FlowOrder branches. Examples could be a correlation ID, a looked-up customer record, or the original request payload.
- Use the Environment.Variables tree when you want to keep data separate from the message body. This avoids polluting the payload and avoids re-parsing it. The Environment tree is useful if the data should travel with the message but without needing extra copying code in every Compute node.

## When using ESQL PROPAGATE
When you use the ESQL PROPAGATE function to control the routing of output messages from a Compute node, be aware of the impact of the `RETURN TRUE;` statement, which causes a message to be sent (propagated) to the Compute node's out terminal. Be particularly mindful when using ESQL IF / ELSE branching logic that you may have chosen to PROPAGATE when one of the conditions evaluates to true and forgotten that at the end of the Compute ESQL Module issue a final `RETURN TRUE;` which would cause an extra propagation to the out terminal.

## ESQL considerations for use of CopyMessageHeaders() and CopyEntireMessage()
When a new ESQL file is generated in the Toolkit, templates are provided for two very simple functions named CopyMessageHeaders() and CopyEntireMessage()
- In general CopyMessageHeaders() is typically used when you are building a new or transformed body but want to keep the headers (such as MQMD, MQRFH2 or HTTP)
- In general CopyEntireMessage() is typically used when the whole logical tree is duplicated in memory
- CopyMessageHeaders tends to be cheaper processing cost because the tendency is for this function to copy less data
- Do NOT call both CopyMessageHeaders() and CopyEntireMessage(). Use CopyEntireMessage() if you only need to make small edits to the existing body. If you are replacing or transforming large sections of the existing body then CopyMessageHeaders() is more appropriate as this avoids copying a body that you are about to overwrite.
- Do NOT copy data if you don't have to - set the Compute node's Mode property deliberately depending on whether you intend to change the message body. Only copy data which you need to the Environment tree. Reduce the number of Compute nodes where possible and avoid consecutive Compute nodes with nothing between them and combine their logic to avoid unnecessary tree copying.
- After using CopyMessageHeaders(), update anything that no longer fits the new message. For example delete headers that you shouldn't forward based on the outbound message transport(s) you intend to use. For example when switching between MQ and HTTP or vice versa, delete headers you won't need and shouldn't forward.
- When using CopyMessageHeaders() be careful with the loop's upper bound. WHILE I < J is deliberate. It stops before the last child, which the template assumes is the body. If your message has no body, or has extra trailing children, you will need to adjust it.

## Field existence checks
- Use `LASTMOVE` or `CARDINALITY` when checking whether a field exists.
- Prefer the most direct existence check that matches the tree navigation pattern being used.

## Tree construction
- Consider `CREATE ... PARSE` when it is more efficient than serializing and reparsing a copied logical tree.
- Be deliberate about Compute node copy behavior and avoid unnecessary message assembly duplication.

## Cross-domain transformation (XML → JSON)

**Never use a direct tree assignment to convert between message domains**, for example:

```esql
-- WRONG: copies XML tree structure verbatim, produces duplicate keys for repeated elements
SET OutputRoot.JSON.Data = InputRoot.XMLNSC.SomeElement;
```

### Step 1 — Analyse the XML structure before writing any ESQL

**Before writing a single line of ESQL**, examine the XML input and answer this question for every element at every level of nesting:

> "Can more than one of these elements appear as siblings under the same parent?"

If the answer is yes — even for a deeply nested element like `<Initial>` inside `<Invoice>` — that element **must** be mapped to a JSON array. There are no exceptions.

Build an explicit repeatability table before writing code:

| XML element | Can repeat? | JSON mapping |
|---|---|---|
| `SaleList/Invoice` | yes | JSON.Array |
| `Invoice/Initial` | yes | JSON.Array |
| `Invoice/Item` | yes | JSON.Array |
| `Item/Code` | yes | JSON.Array |
| `Invoice/Surname` | no | scalar |

Do not start writing ESQL until this table is complete.

### Step 2 — Initialise the output domain correctly

Use `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` followed by `CREATE FIELD OutputRoot.JSON.Data` to initialise the JSON output domain. **Do not use `SET OutputRoot.JSON.Data = NULL`** — that form does not correctly establish the JSON domain in ACE.

```esql
SET OutputRoot.Properties = InputRoot.Properties;
CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';
CREATE FIELD OutputRoot.JSON.Data IDENTITY (JSON.Object)Data;
```

### Step 3 — Declare JSON arrays explicitly before populating them

Every repeating element **must** be declared as a typed JSON array using `CREATE FIELD ... IDENTITY (JSON.Array)` before any elements are added to it. Do not rely on indexed `SET` alone to create arrays.

```esql
CREATE FIELD OutputRoot.JSON.Data.SaleList IDENTITY (JSON.Object)SaleList;
CREATE FIELD OutputRoot.JSON.Data.SaleList.Invoice IDENTITY (JSON.Array)Invoice;
```

### Step 4 — Iterate with REFERENCE + WHILE LASTMOVE, not FOR loops

Use `DECLARE ... REFERENCE TO element[1]` + `WHILE LASTMOVE(...) DO` + `MOVE ... NEXTSIBLING NAME '...'` to iterate repeating elements. Generate this form rather than `FOR srcX AS path[] DO`: it is the reference-variable pattern that the ACE performance guidance (ESQL code tips) recommends, and it keeps one loop shape for XMLNSC and JSON. `FOR` itself is valid ESQL and does traverse XMLNSC repeating elements (see the ESQL FOR statement), so do not report or rewrite an existing `FOR` loop as a defect. When the repeating elements are namespace-qualified, add the `NAMESPACE` clause: `MOVE ref NEXTSIBLING NAMESPACE ns NAME 'X'`. Without it, `NAME` matches only elements in no namespace (see the ESQL MOVE statement), so the loop stops after the first element, or moves to a same-named element in no namespace.

For each object in an array, create it explicitly with `CREATE LASTCHILD ... IDENTITY (JSON.Object)Item` before assigning its fields.

```esql
DECLARE invoiceRef REFERENCE TO InputRoot.XMLNSC.SaleEnvelope.SaleList.Invoice[1];
DECLARE invoiceIndex INTEGER 1;
WHILE LASTMOVE(invoiceRef) DO
    CREATE LASTCHILD OF OutputRoot.JSON.Data.SaleList.Invoice IDENTITY (JSON.Object)Item;

    -- Read scalars with FIELDVALUE()
    SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Surname = FIELDVALUE(invoiceRef.Surname);
    SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Balance = FIELDVALUE(invoiceRef.Balance);

    -- Nested array: declare it, then iterate
    CREATE FIELD OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Initial IDENTITY (JSON.Array)Initial;
    DECLARE initialIndex INTEGER 1;
    DECLARE initialRef REFERENCE TO invoiceRef.Initial[1];
    WHILE LASTMOVE(initialRef) DO
        SET OutputRoot.JSON.Data.SaleList.Invoice.Item[invoiceIndex].Initial.Item[initialIndex] = FIELDVALUE(initialRef);
        SET initialIndex = initialIndex + 1;
        MOVE initialRef NEXTSIBLING NAME 'Initial';
    END WHILE;

    SET invoiceIndex = invoiceIndex + 1;
    MOVE invoiceRef NEXTSIBLING NAME 'Invoice';
END WHILE;
```

### Step 5 — Always use FIELDVALUE() for scalar reads

When reading a scalar value from the XMLNSC tree, always wrap it in `FIELDVALUE()`. Using a bare field reference (e.g. `srcInvoice.Surname`) returns a logical tree node, not a string value, and will produce incorrect JSON output.

```esql
-- WRONG
SET OutputRoot.JSON.Data.Surname = invoiceRef.Surname;

-- CORRECT
SET OutputRoot.JSON.Data.Surname = FIELDVALUE(invoiceRef.Surname);
```

### Step 6 — Use DECLARE REFERENCE for navigation efficiency

Declare `REFERENCE` variables for deeply nested input paths to avoid repeated tree navigation:

```esql
DECLARE saleEnvelope REFERENCE TO InputRoot.XMLNSC.SaleEnvelope;
DECLARE saleList REFERENCE TO saleEnvelope.SaleList;
```

This rule applies whenever the source and target domains differ (XMLNSC→JSON, XMLNSC→JSONARRAY, etc.).

## Transformation fidelity
- Preserve the requested message domain and target structure exactly.
- Do not change an XML-to-JSON request into XML-to-XML or any other domain transformation.
- Do not invent new element names, field names, wrapper objects, or business semantics unless the user explicitly requests them.
- Do not add calculations, enrichment, default values, formatting changes, aggregation, or derived values unless the user explicitly requests them.
- If the user provides an expected output shape, example ESQL, or reference implementation, match that structure as closely as possible.
- For mapping tasks, prefer faithful structural mapping over a redesigned example transformation.

## Message domain and the Properties folder

### How the output message domain is set
- In a Compute node, the domain of the output message is set only by the parser folder that you create or assign to under `OutputRoot`.
- Create the folder explicitly, for example `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON';` or `CREATE LASTCHILD OF OutputRoot DOMAIN 'XMLNSC';`
- Alternatively, assign to a field beneath the parser folder, for example `SET OutputRoot.XMLNSC.Employee.SkillLevel = 'Basic';`
- Do not specify more than one body domain in the ESQL for a single output message.
- Domain names appear as node properties (such as `messageDomainProperty` in a `.msgflow` or `.subflow`) and as parser folder names. They are never fields inside the `Properties` folder.

### Valid fields in the Properties folder
The `Properties` folder contains only the following fields:

| Field | Data type |
|---|---|
| `CodedCharSetId` | INTEGER |
| `ContentType` | CHARACTER |
| `CreationTime` | TIMESTAMP |
| `Encoding` | INTEGER |
| `ExpirationTime` | TIMESTAMP |
| `IdentityMappedIssuedBy` | CHARACTER |
| `IdentityMappedPassword` | CHARACTER |
| `IdentityMappedToken` | CHARACTER |
| `IdentityMappedType` | CHARACTER |
| `IdentitySourceIssuedBy` | CHARACTER |
| `IdentitySourcePassword` | CHARACTER |
| `IdentitySourceToken` | CHARACTER |
| `IdentitySourceType` | CHARACTER |
| `MessageFormat` | CHARACTER |
| `MessageSet` | CHARACTER |
| `MessageType` | CHARACTER |
| `Persistence` | BOOLEAN |
| `Priority` | INTEGER |
| `ReplyIdentifier` | BLOB |
| `ReplyProtocol` | CHARACTER |
| `Topic` | CHARACTER (a list) |
| `Transactional` | BOOLEAN |

- **There is no `MessageDomain` field in the `Properties` folder.** Never generate `SET OutputRoot.Properties.MessageDomain = ...`. It is not valid and the ACE Toolkit reports a warning for it.
- Do not add `SET OutputRoot.Properties.<X> = ...` unless `<X>` is in the table above.
- When generating a new output message, only set `MessageSet`, `MessageType` and `MessageFormat` in `Properties` when the target parser needs them (for example MRM or DFDL). JSON and XMLNSC do not need them.
- The usual way to carry the properties through a Compute node is `SET OutputRoot.Properties = InputRoot.Properties;`

## ESQL file structure
Every `.esql` file for a Compute node must use this skeleton. The module name must match the name given to the Compute node in the `.msgflow` file. The file must be placed in the same Application project as the `.msgflow`.

```esql
CREATE COMPUTE MODULE <ModuleName>_Compute
	CREATE FUNCTION Main() RETURNS BOOLEAN
	BEGIN
		-- implementation here
		RETURN TRUE;
	END;
END MODULE;
```

The `.esql` filename must match the module name, e.g. `XMLToJSON_Compute.esql` for module `XMLToJSON_Compute`.

## REFERENCE vs FIELDVALUE vs direct assignment

Use the correct form depending on what you need:

| Intent | Correct form | Wrong form |
|---|---|---|
| Navigate to a subtree for repeated access | `DECLARE x REFERENCE TO path;` | Repeating the full path each time |
| Read a scalar string/number from XMLNSC | `FIELDVALUE(ref.Field)` | `ref.Field` (returns a tree node, not a value) |
| Copy a scalar from one domain to another | `SET out.Field = FIELDVALUE(in.Field)` | `SET out.Field = in.Field` |
| Move a reference to the next sibling | `MOVE ref NEXTSIBLING NAME 'X';` | n/a — no shorthand alternative |

Always use `FIELDVALUE()` when reading leaf values from an XMLNSC tree for output to a different domain (JSON, MRM, etc.).

## Flow design considerations
- Prefer clean flow design over wiring every failure terminal individually when a consistent catch strategy is more suitable.
- Apply [`skills/shared/error-handling-rules.md`](error-handling-rules.md) for `THROW`, handlers, database errors and reading the ExceptionList.
- Consider whether a subflow or shared library is more appropriate when logic is repeated.

## Validity guardrails
- Do not invent ESQL built-in functions or statements.
- There is no `DOUBLE` data type in ESQL. ESQL does have `DECIMAL`, `FLOAT` and `INTEGER` numeric data types.
- There is no `RANDOM` function in ESQL although there is a `RAND` function.
- `CREATE OUTPUTROOT DOMAIN(...)` is not a valid ESQL statement — do not generate it.
- `SET OutputRoot.JSON.Data = NULL` does not correctly establish the JSON domain — use `CREATE LASTCHILD OF OutputRoot DOMAIN 'JSON'` + `CREATE FIELD OutputRoot.JSON.Data` instead.
- Do not invent JSON scalar type syntax such as `JSON.String`, `JSON.Number`, or `JSON.Boolean`.
- If a requested function or syntax is uncertain, use known ACE ESQL constructs rather than guessing.

## Known valid ACE ESQL constructs
- `FIELDVALUE(...)` is a valid ACE ESQL built-in and must not be replaced because it is described as unresolved.
- `LASTMOVE(...)` is a valid ACE ESQL built-in for reference navigation.
- `MOVE ref NEXTSIBLING NAME 'X';` is valid ACE ESQL syntax.
- `TYPE Name` is valid ACE ESQL syntax for explicit JSON object creation.
- `CREATE FIELD ... IDENTITY (JSON.Array)` is valid ACE ESQL syntax for explicit JSON array creation.
- Create JSON scalar values by assigning them with `SET outRef.Field = value` rather than inventing typed scalar nodes.
- Do not claim that these constructs are invalid unless they are being used in a way that contradicts the ACE guidance in this file.

## ESQL Validation
- When creating new ESQL files or updating existing ESQL files you MUST validate them if possible (this depends on whether you have access to `esql-validate.exe` described below)
- `esql-validate.exe` is provided as part of the ACE product install (from version 13.0.9.1 and above) that can help you use correct syntax when creating `.esql` files
- The `esql-validate.exe` can be found in the ACE installation at `<ACE Install Location>/server/bin`
- For example the location within the default install location on Windows is `C:\Program Files\IBM\ACE\13.0.M.F\server\bin` where M represents the ACE Modification release number and F represents the ACE Fixpack release number such as `13.0.9.1` In general pick the latest available release, and therefore the directory which first of all has the highest available number M, and from within that choice, the highest available number F. So for example 13.0.9.1 would be picked in preference to 13.0.8.2
- For example the location within the default install location on Linux is `/opt/IBM/ace-13.0.M.F/server/bin` where M represents the ACE Modification release number and F represents the ACE Fixpack release number such as `13.0.9.1` In general pick the latest available release, and therefore the directory which first of all has the highest available number M, and from within that choice, the highest available number F. So for example 13.0.9.1 would be picked in preference to 13.0.8.2
- For example the location within the default install location on MacOS is `/Applications/IBM App Connect Enterprise 13.0.M.F/server/bin` where M represents the ACE Modification release number and F represents the ACE Fixpack release number such as `13.0.9.1` In general pick the latest available release, and therefore the directory which first of all has the highest available number M, and from within that choice, the highest available number F. So for example 13.0.9.1 would be picked in preference to 13.0.8.2
- If you do not have access to the `esql-validate.exe` file you should ask the user to provide it from the above location.
- To validate ESQL using `esql-validate.exe` use the command with a single parameter which represents the absolute path to the directory on disk holding the ESQL files which you have generated: `esql-validate.exe <AbsoluteFilePathToDirectoryContainingESQL> --json` The outputs of the command will tell you if there are any syntax issues. If there are issues you MUST correct them and then repeat the validation process until there are no further errors
- If `esql-validate.exe` is not available on the local machine (because for example the ACE product is not installed) then warn the user about this and then proceed without validating

## Related files
- [`skills/ace-esql/SKILL.md`](../ace-esql/SKILL.md)
- [`skills/shared/review-checklist.md`](review-checklist.md)
