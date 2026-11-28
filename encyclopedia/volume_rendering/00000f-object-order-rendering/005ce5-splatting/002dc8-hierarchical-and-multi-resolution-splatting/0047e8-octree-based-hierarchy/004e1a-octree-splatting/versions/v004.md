# Octree Splatting
## Overview
Octree Splatting is a hierarchical object-order volume rendering technique that renders non-leaf octree nodes as single primitives to accelerate the visualization of dense volumetric data [S1, unofficial][S2, book]. It distinguishes itself from standard splatting by actively utilizing internal tree nodes as renderable kernels, rather than restricting them to occlusion culling [S1, unofficial][S2, book].

## How it works
**Inputs:** A volume dataset organized into an octree structure [G].
**Outputs:** A 2D image composed of splatted primitives [G].

**Data Structures:**
Each octree node stores the children's average value $\mu$ and an error metric $\sigma$ based on the standard deviation of the children's values [S1, unofficial][S2, book].

**Algorithm Steps:**
1. **Tree Construction:** Build the octree from the input volume. For each internal node, compute $\mu$ as the average of its children's values and $\sigma$ as the standard deviation of those values [G][S1, unofficial][S2, book].
2. **Node Selection:** Traverse the octree from the root. For each node, compare its pre-computed error metric $\sigma$ against a user-defined pre-set threshold $\epsilon$ [S1, unofficial][S2, book]. If $\sigma \leq \epsilon$, the node is selected as a leaf for rendering. Otherwise, recurse to its children [S1, unofficial][S2, book].
3. **Primitive Approximation:** When a non-leaf node is selected, it is approximated by a single kernel (splat) [S1, unofficial][S2, book]. The footprint radius $R$ of the splat is set to twice the radius of the kernels of its immediate children ($R_{child}$), i.e., $R = 2 \times R_{child}$ [S1, unofficial][S2, book].
4. **Compositing:** Render the selected splats using standard alpha blending or accumulation techniques [G].

**Parameters:**
* $\epsilon$: Pre-set error threshold for node selection. Typical values are not given in the sources [S1, unofficial][S2, book].

**Edge Cases and Pitfalls:** Not given in the sources [S1, unofficial][S2, book].

## Subtopics

### [Adaptive Hierarchical Splatting with View-Dependent LOD](15498-adaptive-hierarchical-splatting-with-view-dependent-lod.md)
A rendering method for non-uniform volumetric data that uses a hierarchical structure (octree or adaptive tree) to select levels of detail (LOD) based on a screen-space error metric, allowing for interactive navigation and progressive refinement.

## Operating characteristics
* **Time Complexity:** Not given in the sources [S1, unofficial][S2, book].
* **Memory Complexity:** Requires storing the octree structure plus the average and error metric for each node [G].
* **Strengths:** Not given in the sources [S1, unofficial][S2, book].
* **Limitations:** Not given in the sources [S1, unofficial][S2, book].
* **When to use:** Not given in the sources [S1, unofficial][S2, book].
* **When not to use:** Not given in the sources [S1, unofficial][S2, book].

## Background
The method was proposed by Laur and Hanrahan [S1, unofficial][S2, book]. It is based on the idea that non-leaf nodes in an octree can be approximated by kernels with a radius twice that of their children, providing a magnified footprint that represents the aggregate data of the subtree [S1, unofficial][S2, book]. This approach contrasts with other hierarchical methods that use internal nodes only for occlusion culling [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
