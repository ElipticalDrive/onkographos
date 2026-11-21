# BasicDAGColorsBase
Also called: BaseDAGColors
## Overview
BasicDAGColorsBase is the foundational family for managing color data within Sparse Voxel DAG structures. It solves the problem of decoupling leaf count storage from the primary geometry data, ensuring that color metadata remains consistent with the hierarchical structure. It differs from sibling methods by establishing the base array layout for enclosed leaf counts, whereas [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md) focuses on raw storage and [Mixture Graph](11065-mixture-graph.md) introduces mixture-based color representations.
## Shared principles
The core mechanism relies on an `enclosedLeaves` array to determine the number of leaves, which dictates the scope of subsequent operations such as color error computation or compression. This structure decouples leaf count storage from the main geometry array.

The data layout consists of:
- `topLevels`: A `uint32` value storing the number of top levels in the DAG hierarchy.
- `enclosedLeaves`: A `StaticArray<EnclosedLeavesType>` storing leaf counts for upper-level nodes.

This structure ensures that the leaf count is explicitly tracked and accessible for any child class that requires per-leaf processing, such as error calculation or color blending.
## Subtopics

### [BasicDAGColorErrors](6743-basicdagcolorerrors.md)
Provides a binary quality control mechanism for sparse voxel DAG color data by computing an error flag between compressed and uncompressed color representations. It differs from sibling methods that may store continuous error metrics or skip validation by returning a strict binary indicator (1.0 or 0.0) based on a fixed threshold, ensuring adaptive quality control without storing additional floating-point error values per leaf.

## Background
The concept originates from the HashDAG implementation, which utilizes a directed acyclic graph structure for volume geometry. The specific color base class was developed to manage color data efficiently within this sparse voxel framework.
## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
