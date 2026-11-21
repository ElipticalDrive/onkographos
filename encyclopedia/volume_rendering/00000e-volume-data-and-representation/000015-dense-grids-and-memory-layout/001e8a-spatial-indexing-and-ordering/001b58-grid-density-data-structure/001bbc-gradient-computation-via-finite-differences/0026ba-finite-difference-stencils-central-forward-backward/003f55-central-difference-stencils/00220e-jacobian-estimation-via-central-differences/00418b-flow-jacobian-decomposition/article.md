# Flow Jacobian Decomposition
Also called: Strain-Rate Decomposition

## Overview
Flow Jacobian Decomposition splits the 3x3 Jacobian matrix of a displacement field into symmetric and antisymmetric parts to isolate local geometric deformation (strain) from rigid-body rotation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   Input: The 3x3 Jacobian matrix $J$ of the displacement field $\mathbf{u}$, computed via [Jacobian Estimation via Central Differences](2650-jacobian-estimation-via-central-differences.md) [G].
*   Output: The Infinitesimal Strain Tensor $\boldsymbol{\varepsilon}$ and the Infinitesimal Rotation Tensor $\mathbf{r}$ [S1, unofficial].

**Data Structures:**
*   $J$: A $3 \times 3$ matrix where $J_{ij} = \frac{\partial u_i}{\partial x_j}$ [G].
*   $\boldsymbol{\varepsilon}$: A $3 \times 3$ symmetric matrix [S1, unofficial].
*   $\mathbf{r}$: A $3 \times 3$ antisymmetric matrix [S1, unofficial].

**Steps:**
1.  Compute the symmetric part of $J$ to obtain the Infinitesimal Strain Tensor $\boldsymbol{\varepsilon}$ [S1, unofficial].
2.  Compute the antisymmetric part of $J$ to obtain the Infinitesimal Rotation Tensor $\mathbf{r}$ [S1, unofficial].

**Formulas:**
The decomposition is defined as:
$$ J_{ij} = \varepsilon_{ij} + r_{ij} $$
where:
$$ \varepsilon_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right) $$
$$ r_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right) $$
This matches the identity $(\nabla u)_{ij} = \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} + \frac{\partial u_j}{\partial x_i} \right) + \frac{1}{2} \left( \frac{\partial u_i}{\partial x_j} - \frac{\partial u_j}{\partial x_i} \right)$ [S1, unofficial]. Here, $\varepsilon_{ij}$ represents the symmetric strain component, $r_{ij}$ represents the antisymmetric rotation component, $u_i$ is the $i$-th displacement component, and $x_j$ is the $j$-th spatial coordinate [S1, unofficial].

**Parameters with typical values:**
Not given in the sources.

**Edge Cases:**
*   If $J$ is not square or dimensions mismatch, the decomposition is undefined [G].
*   For large deformations where $\|\nabla \mathbf{u}\| \not\ll 1$, this linear approximation becomes inaccurate [S1, unofficial].

## Subtopics

### [Infinitesimal Rotation Tensor](10633-infinitesimal-rotation-tensor.md)
Isolates the rotational component related to the curl of the vector field, distinguishing pure orientation changes from geometric distortion. Choose this when analyzing rigid-body motion within a flow [G].

## Operating characteristics
*   **Time Complexity:** $O(1)$ per voxel, as it involves a fixed number of arithmetic operations on a $3 \times 3$ matrix [G].
*   **Memory Complexity:** $O(1)$ additional storage per voxel for the two resulting tensors [G].
*   **Strengths:** Provides immediate physical insight into local flow behavior (stretching vs. rotation) [G].
*   **Limitations:** Valid only for infinitesimal deformations. Not suitable for large strain scenarios without modification [S1, unofficial].

## Background
Originating from continuum mechanics, this decomposition is standard in fluid dynamics and solid mechanics to analyze displacement gradients [S1, unofficial]. Application to volume rendering visualization is Not given in the sources.

## Sources
[S1] Multi-Field Visualization, https://kluedo.ub.rptu.de/files/2291/_diss.pdf, fetched 2026-10-27 (unofficial)
