# BasicDAGColorsBase

Also called: BaseDAGColors

## Overview
BasicDAGColorsBase is a foundational data structure for managing color information within Sparse Voxel DAGs, specifically designed to decouple color storage from the primary geometry array. It solves the problem of memory fragmentation and traversal inefficiency by maintaining a separate, compact representation of leaf counts for upper-level nodes. It differs from its siblings, [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md) and [Mixture Graph](11065-mixture-graph.md), by providing a standardized, minimal interface for leaf count management rather than storing raw color data or complex mixture models.

## Shared principles
The family shares a specific data layout that separates the storage of leaf counts from the main DAG geometry. This allows for independent access to leaf counts without re-implementing storage logic for every child implementation.

The core structure consists of two main components:

1.  **`topLevels`**: A `uint32` value storing the number of top levels in the DAG hierarchy. This defines the depth at which the enclosed leaf counting begins.
2.  **`enclosedLeaves`**: A `StaticArray<EnclosedLeavesType>` storing the leaf counts for upper-level nodes. This array allows the system to know how many leaves are enclosed by each node in the upper hierarchy, facilitating efficient traversal and color retrieval.

The `EnclosedLeavesType` is not explicitly defined in the provided sources, but it is implied to be an integer type capable of storing the count of leaves. The `StaticArray` suggests a fixed-size or efficiently allocated array structure.

The primary mechanism is the decoupling of leaf count storage. By storing these counts in a separate array (`enclosedLeaves`) indexed by node or level, the system avoids the need to traverse the entire subtree to count leaves during color operations. This is particularly useful for error computation and adaptive quality control, as seen in [BasicDAGColorErrors](6743-basicdagcolorerrors.md).

## Subtopics

### [BasicDAGColorErrors](6743-basicdagcolorerrors.md)
A specific implementation that uses the leaf counts from `BasicDAGColorsBase` to retrieve corresponding color data for error calculation. It introduces a binary error flag mechanism for adaptive quality control.

### [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md)
(Sibling) Stores uncompressed color data directly.

### [Mixture Graph](11065-mixture-graph.md)
(Sibling) Uses a mixture model for color representation.

## Background
The concept originates from the work on Sparse Voxel DAGs, specifically within the context of efficient volume rendering and compression. The `BasicDAGColorsBase` class serves as a base class for DAG color data, as described in the unofficial source [S1, unofficial]. It is part of the `Phyronnaz/HashDAG` repository, which explores various DAG structures for volume data.

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
