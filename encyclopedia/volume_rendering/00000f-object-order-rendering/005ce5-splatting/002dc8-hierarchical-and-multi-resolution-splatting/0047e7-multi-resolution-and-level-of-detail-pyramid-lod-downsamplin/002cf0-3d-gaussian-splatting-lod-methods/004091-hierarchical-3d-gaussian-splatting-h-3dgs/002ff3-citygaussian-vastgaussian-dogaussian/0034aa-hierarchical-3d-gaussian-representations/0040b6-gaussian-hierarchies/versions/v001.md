# Gaussian Hierarchies
Also called: H-3DGS hierarchy
## Overview
Gaussian Hierarchies enable real-time rendering of ultra-large-scale 3D Gaussian Splatting datasets by organizing Gaussians into a recursive tree structure [S1, unofficial]. It solves the memory and bandwidth bottlenecks of flat Gaussian representations by providing a view-adaptive Level of Detail (LoD) mechanism, allowing the renderer to select a sparse subset of nodes that approximate the scene geometry at varying distances [S1, unofficial]. It differs from flat splatting by introducing spatial hierarchy and from sibling methods like Budgeted-GS by relying on a strict tree-based cut condition rather than global budget optimization.
## How it works
**Inputs:** A set of original 3D Gaussians (leaves), camera position $\mathbf{p}_{\text{cam}}$, and a global LoD threshold $T$.
**Outputs:** A cut set of nodes to render.

**Data Structures:**
The structure is a tree where:
- **Leaves:** Correspond to the original optimized Gaussians.
- **Non-leaf nodes:** Approximate their children. Each node $i$ has a mean $\boldsymbol{\mu}_i$ and a scale metric $s_i^j$.
- **Tree Layout:** Implicit or explicit parent-child pointers.

**Algorithm:**
1. Initialize an empty cut set $\mathcal{C}$.
2. Perform a Breadth-First Search (BFS) starting from the root node.
3. For each node $i$ visited:
   a. Evaluate the cut condition $c_{\text{hier}}(i, \text{cam})$.
   b. If $c_{\text{hier}}(i, \text{cam})$ is true:
      i. Add node $i$ to $\mathcal{C}$.
      ii. Skip processing all children of node $i$.
   c. If false, continue BFS to process children of node $i$.
4. Return $\mathcal{C}$.

**Formulas:**
The cut condition is defined as [S1, unofficial]:
$$c_{\text{hier}}(i, \text{cam}) = \left\| \boldsymbol{\mu}_i - \mathbf{p}_{\text{cam}} \right\|_2 \geq m_d(i)$$
where:
- $\boldsymbol{\mu}_i$ is the mean position of node $i$.
- $\mathbf{p}_{\text{cam}}$ is the camera position.
- $m_d(i)$ is the distance threshold for node $i$, calculated as:
$$m_d(i) = \frac{T}{\max_j s_i^j}$$
- $T$ is a global LoD threshold parameter.
- $s_i^j$ represents the scale components of node $i$.

**Parameters:**
- $T$: Global LoD threshold. Typical values are not explicitly given in the sources, but it controls the granularity of the hierarchy.

**Edge Cases:**
- **Precision:** Distance calculations must handle floating-point precision near the threshold boundary.
- **Boundaries:** The cut set must be proper, meaning no parent or child of an included node is also included [S1, unofficial].
- **Degenerate Input:** If a node has no children (leaf), the condition is evaluated against the leaf's properties.

## Subtopics

### Gaussian Grouping Strategy
This strategy determines how nearby Gaussians are merged into non-leaf nodes. The sources indicate that non-leaf nodes approximate their children, but the specific grouping algorithm (e.g., clustering, spatial hashing) is not detailed in the provided excerpts [S1, unofficial]. It is a prerequisite for building the hierarchy tree.

## Operating characteristics
**Time Complexity:** The BFS traversal is linear with respect to the number of nodes in the hierarchy. The cut condition evaluation is constant time per node.
**Memory Complexity:** The hierarchy structure adds overhead for parent-child pointers and aggregated node properties, but allows for significant reduction in the number of Gaussians rendered per frame by skipping distant subtrees.
**Performance:** Limits typical training to roughly 500,000 Gaussians per GB of GPU memory, imposing strict constraints on detail and extent without hierarchy [S1, unofficial]. With hierarchy, ultra-large scale reconstructions become feasible.
**Strengths:** View-adaptive LoD, efficient memory usage for large scenes.
**Limitations:** Requires hierarchical construction; performance depends on the quality of the cut condition and tree balance.

## Background
Gaussian Hierarchies were introduced in H-3DGS by Kerbl et al. in 2024 [S1, unofficial]. The method addresses the memory limitations of standard 3D Gaussian Splatting, which typically supports only ~500,000 Gaussians per GB of GPU memory [S1, unofficial]. It enables real-time rendering of ultra-large-scale reconstructions by leveraging external memory and hierarchical culling.

## Sources
[S1] A LoD of Gaussians: Unified Training and Rendering for Ultra-Large Scale Reconstruction with External Memory, https://doi.org/10.48550/arXiv.2507.01110, fetched 2026-10-28 (unofficial)
