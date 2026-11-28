# BasicDAGColorsBase
Also called: BaseDAGColors
## Overview
BasicDAGColorsBase is a base class for DAG color data that decouples leaf count storage from the main geometry array using an enclosed leaves array [S1, unofficial]. It differs from sibling methods by establishing the base array layout for enclosed leaf counts, whereas [BasicDAGUncompressedColors](6741-basicdaguncompressedcolors.md) focuses on raw storage and [Mixture Graph](11065-mixture-graph.md) introduces mixture-based color representations.
## Shared principles
The core mechanism relies on an `enclosedLeaves` array to determine the number of leaves, which dictates the scope of subsequent operations such as color error computation [BasicDAGColorErrors](6743-basicdagcolorerrors.md). This structure decouples leaf count storage from the main geometry array [S1, unofficial]. The data layout consists of:
- `topLevels`: A `uint32` value storing the number of top levels in the DAG hierarchy [S1, unofficial].
- `enclosedLeaves`: A `StaticArray<EnclosedLeavesType>` storing leaf counts for upper-level nodes [S1, unofficial].
## Subtopics

### [BasicDAGColorErrors](6743-basicdagcolorerrors.md)
Provides a binary quality control mechanism for sparse voxel DAG color data by computing an error flag between compressed and uncompressed color representations. It differs from sibling methods that may store continuous error metrics or skip validation by returning a strict binary indicator (1.0 or 0.0) based on a fixed threshold, ensuring adaptive quality control without storing additional floating-point error values per leaf.

## Background
The concept originates from the HashDAG implementation [S1, unofficial]. Not given in the sources is the specific development history or additional architectural details beyond the base class definition.
## Sources
[S1] Phyronnaz/HashDAG, https://github.com/Phyronnaz/HashDAG, fetched 2026-10-26 (unofficial)
