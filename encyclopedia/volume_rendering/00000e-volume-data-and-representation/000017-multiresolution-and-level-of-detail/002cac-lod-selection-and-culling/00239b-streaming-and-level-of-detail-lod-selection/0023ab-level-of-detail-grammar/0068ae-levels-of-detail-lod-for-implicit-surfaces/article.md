# Levels of Detail (LOD) for Implicit Surfaces

Also called: Hierarchical Implicit Surfaces, Neural Network LOD

## Overview
Levels of Detail (LOD) for Implicit Surfaces is a strategy for managing computational complexity in hierarchical implicit modeling by replacing computationally intensive sub-trees with simpler models based on accuracy requirements [S1]. Unlike proxy nodes, which modify the field function $f$ in regions not straddling the surface to preserve the object volume $O$, LOD nodes explicitly change the surface $\partial O$ itself [S1]. This family enables interactive editing and visualization speeds up by an order of magnitude [S1].

## Shared principles
The core mechanism involves inserting LOD nodes into the construction tree $T$ to substitute complex sub-trees with simplified representations [S1]. An implicit surface is mathematically defined as the set of points $p$ in space satisfying $f(p) = 0$:

$$
\partial O = \{p \in \mathbb{R}^3 \mid f(p) = 0\}
$$

While proxy nodes operate on regions $V$ that do not straddle the surface (preserving $O$), LOD nodes operate on regions that partially or entirely intersect the surface, thereby altering $\partial O$ [S1]. The simplification is achieved through progressive simplification of the implicit surface definition [S1].

## Subtopics

### Continuous LOD Nodes
These nodes define a progressive simplification of the implicit surface in regions intersecting the surface [S1]. They replace complex sub-trees with simpler models while maintaining continuity [S1].

### Normal Warping
An affordable method to enhance surface details on a simpler model while maintaining the overall consistency of the tree-based structure [S1]. It allows a simplified geometry to retain high-frequency visual details without the computational cost of evaluating the full complex tree [S1].

## Background
Cani et al. [CH01] pioneered the use of skeletal primitives with levels of detail by adapting the subdivision level of curves and surfaces according to visualization requirements, specifically using convolution surfaces which are computationally expensive [S1]. Barbier et al. [BGA04] improved this by proposing a hierarchical system with automatic LOD management for interactive modeling using the BlobTree formalism [S1]. Their system handled LOD with smooth transitions and tree optimizations, speeding up visualization by an order of magnitude to allow interactive editing [S1]. More recently, TLY* et al. [TLY* 21] explored using a hierarchy of neural networks to learn an implicit surface at different levels of detail, reconstructing input black-box surfaces and allowing the method to work on various underlying representations, including trees, neural networks, or grids [S1].

## Sources
[S1] Accelerating Signed Distance Functions, https://doi.org/10.1111/cgf.70258, fetched 2026-10-26
