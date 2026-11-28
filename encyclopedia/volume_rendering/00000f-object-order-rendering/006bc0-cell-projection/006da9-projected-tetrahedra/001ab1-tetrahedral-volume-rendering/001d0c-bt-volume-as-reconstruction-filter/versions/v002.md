# BT Volume as Reconstruction Filter

Also called: Convolution using BT Volumes

## Overview
This method represents the result of convolving an arbitrary scalar-valued volumetric function with a reconstruction filter as a BT Volume, provided the filter itself is a BT Volume sharing a similar $\kappa$ function [S1]. It solves the problem of generating high-quality, continuous volumetric data from discrete samples by exploiting the closure of BT Volumes under addition [S1]. Unlike standard convolution which sums kernel functions, this method ensures that the discrete convolution produces a single BT Volume where each tetrahedron is covered by one tetrahedron from each piecewise BT kernel in the sum [S1].

## How it works
**Inputs:**
- A scalar volume $A$ defined on a regular grid with voxel indices $(a,b,c) \in C$ [S1].
- A reconstruction filter $G$ represented as a BT Volume [S1].
- The filter voxels must coincide with the voxels in the scalar volume [S1].

**Outputs:**
- A resulting BT Volume representing the convolved data [S1].

**Data Structures:**
- The BT Volume consists of a grid of cubic Bezier Tetrahedra (BT) [S1].
- Each BT is defined by weights $w_{ijkl}$ for $i+j+k+l=3$ and a $\kappa$ function that maps a local coordinate to a tetrahedron index [S1].
- The $\kappa$ function determines which tetrahedron within a voxel contains a query point [S1].

**Steps:**

1.  **Voxel Inversion:** Invert each voxel in the scalar volume $A$ in place before evaluation [S1]. This is necessary because the order of voxels is reversed in the convolution filter [S1]. This step ensures continuity between voxels in the result [S1]. Note that this inversion causes the $\kappa$ function of the resulting BT Volume to be an inverted version of the filter's $\kappa$ function [S1].

2.  **Filter Evaluation Setup:** For a query point $\mathbf{X}$, determine the tetrahedron index $h = \kappa(\mathbf{X} - \lfloor \mathbf{X} \rfloor)$ [S1]. Since $\kappa$ is defined not to have overlap, we only loop for tetrahedra with this specific index [S1].

3.  **Convolution Calculation:** Evaluate the convolution using the rearranged equation derived from substituting the BT Volume representation of the filter into the discrete convolution formula [S1]. The specific tetrahedron $\tau$ in the reconstruction filter centered at $(a,b,c)$ is given by:
    $$ \tau = BT_{[\mathbf{X}]-(a,b,c)}(\kappa(\mathbf{X}-[\mathbf{X}])) \quad (11) $$
    
    The filter value $G(\mathbf{X} - (a, b, c))$ is evaluated as:
    $$ G(\mathbf{X} - (a, b, c)) = \sum_{i+j+k+l=3} w_{ijkl} \binom{3}{ijkl} r^i s^j t^k u^l \quad (12) $$
    where $r = (r, s, t, u) = (\mathbf{X} - (a, b, c)) \cdot (\mathbf{T})^{-1}$ and $\mathbf{T}$ is defined from the bounding tetrahedron of $\tau$ [S1].

4.  **Weight Aggregation:** Substitute the filter evaluation into the convolution sum and rearrange to move terms independent of the outer summation out of the loop [S1]. The indices $i, j, k, l$ and the values $r, s, t, u$ do not depend on the outer summation over $(a,b,c)$ [S1]. The resulting equation is:
    $$ \sum_{\substack{i+j+k+l=3}} \binom{3}{ijkl} r^i s^j t^k u^l \sum_{(a,b,c) \in C} (w_{ijkl} \cdot A(a,b,c)) \quad (13) $$
    
    By inspection, this is the equation of a BT Volume where the weight component for each basis function is equal to the inner summation:
    $$ W_{ijkl} = \sum_{(a,b,c) \in C} (w_{ijkl} \cdot A(a,b,c)) $$
    [S1].

**Parameters:**
- Filter Voxels: Must coincide with voxels in the scalar volume [S1].

**Edge Cases and Pitfalls:**
- **Continuity:** Failure to invert voxels in place before evaluation breaks continuity between voxels [S1].
- **$\kappa$ Function:** The $\kappa$ function of the resulting BT Volume will be an inverted version of the filter $\kappa$ due to the voxel inversion step [S1].
- **Overlap:** The derivation assumes $\kappa$ does not have overlap, allowing the summation to be restricted to tetrahedra with a specific index [S1].

## Subtopics

### [BT Volume Construction](1421-bt-volume-construction.md)
A method to approximate an arbitrary scalar-valued volumetric function as a BT Volume using a least-squares fit. It differs from direct discretization by projecting the target function onto the polynomial basis of the BT Volume, ensuring $C^0$ continuity across tetrahedral boundaries. It serves as the primary generation technique for the data consumed by [BT Volume Isosurface Rendering](1427-bt-volume-isosurface-rendering.md).

### [BT Volume Isosurface Rendering](1427-bt-volume-isosurface-rendering.md)
BT Volume Isosurface Rendering is a real-time technique for rendering exact isosurfaces from volumetric data by representing the volume as a grid of cubic Bezier Tetrahedra (BT). It solves the problem of low-quality, faceted surfaces produced by linear reconstruction methods like Marching Cubes, while avoiding the computational cost of generating and rendering explicit triangle meshes. Unlike its sibling [BT Volume Construction](1421-bt-volume-construction.md), which focuses on generating the BT data structure from a reconstruction filter, this method focuses on the rendering pipeline: performing local ray tracing within each tetrahedron inside a GPU pixel shader to find exact intersections with arbitrary isosurface levels.

## Operating characteristics
- **Time Complexity:** The method allows for real-time rendering of high-quality convolved volumes [S1]. The convolution itself involves summing contributions from overlapping BT kernels, but the closure property ensures the result is a single BT Volume, avoiding explicit mesh generation [S1].
- **Memory Complexity:** Not given in the sources [S1].
- **Strengths:** Represents a volume convolved with a BT filter as a BT Volume, enabling real-time rendering [S1]. Resolution independent [S1].
- **Limitations:** Requires the reconstruction filter to be representable as a BT Volume sharing a similar $\kappa$ function [S1].
- **When to use:** When high-quality, continuous volumetric data is needed from discrete samples and real-time rendering is required [S1].
- **When not to use:** When the filter cannot be represented as a BT Volume or when the $\kappa$ function overlap assumptions are violated [S1].

## Background
This method is described in Section 4.2.2 of the source paper [S1]. It builds on the least-squares approximation method described in Section 4.2.1 [S1]. The approach was developed to allow the representation of volume convolved with a BT filter as a BT Volume, leveraging the fact that BT volumes can be rendered in real time [S1].

## Sources
[S1] Interactive volume isosurface rendering using BT volumes, https://doi.org/10.1145/1342250.1342257, fetched 2026-10-30
