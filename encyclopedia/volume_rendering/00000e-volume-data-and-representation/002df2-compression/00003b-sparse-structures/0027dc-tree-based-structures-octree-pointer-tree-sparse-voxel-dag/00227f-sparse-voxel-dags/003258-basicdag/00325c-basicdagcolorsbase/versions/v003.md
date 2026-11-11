# BasicDAGColorsBase

Also called: BaseDAGColors

## Overview
BasicDAGColorsBase is a base class for managing color data within a Sparse Voxel DAG structure. It solves the problem of decoupling leaf count storage from the main geometry array by utilizing an enclosed leaves array. This separation allows for more flexible memory management and traversal strategies compared to tightly coupled geometry-color representations. It differs from its siblings, [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md) and [Mixture Graph](11065-mixture-graph.md), by providing the foundational storage mechanism for leaf counts without implementing specific compression or mixture logic itself.

## Shared principles
The core mechanism of BasicDAGColorsBase involves storing leaf counts for upper-level nodes in a dedicated structure, distinct from the primary geometry data. This design ensures that the number of leaves associated with a node can be accessed independently of the node's geometric position or color attributes.

The data layout consists of two primary components:
1.  `topLevels`: A `uint32` value storing the number of top levels in the DAG hierarchy. This defines the depth of the tree structure relevant to the color data management.
2.  `enclosedLeaves`: A `StaticArray<EnclosedLeavesType>` that stores the leaf counts for upper-level nodes. This array provides the necessary metadata for traversing the DAG and determining the extent of leaf nodes at various levels of the hierarchy.

This structure allows child classes to build upon a consistent interface for accessing leaf counts, facilitating operations such as error calculation or compression without needing to re-implement the basic storage logic.

## Subtopics

### [BasicDAGColorErrors](6743-basicdagcolorerrors.md)
Computes a binary error flag by comparing compressed and uncompressed color representations for adaptive quality control. It differs from other siblings by focusing exclusively on error metric generation rather than storage optimization or compression algorithms.

### [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md)
Stores color data in an uncompressed format. It differs by prioritizing direct access and simplicity over memory efficiency, suitable for scenarios where compression overhead is unacceptable.

### [Mixture Graph](11065-mixture-graph.md)
Implements a mixture model for color data. It differs by allowing complex blending of color attributes, suitable for advanced rendering techniques requiring material mixing.

## Background
The BasicDAGColorsBase class originates from the HashDAG project, an implementation of Sparse Voxel DAGs for volume rendering. It was developed to provide a robust foundation for managing color data within the DAG structure, enabling efficient traversal and manipulation of volume data. The class is part of a broader effort to optimize memory usage and performance in large-scale volume rendering applications.

## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
