## Validating generated .msgflow files (mandatory)

- Every .msgflow file you create or modify must be validated against the supplied XML Schema before you tell the user it is finished.
- Do not report a file as valid because it looks right to you. Only a validator's result counts.

### The schemas
- `msgflow.xsd` is the single entry point. Always validate against it.
- It depends on `GeneratedMessageFlow-Dispatch.xsd`, `GeneratedMessageFlow.xsd` and `xmi.xsd`. 
- These schemas contain references to one another which resolve due to their co-location in the same folder.
- These schemas use XSD 1.1 features, which determine what you can validate with

### The validator must support XSD 1.1
The schemas rely on `xs:alternative` (choosing a message flow node's definition from its `xmi:type`), `xs:assert`, `defaultOpenContent` and `defaultAttributes`. That is how the schemas check that each property belongs to the node it is on.

- When validating a message flow against msgflow.xsd, you can use any validator that conforms to XSD 1.1. Choose a validator that is already available. If none is available, tell the user what you need and offer to help set one up. Do not skip validation.
- Validators that implement only XSD 1.0 cannot be relied on for this purpose. Either they refuse the schema or they ignore the parts that matter, so a file can pass while being wrong. If your validator can be set to XSD 1.0 or 1.1 mode, select 1.1.
- If you cannot run any XSD 1.1 validator, say so plainly. Do not claim the file is valid. Describe what you could not check.

### Prove the validator is working before you trust it
A validator that fails to load the schema, or loads only part of it, can report "no errors" for every file. Before relying on a result, validate a copy of the file with one deliberate mistake, for example a made-up property name on a node. Confirm that the validator rejects it. Then validate the real file. If the broken copy is accepted, the setup is not checking anything. Fix that first.

Do the same for terminals, because they are checked by a separate part of the schema (an assertion) that a partly working validator can skip. In a second copy, change the `sourceTerminalName` of one connection to a made-up name such as `OutTerminal.bogus`, and confirm that it is rejected. Choose a connection whose source node is a built-in node type with fixed terminals (see "Terminal names" below). If you choose a subflow or user-defined node, the copy is correctly accepted and proves nothing.

### Procedure
1. Create the .msgflow file.
2. Validate it against `msgflow.xsd`.
3. If there are errors, fix the .msgflow file and validate again. Repeat until there are none. Fix the file, never the schema, and do not suppress or filter errors.
4. In your final answer, say which validator you used and report the result for each file. If a step was not possible, say which one.

### Understanding the errors
Wording varies between validators, but the meanings are consistent:
- A property not allowed on a node: it belongs to a different node type, or its name is misspelled. Remove it, or you chose the wrong node type.
- A required property is missing: that node type needs it. Add it.
- A value is not among the permitted values: use one of the listed values exactly as written.
- An identifier reference cannot be found: a connection or sticky note refers to a node that does not exist. Check the `xmi:id` values.
- An assertion failed: there are two possible causes. Either a node is missing `location`, `xmi:id` or `xmi:type`, or a connection uses a terminal name that its node does not have (see "Terminal names" below). Check both. Some validators report the second case against the whole `composition` element and do not name the connection, so you must find the bad connection yourself by comparing every `sourceTerminalName` and `targetTerminalName` with the terminals of the node it connects.

### Rules that keep the file valid
- Every node needs `xmi:type`, a unique `xmi:id`, and `location="x,y"`.
- `xmi:type` must be `<NodeName>.msgnode:FCMComposite_1` (for example `ComIbmMQInput.msgnode:FCMComposite_1`) or, for a subflow, `<SubflowName>.subflow:FCMComposite_1`. The node names recognised are listed in `GeneratedMessageFlow-Dispatch.xsd`.
- Do not invent property names. Each node type's permitted properties are defined in `GeneratedMessageFlow.xsd`, so look there before writing one.
- Never get a file to pass by changing a node to an unrecognised type, by deleting required properties, or by editing the schema.

### Terminal names
Every connection joins an output terminal of one node to an input terminal of another. The terminal names are written in the connection as `sourceTerminalName="OutTerminal.<name>"` and `targetTerminalName="InTerminal.<name>"`, for example `OutTerminal.out` and `InTerminal.in`.

- The schema checks the name against the list of terminals that IBM defines for the node's type. The source terminal must be an output terminal of the source node, and the target terminal must be an input terminal of the target node.
- This catches made-up names (`OutTerminal.bogus`), the wrong direction (an input terminal used as a source, or `OutTerminal.in`), and a terminal that exists on a different node type (for example `InTerminal.failure`, which is an output on every node that has it).
- Do not guess a terminal name. Each node type's terminals are listed in `GeneratedMessageFlow.xsd`, in the `<terminals>` section of that node type's annotation, each marked `direction="in"` or `direction="out"`. Look the node type up there before you write a connection. `GeneratedMessageFlow-Dispatch.xsd` holds the same lists in the form the check uses.
- Write `InTerminal.` or `OutTerminal.` in front of the identifier from that list, and keep the case exactly (for example `OutTerminal.EOD`, not `OutTerminal.eod`).
- Names are not checked against a list in these cases. Only the `InTerminal.` or `OutTerminal.` prefix is checked, so a wrong name can pass:
  - Nodes whose terminals depend on how they are configured. These are the Collector, DatabaseRoute, DotNetCompute, DotNetInput, EventInput, JSONataIf, Output, Request and Route nodes.
  - Subflow nodes, whose terminal names come from the subflow itself (for example `InTerminal.Input_1` and `OutTerminal.Output_1`) rather than from IBM's list.
  - User-defined nodes, nodes with an unrecognised `xmi:type`, and the node types where IBM has not provided information about the terminal names inside the MessageFlow.xsd file which is provided within the IBM App Connect Enterprise installation (and not in the Flow Pilot skill). There are two node types where IBM has not provided information about the terminal names, nodes ComIbmJMSIPOptimizedFlow and ComIbmMQOptimizedFlow, which have both been removed from all recent versions of the product.
  - For these you must work out the right names yourself from the node's own definition, and tell the user that the schema could not confirm them.

### What a pass does and does not mean
- A node whose `xmi:type` is not recognised is checked for structure only. It still passes, even if its properties are wrong. Make sure every `xmi:type` you write is one the schema recognises, or report it to the user in case the node has a new type which is user-defined or somehow not yet known by the schema, or due to a defect with the schema. These are situations that only a user can conclude upon, not an AI agent.
- Subflow nodes and user-defined nodes are not checked for properties.
- Terminal names are checked only for node types that have a fixed list (see "Terminal names" above). A connection to or from a subflow, a user-defined node, a node with dynamic terminals or an unrecognised node type passes with any name that has the right `InTerminal.` or `OutTerminal.` prefix. State this in your final answer for any such connection in the flow.
- The check confirms that a terminal exists on its node and points the right way. It does not check that the flow makes sense, for example that every node is wired up or that an error path is handled.
- The content of a node's `<translation>` is not checked.
- A passing file is schema-valid, not guaranteed to deploy or run.
- A file may fail because the schema requires a property that the IBM Toolkit supplies another way, for example through an operational policy or a pattern. If you think the schema is wrong, report the error to the user and ask. Do not silently work around it.