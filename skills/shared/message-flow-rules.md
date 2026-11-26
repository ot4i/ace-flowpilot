# ACE Message Flow Rules

## Purpose
This document defines generic rules for creating `.msgflow` files for IBM App Connect Enterprise (ACE) Toolkit.

## When to use
Use this document for generic message flow creation and review before consulting connector-specific guidance.

## Artifact type
- Create ACE Toolkit `.msgflow` files.
- Do not confuse ACE Toolkit `.msgflow` files with ACE Designer YAML artifacts.

## Core rules
- Every node added to a message flow must use the exact validated `xmi:type` value for that node.
- Do not invent namespace prefixes or node type names.
- Do not rely on old example `.msgflow` files as the source of truth for `xmi:type`.
- For each node type used in a message flow, you must declare a namespace in the `<ecore:EPackage>` element and then use the declared prefix in the node's `xmi:type` attribute
- For example, in `node-types.md` the HTTP Input node → namespace prefix of `ComIbmWSInput.msgnode`
  - Node type reference: "namespace prefix of `ComIbmWSInput.msgnode`"
  - Namespace declaration: `xmlns:ComIbmWSInput.msgnode="ComIbmWSInput.msgnode"`
  - xmi:type usage: `xmi:type="ComIbmWSInput.msgnode:FCMComposite_1"`
- Validate node type mappings against the ACE Message Flow schema for the relevant ACE version.
- Within the .msgflow use a root tag name of `ecore:EPackage`
- Within the `ecore:EPackage` element, the .msgflow should always carry the following hardcoded attribute values `xmi:version="2.0" xmlns:xmi="http://www.omg.org/XMI" xmlns:ecore="http://www.eclipse.org/emf/2002/Ecore" xmlns:eflow="http://www.ibm.com/wbi/2005/eflow" xmlns:utility="http://www.ibm.com/wbi/2005/eflow_utility"`
- Within the .msgflow create a child element of `ecore:EPackage` with name of `eClassifiers`
- Within the `eClassifiers` element, create a child element with content `<eSuperTypes href="http://www.ibm.com/wbi/2005/eflow#//FCMBlock"/>`
- Within the `eClassifiers` element, create a child element with content `<colorGraphic16 xmi:type="utility:GIFFileGraphic" resourceName="platform:/plugin/ProjectName/icons/full/obj16/FlowName.gif"/>` where `ProjectName` must be substituted with the name of the project in which the message flow is being created and `FlowName` must be substituted with the name of the message flow.
- Within the `eClassifiers` element, create a child element with content `<colorGraphic32 xmi:type="utility:GIFFileGraphic" resourceName="platform:/plugin/ProjectName/icons/full/obj30/FlowName.gif"/>` where `ProjectName` must be substituted with the name of the project in which the message flow is being created and `FlowName` must be substituted with the name of the message flow.
- Within the `.msgflow` use a `composition` element to contain multiple child elements named `nodes` and `connections`
- Unless the user explicitly asks for square shaped message flow nodes rather than rectangles, when creating a `.msgflow` specify an attribute of `nodeLayoutStyle="RECTANGLE"` within the `eClassifiers` element.

## Node type reference
Use [`skills/shared/node-types.md`](node-types.md) for the canonical node mapping list.

## Connector-specific rules
If the requested node is connector-specific, also read:
- [`skills/shared/connector-index.md`](connector-index.md)
- the relevant file under [`skills/shared/connectors/`](connectors)

## Simplicity
Use the minimum set of nodes required to satisfy the user's request.
- Do not add a Compute node or ESQL file unless the user explicitly asked for message transformation or routing logic.
- A direct wire from an Input node to a Reply node is the correct implementation of a pass-through or echo flow.
- Do not add nodes, files, or abstractions "just in case" — every node must trace directly to a stated requirement.

## Required inputs
If a node requires a value that the user has not provided, ask for it unless there is an obvious safe default.
For example:
- MQ Input nodes require a queue name.
- HTTP Input nodes require a URLSpecifier whose value starts with a leading `/` character
If the flow includes a Compute node that reads from a specific input domain (XMLNSC, JSON, etc.), the upstream Input node's `messageDomainProperty` must be set to that domain.

## Error handling for input nodes
These rules apply to every flow with an MQ or HTTP input node, even when the user does not mention errors. Before generating the flow, you MUST read [`skills/ace-error-handling/SKILL.md`](../ace-error-handling/SKILL.md) and [`skills/shared/error-handling-rules.md`](error-handling-rules.md).
- Wire the input node `catch` terminal to an error handler, or state in the response why the node default is acceptable.
- Every catch flow must either handle the error (log, deliver, commit) or rethrow it (log, throw, roll back). Say which one.
- MQ Input: keep `Transaction mode` `Yes` (the default). Ask for the `BOTHRESH` and `BOQNAME` values of the input queue, or state them as an assumption.
- HTTP Input: every catch path ends in an HTTP Reply node with an explicit status code and a short JSON error body.
- Do not add Try Catch nodes, Throw nodes or extra `failure` wiring unless the user asks for them.
- End the response with an error strategy summary: where errors are caught, the transaction outcome, where failed messages go, and what is logged.

## Message Flow naming conventions
- Message Flow names must be unique within an integration server.
- Message Flow node names must be unique within the flow, even across node types, so for example two MQOutput nodes within the same flow each need their own name and as another example you cannot name two nodes of different types such as a Compute node and a JavaCompute node with the same name in the same message flow.
- Pick a message flow name that reflects the flow's function for example like `OrderProcessing`
- Keep paths short. ESQL and map file paths must not exceed 256 characters on Windows. If they do, the compiled flow isn't generated and can't be deployed. Keep the names of ESQL files, mapping files, projects and broker schemas short where possible.
- When defining Label nodes, the Label node's Label Name property must be unique within a message flow. This is also true if a subflow contains a Label node and is then embedded more than once in a parent flow.
- In general do not leave nodes with default names such as `Compute` or `Compute1` because Monitoring events, statistics and error messages use these names so they are not very descriptive for a human to understand at a glance what the purpose of the node is within the context of the specific flow.
- Avoid spaces and special characters in then names of message flows, applications and libraries. They can cause trouble in command-line tools, BAR overrides and scripts.
- Avoid environment names such as _DEV or _PROD in message flow names. Use configuration for environment differences.

## Validation
Before returning generated `.msgflow` content:
- you MUST read [`skills/shared/FlowValidation/message-flow-validation.md`](message-flow-validation.md)
- confirm the node types are valid
- you MUST read the example message flow in an application project [`./ExampleApplication/Example.msgflow`]
- confirm connector-specific rules were applied where relevant
- confirm the error handling for input nodes rules were applied, and the response includes the error strategy summary
- confirm any required supporting artifacts were also created or identified
- apply the common checks in [`skills/shared/review-checklist.md`](review-checklist.md)

## Examples 
- When creating `.msgflow` content in an Application project note the example [`skills/shared/ExampleApplication/Example.msgflow`](Example.msgflow)
- When creating `.msgflow` content in a REST API project note the example [`skills/shared/ExampleAPI/gen/ExampleAPI.msgflow`](ExampleAPI.msgflow)
- When creating `.subflow` content in a REST API project note the example [`skills/shared/ExampleAPI/createWidget.subflow`](createWidget.subflow), [`skills/shared/ExampleAPI/retrieveWidget.subflow`](retrieveWidget.subflow), [`skills/shared/ExampleAPI/updateWidget.subflow`](updateWidget.subflow), [`skills/shared/ExampleAPI/deleteWidget.subflow`](deleteWidget.subflow)

## Related files
- [`skills/shared/ace-versions.md`](ace-versions.md)
- [`skills/shared/node-types.md`](node-types.md)
- [`skills/shared/review-checklist.md`](review-checklist.md)
- [`skills/shared/ExampleApplication/Example.msgflow`]
- [`skills/shared/ExampleAPI/gen/ExampleAPI.msgflow`]
- [`skills/shared/ExampleAPI/createWidget.subflow`]
- [`skills/shared/ExampleAPI/retrieveWidget.subflow`]
- [`skills/shared/ExampleAPI/updateWidget.subflow`]
- [`skills/shared/ExampleAPI/deleteWidget.subflow`]
- [`skills/shared/FlowValidation/message-flow-validation.md`]
