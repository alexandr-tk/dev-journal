---
project: Blender
tags: [c++, glsl, systems-architecture, open-source, gsoc]
status: Proposal Submitted (Awaiting GSoC Results)
---

# Architecting a GSoC Proposal: Native Types and Node Parity in the Compositor

**The Result:** Finalized [90-hour](https://docs.google.com/document/d/1_ZbeRayYowTjjrMMiOyJxCs2gzyoCwXjkkz_AgfSxoQ/edit?usp=sharing)
and [175-hour](https://docs.google.com/document/d/18j9eI6JBnmUSVOIag35sbC-QoJMkaOQUXlHnLtccbG4/edit?usp=sharing) Project Architecture and Proposal

## 1. Workflow

- **Task:** Conduct a comprehensive technical audit of Blender's node systems, define project scope, and architect a formal Google Summer of Code (GSoC) proposal to bring Geometry Nodes parity to the Real-Time GPU Compositor.
- **Reasoning:** - Identify a high-impact, architecturally sound project that solves a core limitation in Blender's compositing pipeline.
    - Continuously iterate on technical scope through direct communication with core module owners to align with Blender's active `main` branch development.
    - Secure a rigorous 175-hour project timeline that balances backend systems infrastructure with tangible UI deliverables.

## 2. Context

Blender's Compositor and Geometry Nodes environments currently operate on different underlying paradigms. The Compositor routes all data, including integers and booleans, through floating-point buffers. This forced float-casting creates critical limitations: boolean logic suffers from fractional alpha bleeding, and integer/bit math (like Modulo operations or Cryptomatte unpacking) experiences floating-point inaccuracies.

The objective of this project was to design a technical roadmap to implement native `int[234]` and `boolean` data types into the Compositor's C++ evaluator and GLSL compiler, and subsequently expose compatible Geometry Nodes to utilize this new infrastructure.

## 3. Requirements Gathering: The Node Survey

Before defining the exact deliverables, I conducted an exhaustive technical [survey of all 314 existing Geometry Nodes](https://docs.google.com/spreadsheets/d/1ux5dBb3liSjhjqsY-RPYL7YSD66ZxLS0jlsjn4-O0Jw/edit?usp=sharing). The goal was to systematically identify nodes that could be viably ported to the 2D Compositor context.

## 4. Iterative Scoping & Cross-Module Collaboration

Defining the project scope required five distinct architectural drafts, shaped by constant repository analysis and back-and-forth communication with core developers across multiple Blender modules.

### Pre-Draft & Draft 1: The Architectural Blocker

The initial concept was simply a list of nodes to port. However, core developer feedback quickly highlighted a major architectural blocker: the GPU Compositor lacked native support for `int` and `bool` types. Draft 1 was updated to include this foundational type-support work, though the technical synopsis remained overly broad.

### Draft 2: Cross-Module Dependencies & CPU Reusability

Feedback on Draft 2 introduced a new constraint: introducing native types required consensus beyond the Compositor module, specifically needing buy-in from the Nodes and Physics teams. Furthermore, the draft incorrectly implied that I would be writing CPU node implementations from scratch. The proposal was corrected to emphasize reusing existing `Function Node` C++ logic, minimizing redundant work and optimizing the timeline.

### Draft 3: Module Alignment

Following communication with the Nodes team, cross-module approval was secured confirming that the native type architecture was viable. During this phase, the node selection was guided by the viability survey, though the survey itself had not yet been formally linked or documented within the proposal text.

### Draft 4: Upstream Merges & The Scope Split

Draft 4 formally integrated the node survey to justify the technical selections. However, continuous monitoring of the `main` branch revealed a scope clash: the planned Matrix nodes had just been implemented and merged by a core developer. Removing these nodes left the remaining scope too thin to justify a 175-hour project.

This required a strategic pivot. I split the planning into two paths: a baseline 90-hour scope, and an expanded 175-hour proposal that introduced advanced **GPU Matrix Decomposition**. This expansion included a custom `Singular Value Decomposition (SVD)` solver and the `Separate/Combine Transform` nodes.

### Draft 5 (Final Submission): The 175-Hour Architecture

The final submission successfully integrated all developer feedback. It removed redundant upstream nodes, formalized the cross-module native type architecture, and solidified a mathematically rigorous 175-hour timeline anchored by the advanced matrix decomposition tasks.

## 5. Prototyping and Validation

To validate the technical feasibility of the proposal and familiarize myself with the node transfer process, I developed a working Proof of Concept (POC) prior to final submission. I implemented the **Integer Math** node within the Real-Time Viewport Compositor.

## 6. Final Architecture & Next Steps

The finalized software engineering proposal establishes a complete pipeline for updating the Compositor ecosystem. The deliverables are structured sequentially across four phases:

1.  **Native GPU Type Architecture:** Implementing `int[234]` and `boolean` buffers directly in the Compositor context and updating the implicit GLSL type-conversion pipeline.
2.  **Core Math & Logic Porting:** Porting standard primitives (Integer Math, Boolean Math, Bit Math, Compare) to strictly utilize the new data types.
3.  **Input & Procedural Utilities:** Integrating input interfaces (Input Integer, Input Boolean, Menu Switch) and generation nodes (Random Value, Hash Value).
4.  **Matrix Decomposition:** Developing the GLSL SVD solver and the Separate/Combine Transform suite alongside comprehensive regression testing.

The proposal has been officially submitted, and the project is currently awaiting final GSoC selection results.
