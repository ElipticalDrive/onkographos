# Implicit Scene Representations

Also called: Neural Implicit Surfaces (SDF), SDF-based Implicit Representations

## Overview
Implicit Scene Representations model 3D geometry by parameterizing a Signed Distance Function (SDF) using a neural network or grid, mapping any 3D point to its distance from the nearest surface [S1]. Unlike volumetric NeRF representations that suffer from "floaters" in empty space, this method explicitly defines geometry as a zero-level set, yielding watertight surfaces [S1]. It builds on [Implicit methods (SDF-based)](12421-implicit-methods-sdf-based.md) by specifying the functional form (SDF) and common parameterization strategies.

## How it works
**Inputs:** 3D coordinates $\mathbf{x} \in \mathbb{R}^3$; learnable parameters $\theta$ (weights and biases of the network or grid values).
**Outputs:** Scalar SDF value $s \in \mathbb{R}$.

**Data Structures:**
The SDF function $f$ is parameterized by $\theta$. Common implementations include:
1.  **Dense Grid:** A 3D volume $G_\theta$ with resolution $R_H \times R_W \times R_D$. Each cell stores a learnable SDF value [S1].
2.  **MLP:** A multi-layer perceptron with weights $\theta$ [S1].
3.  **Hybrid:** An MLP combined with feature grids (single- or multi-resolution) [S1].

**Steps:**
1.  **Query:** For a given 3D point $\mathbf{x}$, retrieve or compute the SDF value $s$.
    *   If using a **Dense Grid**, perform interpolation (e.g., trilinear) on $G_\theta$ at location $\mathbf{x}$ [S1]:
        $$ \hat{s} = \text{interp}(\mathbf{x}, G_\theta) $$
    *   If using an **MLP**, pass $\mathbf{x}$ through the network defined by $\theta$ [S1].
2.  **Output:** Return $s$. The surface is defined where $s = 0$ [S1].

**Formulas:**
The SDF is defined as:
$$ f : \mathbb{R}^3 \rightarrow \mathbb{R} $$
$$ \mathbf{x} \mapsto s = \text{SDF}(\mathbf{x}) \quad (1) $$
where $\mathbf{x}$ is the 3D point and $s$ is the distance to the closest surface [S1].

**Parameters:**
*   $\theta$: Learnable parameters. For grids, these are the values in $G_\theta$. For MLPs, these are weights and biases [S1].
*   Resolution $R_H \times R_W \times R_D$: Defines the granularity of the dense grid representation [S1].

**Edge Cases and Pitfalls:**
*   **Interpolation:** When using dense grids, the choice of interpolation (e.g., trilinear) affects smoothness and accuracy [S1].
*   **Degenerate Input:** Points far from any surface may yield large $s$ values, potentially causing numerical issues in downstream rendering or marching cubes if not clamped [G].
*   **Boundary Conditions:** Grid-based methods require handling points outside the defined volume bounds [G].

## Subtopics

### [Neural Signed Distance Function (Neural SDF)](21608-neural-signed-distance-function-neural-sdf.md)
Explicitly models geometry as a zero-level set for watertight surfaces, avoiding floaters common in volumetric NeRFs.

## Operating characteristics
*   **Time Complexity:** Depends on parameterization. Grid lookup is $O(1)$ with interpolation cost; MLP evaluation is linear in network depth/width [S1].
*   **Memory Complexity:** Dense grids scale with resolution $R_H \times R_W \times R_D$; MLPs scale with number of parameters in $\theta$ [S1].
*   **Strengths:** High-quality, watertight surfaces; arbitrary resolution via continuous function [S1].
*   **Limitations:** "Floaters" are avoided, but training can be sensitive to initialization and loss functions [S1].
*   **When to use:** When explicit, clean geometry is required.
*   **When not to use:** When texture/detail is more important than precise geometry, or when memory for dense grids is prohibitive [S1].

## Background
Originated in works exploring monocular geometric cues for neural implicit surface reconstruction, such as MonoSDF [S1]. It represents a shift from volumetric radiance fields to explicit surface representations using SDFs [S1].

## Sources
[S1] MonoSDF: Exploring Monocular Geometric Cues for Neural Implicit Surface Reconstruction, https://doi.org/10.52202/068431-1814, fetched 2026-10-26
