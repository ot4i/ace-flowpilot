# ACE Message Flow Comparison

## Purpose

This document provides guidance for particular items to consider when comparing `.msgflow` files for IBM App Connect Enterprise (ACE)

## When to use

Use this document when comparing two or more message flows

## Artifact type

- ACE Toolkit `.msgflow` files.
- Do not confuse ACE Toolkit `.msgflow` files with ACE Designer YAML artifacts.

## Core rules

When comparing message flows:

- You should note the specific disk location of the .msgflow files which are being compared and you MUST report on the date and time when they were last modified.

- If the message flow has a defined version number (in the `.msgflow` xml this is saved in the attribute named `string` of the `version` element which is a child of `eClassifiers`) then you MUST report that as part of the comparison summary.

- You should include the Project name (and Broker Schema subdirectory if the `.msgflow` is not in the project root) where the message flow is located.

- Report the total number of message flow nodes in each flow

- Specifically highlight differences regarding which input nodes (which cause the flow to be triggered) are present in each message flow.

- Highlight differences between the message flows in terms of the message flow nodes which they contain (expressed in terms of the node types and the node names) but in particular note the areas where differences in the message flow nodes will not directly influence the ability of the flow to process data such as instances of Pass through nodes, Trace nodes, and Log nodes.

- Specifically highlight any (non-input) message flow nodes that do not have ANY input terminals connected.

- Do not focus on differences in the `.msgflow` files which only provide cosmetic impact to the way that the Toolkit displays the flow. This includes:
  - Ignore differences in the value of the `location` attribute of instances of the `nodes` element. If you find this type of difference, just note to the user that the flows might look slightly different when opened in the Toolkit even though their runtime functional behaviour is not impacted.
  - Ignore differences in the `colorGraphic16` and `colorGraphic32` xml elements
  - Ignore the `nodeLayoutStyle` attribute of the `eClassifiers` element
  - Ignore differences in the content of the `stickyBoard` element (which is a parent of `stickyNote`) if it appears in the `.msgflow` (a child of `eClassifiers` element)

- Highlight in particular different paths that can be travelled through the flows being compared. Are there additional nodes that have been inserted into one of the flows compared to the other? Are there differences in the terminals of specific nodes that have been connected?

- Are there specific message flow node properties that carry different values when the flows are compared? In particular look for differences in the value of the messageDomainProperty which controls the parser used by an input node or mid-flow node when converting data into the internal logical tree structure which passes from one node to the next. Differences in this value can cause data to be propagated in different directions through a message flow.

- Highlight differences in the definition of message flow user defined properties which are located within the `propertyOrganizer` section (child of `eClassifiers`) of the .msgflow

- Where possible, draw separate mermaid diagrams with one representing each message flow that is being compared (use boxes to represent message flow nodes, and arrows to represent each connected message flow node terminal)
