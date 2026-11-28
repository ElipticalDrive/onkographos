# Inversion Force Recovery

Also called: Inversion Force Correction, Inversion Handling

## Overview
A method to modify forces of inverted elements so they fall back to an uninverted state, mixing approaches from [17] and [23] [S1, unofficial]. It sets apart from standard FEM force computation by detecting geometric inversion and applying a corrective local frame rotation to restore physical plausibility, whereas standard methods simply compute forces based on the current deformed state [S1, unofficial].

## How it works
**Inputs:**
- Current vertex positions of element $j$.
- Initial rest shape $\mathbf{x}^0$ (equilateral triangle with edge length $\hat{e}$) [S1, unofficial].
- Precomputed stiffness matrix $\mathbf{K}$ [S1, unofficial].
- Rotation matrix $\mathbf{R}_j$ encoding the local frame orientation [S1, unofficial].

**Outputs:**
- Corrected force vector $\mathbf{f}_j$ applied to the vertices of element $j$ [S1, unofficial].

**Data Structures:**
- Element $j$ consists of three vertices.
- Local frame coordinates $(u, v, w)$ for the three vertices [S1, unofficial].
- Displacement vector $\mathbf{d}$ derived from local frame coordinates [S1, unofficial].

**Steps:**
1. **Inversion Detection:** Identify if element $j$ is inverted. This requires one additional evaluation of vertex gradients per inverted element [S1, unofficial].
2. **Local Frame Selection:** If inverted, identify the inverted vertex as the one with the smallest height in the local frame [S1, unofficial]. This aligns with the approach in [17] [S1, unofficial].
3. **Displacement Computation:** Compute displacements for the inverted element using the modified local frame coordinates $(u, v, w)$ [S1, unofficial]. The displacement is defined as $(\mathbf{x} - \mathbf{x}_0)$ [S1, unofficial].
4. **Force Calculation:** Compute forces classically using the standard FEM formula (referred to as formula (3) in the source) with these displacements [S1, unofficial]. The formula is:
   $$ \mathbf{f}_j(\mathbf{x}) = \mathbf{R}_j \mathbf{K} (\mathbf{R}_j^T \mathbf{x} - \mathbf{x}^0) $$
   where $\mathbf{K}$ and $\mathbf{x}^0$ are constant for all elements [S1, unofficial].

**Parameters:**
- $\hat{e}$: Edge length of the equilateral rest shape [S1, unofficial].
- $\mathbf{R}_j$: Rotation matrix for element $j$ [S1, unofficial].

**Pitfalls:**
- Inversion detection requires one additional evaluation of the vertex gradients per inverted element [S1, unofficial].
- The method assumes the third vertex is assimilated as the inverted one based on height [S1, unofficial].

## Subtopics

### Inversion Prevention
A set of criteria to avoid inversions before they occur, ensuring elements are stretched most of the time [S1, unofficial].
- **Step:** Set subdivision threshold $e_{max} = 2$ and collapse threshold $e_{min} = 1$ [S1, unofficial]. This ensures the "skin" is always stretched and never makes wrinkles that could lead to inversions [S1, unofficial].
- **Step:** Avoid collapsing an edge if doing so creates an inversion [S1, unofficial].
- **Step:** Forbid subdivision of edges belonging to an inverted triangle [S1, unofficial].
- **Choice:** Use this when the simulation topology allows for dynamic mesh refinement and collapse, and the goal is to prevent the need for force recovery entirely [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Adds one additional evaluation of vertex gradients per inverted element [S1, unofficial]. Otherwise, force computation does not require more computations than in the case of a non-inverted element [S1, unofficial].
- **Memory Complexity:** No additional memory structures required beyond standard FEM storage [S1, unofficial].
- **Strengths:** Fast recovery; mixes elegant and fast approaches [S1, unofficial].
- **Limitations:** Does not prevent inversions, only recovers from them [S1, unofficial].

## Background
Originated in the context of dynamic triangulation of implicit surfaces, specifically for handling topology changes in FEM-based simulation [S1, unofficial]. It addresses the limitation of mass-spring systems which are worse at avoiding inversions, as FEM forces increase significantly when a triangle is compressed and approaches an inversion state [S1, unofficial].

## Sources
[S1] Dynamic Triangulation of Implicit Surfaces: towards the handling of topology changes, https://inria.hal.science/inria-00132537, fetched 2026-10-26 (unofficial)
