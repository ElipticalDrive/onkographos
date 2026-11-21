# Spatial Hierarchies
Also called: Spatial Tree Structures
## Overview
Spatial Hierarchies organize data into tree structures to accelerate point location via recursive search [S1, unofficial]. They solve the problem of inefficient traversal in sparse scenes by reducing intersection tests.
## Shared principles
See [Hierarchical Data Structures](15869-hierarchical-data-structures.md) for shared family principles.
## Subtopics

### [Insertion into Spatial Hierarchies](13180-insertion-into-spatial-hierarchies.md)
Algorithmic process of adding primitives to a hierarchy. It solves the problem of maintaining spatial coherence in dynamic scenes. Unlike static build methods, it processes primitives incrementally, making it suitable for dynamic environments but sensitive to insertion order.

## Background
Spatial hierarchies such as octtrees, BSP trees, and KD trees were developed to optimize point location by recursive search [S1, unofficial].
## Sources
[S1] PPT - Ray Tracing 2: Acceleration PowerPoint Presentation, free ..., https://www.slideserve.com/chaim/ray-tracing-2-acceleration, fetched 2026-10-26 (unofficial)
