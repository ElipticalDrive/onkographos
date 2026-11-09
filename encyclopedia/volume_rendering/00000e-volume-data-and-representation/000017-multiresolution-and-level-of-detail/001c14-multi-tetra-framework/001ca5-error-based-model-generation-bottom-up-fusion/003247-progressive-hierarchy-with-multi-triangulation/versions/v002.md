# Progressive Hierarchy with Multi-triangulation

Also called: Multi-triangulation LOD

## Overview
A dynamic Level-of-Detail (LOD) technique for volume rendering on unstructured grids that creates a progressive hierarchy of tetrahedra stored in a multi-triangulation data structure [S1, S2, unofficial]. It differs from static LOD methods by dynamically updating the mesh to achieve interactive frame rates, building on the bottom-up fusion principles of [Error-Based Model Generation (Bottom-Up Fusion)](1306-error-based-model-generation-bottom-up-fusion.md) by encoding all necessary simplification operations in advance [S1, S2, unofficial].

## How it works
**Inputs:** A high-resolution tetrahedral mesh representing the volume data [S1, S2, unofficial].
**Outputs:** A dynamically updatable multi-triangulation structure allowing selective refinement to a target LOD [S1, S2, unofficial].

**Data Structures:**
The core structure is a multi-triangulation, which encodes the history of edge-collapse operations [S1, S2, unofficial]. Each operation links a parent tetrahedron to its child tetrahedra, forming a hierarchy [S1, S2, unofficial]. Exact layout fields, sizes, and bit packing are Not given in the sources.

**Steps:**
1. Pre-computation of Simplification Path: Perform a bottom-up simplification of the input tetrahedral mesh [S1, S2, unofficial]. This involves iteratively collapsing edges to reduce the number of tetrahedra until a base mesh (lowest resolution) is reached [S1, S2, unofficial].
2. Construction of Multi-triangulation: Store the base mesh and the sequence of edge-collapse operations in the multi-triangulation data structure [S1, S2, unofficial]. This structure allows for the dynamic reconstruction of any intermediate LOD level by applying a subset of the recorded operations [S1, S2, unofficial].
3. Dynamic Refinement: To render at a specific LOD, select a subset of edge-collapse operations to apply to the base mesh [S1, S2, unofficial]. This selection is guided by selection heuristics [S1, S2, unofficial].
4. Rendering: Render the resulting tetrahedral mesh [S1, S2, unofficial].

**Formulas:** Not given in the sources.

**Parameters:**
* Target LOD: The desired level of detail, often determined by performance constraints or visual fidelity requirements [G].
* Selection Heuristics: The criteria for choosing which edge collapses to apply [S1, S2, unofficial].

**Edge Cases and Pitfalls:**
* Dynamic Update Overhead: The cost of dynamically updating the mesh must be low enough to maintain interactive frame rates [S1, S2, unofficial].
* Visual Artifacts: Improper selection of edge collapses may lead to visual artifacts or inconsistencies in the volume representation [G].

## Operating characteristics
* Time Complexity: Pre-computation of the multi-triangulation is offline. Dynamic refinement time depends on the number of edge collapses applied and the efficiency of the selection heuristics [G].
* Memory Complexity: Depends on the size of the base mesh and the storage required for the sequence of edge-collapse operations in the multi-triangulation [G].
* Strengths: Enables interactive rendering of large unstructured grids by dynamically adjusting detail. Builds on established bottom-up simplification techniques [S1, S2, unofficial].
* Limitations: Requires pre-computation of the simplification hierarchy. Performance is dependent on the efficiency of the dynamic update mechanism [G].
* When to use: When interactive rendering of large unstructured tetrahedral grids is required and hardware capabilities vary [S1, S2, unofficial].
* When not to use: When offline rendering is acceptable and static LOD is sufficient, or when the pre-computation cost is prohibitive [G].

## Background
Proposed by Cignoni et al. [S1, S2, unofficial], this technique is part of the broader effort to enable GPU-based volume rendering of unstructured grids. It addresses the challenge of rendering large, complex tetrahedral meshes interactively by leveraging dynamic LOD techniques. The approach is similar in spirit to other dynamic LOD methods, such as those by Museth and Lombeyda, but specifically tailored for tetrahedral meshes using a multi-triangulation structure [S1, S2, unofficial].

## Sources
[S1] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S2] A Survey of GPU-Based Volume Rendering of Unstructured Grids., https://www.researchgate.net/publication/220162223_A_Survey_of_GPU-Based_Volume_Rendering_of_Unstructured_Grids, fetched 2026-10-28 (unofficial)
