# BasicDAGColorsBase

Also called: BaseDAGColors

## Overview
BasicDAGColorsBase is the foundational color storage mechanism for the BasicDAG sparse voxel structure, designed to decouple leaf node metadata from the primary geometry array. It solves the problem of efficiently tracking the number of leaf voxels associated with upper-level nodes without embedding this count directly into the main DAG node indices. This separation allows the geometry traversal logic to remain independent of color data management, facilitating modular updates to leaf counts during compression or refinement.

## Shared principles
The family shares a specific memory layout strategy where leaf counts are stored in a dedicated, enclosed array rather than interleaved with the DAG node indices. This design ensures that the core DAG structure, which stores node indices in a flat array for fast traversal, remains compact and uniform.

### Data Layout
The structure consists of two primary components:
1.  **topLevels**: A `uint32` value storing the total number of top levels in the DAG hierarchy. This defines the depth of the tree structure relevant to the color data management.
2.  **enclosedLeaves**: A static array (`StaticArray<EnclosedLeavesType>`) that stores the leaf counts for upper-level nodes. The type `EnclosedLeavesType` is determined by the specific implementation but generally aligns with the integer size required to represent the maximum number of leaves in a subtree.

### Decoupling Mechanism
By enclosing the leaf counts, the BasicDAGColorsBase class isolates the metadata required for color processing (such as error calculation or compression) from the geometric topology. This allows sibling implementations to vary how they store or compress the actual color values while maintaining a consistent interface for leaf count access.

## Subtopics

### [BasicDAGColorErrors](6743-basicdagcolorerrors.md)
Extends the base class to compute binary error flags by comparing compressed and uncompressed color representations, enabling adaptive quality control.

### [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md)
Stores raw, uncompressed color data for leaves, providing a baseline for error calculation and visualization.

## Background
The BasicDAGColorsBase class originates from the HashDAG project, an implementation of Sparse Voxel DAGs for volume rendering. It was developed to address the inefficiencies of storing leaf metadata within the main DAG index array, which could lead to cache misses or increased memory footprint during traversal. By separating leaf counts, the structure supports more flexible compression strategies and error-based refinement.

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
