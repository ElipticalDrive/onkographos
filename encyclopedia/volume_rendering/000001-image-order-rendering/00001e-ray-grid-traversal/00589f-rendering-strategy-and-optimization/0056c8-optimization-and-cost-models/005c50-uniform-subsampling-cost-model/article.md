# Uniform Subsampling Cost Model
Also called: Model 3
## Overview
The Uniform Subsampling Cost Model estimates the total latency for remote visualization tasks where the original volumetric data is subsampled onto a uniform grid, and the reduced grid data is transmitted to a local workstation for isosurface generation [S1]. It differs from its siblings by assuming a fixed grid topology with no additional spatial or coordinate overhead beyond the grid parameters, unlike the Multiresolution Subsampling Cost Model which accounts for variable vertex counts and connectivity in octree structures [S1]. It builds on [Ray-grid traversal](17-ray-grid-traversal.md) by optimizing the parameters under which the rendering strategy runs, rather than altering the traversal logic itself [S1].

## How it works
The model calculates the total cost ($MU$) as the sum of four distinct terms: computational cost for data reduction, computational cost for isosurface generation, transmission cost for the subsampled data, and network latency cost [S1].

**Inputs and Outputs**
*   **Inputs:**
    *   $N$: Total number of elements in the original dataset [S1].
    *   $I$: Number of isosurfaces to be computed and rendered [S1].
    *   $P$: Number of processors available for parallel computation [S1].
    *   $R$: Network bandwidth in Megabits per second (Mbs) [S1].
    *   $L$: Network latency in seconds (s) [S1].
    *   $C_{Ru}$: Uniform grid subsampling cost (system-dependent computational constant) [S1].
    *   $C_{Iu}$: Uniform grid isosurface cost per element (system-dependent computational constant) [S1].
    *   $N_R$: Number of subsampled elements in the reduced uniform grid [S1].
*   **Outputs:**
    *   $MU$: Total cost (latency) for a single request [S1].

**Data Structures**
The model assumes a uniform grid structure [S1]. The data transmitted consists of:
1.  **Grid Parameters:** Origin, grid spacing, and problem size in each of the three dimensions [S1].
2.  **Scalar Data:** Scalar value and error for each of the $N_R$ elements [S1].
3.  **Bit Depth:** All scalar and integer values are assumed to be 32-bit [S1].

**Formula**
The total cost $MU$ is defined as [S1]:

$$ MU = \frac{C_{Ru}}{P} + N_R C_{Iu} I + \frac{32 \cdot (9 + 2N_R)}{10^6 R} + 2 L $$

**Symbol Definitions**
*   $C_{Ru}$: Uniform Grid Subsampling Cost. Represents the computational effort to reduce the original data to the subsampled grid [S1].
*   $P$: Number of processors. The subsampling cost scales linearly with the number of processors [S1].
*   $N_R$: Number of subsampled elements. This parameter defines the size of the reduced dataset [S1].
*   $C_{Iu}$: Uniform Grid Isosurface Cost/Element. The computational cost to generate isosurface geometry for one element [S1].
*   $I$: Number of isosurfaces. The model assumes isosurfaces are computed in parallel [S1].
*   $R$: Network bandwidth in Mbs. Used to calculate transmission time [S1].
*   $L$: Network latency in seconds. Represents the round-trip time overhead [S1].
*   $32$: Bit width of each value (scalar or integer) [S1].
*   $9$: Number of grid parameter values (origin x,y,z; spacing x,y,z; size x,y,z) [S1].
*   $2$: Number of values per element (scalar data and error) [S1].
*   $10^6$: Conversion factor from bits to Megabits [S1].

**Pseudocode**
The following Python-like pseudocode implements the calculation of the total cost $MU$ [S1]. It assumes all inputs are provided as floating-point or integer values [S1].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def calculate_uniform_subsampling_cost(
    C_Ru,  # Uniform grid subsampling cost [S1]
    P,     # Number of processors [S1]
    N_R,   # Number of subsampled elements [S1]
    C_Iu,  # Uniform grid isosurface cost per element [S1]
    I,     # Number of isosurfaces [S1]
    R,     # Network bandwidth in Mbs [S1]
    L      # Network latency in seconds [S1]
):
    # Term 1: Computational cost for data reduction on the reduced uniform grid [S1]
    term_reduction = C_Ru / P

    # Term 2: Computational cost for isosurface generation on the reduced uniform grid [S1]
    term_generation = N_R * C_Iu * I

    # Term 3: Transmission cost for a single request [S1]
    # Data transmitted: 9 grid parameters + 2 values (scalar + error) per element [S1]
    # All values are 32-bit integers/floats [S1]
    total_bits = 32 * (9 + 2 * N_R)
    transmission_cost = total_bits / (10**6 * R)

    # Term 4: Latency cost for a single request [S1]
    latency_cost = 2 * L

    # Total cost [S1]
    MU = term_reduction + term_generation + transmission_cost + latency_cost

    return MU
```

**Edge Cases and Pitfalls**
*   **No Additional Spatial Information:** The model explicitly states that no additional spatial or coordinate information is required beyond the grid structure parameters (origin, spacing, size) [S1]. This simplifies the transmission cost compared to multiresolution methods [S1].
*   **Parallel Assumption:** The model assumes that isosurface computation is performed in parallel across processors [S1]. If the hardware does not support this parallelism, the computational cost terms may not scale as expected [S1].
*   **Fixed Bit Depth:** The model assumes a fixed 32-bit precision for all transmitted values [S1]. Using different precisions would alter the transmission cost term [S1].
*   **Uniform Grid Topology:** The model is only valid for uniform grids [S1]. It does not account for the variable connectivity and vertex sharing found in non-uniform or octree-based representations [S1].

## Subtopics

### [Multiresolution Subsampling Cost Model](17125-multiresolution-subsampling-cost-model.md)
Estimates latency for octree-based data where variable vertex counts and connectivity overhead are transmitted. Choose this when the data structure is non-uniform and spatial coordinates/connections must be sent.

### [Ray Marching Optimization](10233-ray-marching-optimization.md)
Optimizes the ray marching step size and traversal logic [G]. Choose this to reduce the number of samples per ray [G].

### [Hardware-Specific Optimization](10277-hardware-specific-optimization.md)
Optimizes for specific GPU/CPU architectures [G]. Choose this when targeting specific hardware features like SIMD or tensor cores [G].

### [Grid Resolution Cost Model](12776-grid-resolution-cost-model.md)
Optimizes the grid resolution itself [G]. Choose this when the grid resolution is a variable to be tuned for performance [G].

### [Image-based Rendering Cost Model](17116-image-based-rendering-cost-model.md)
Estimates cost for image-based rendering techniques using depth images. Choose this when rendering isosurfaces via depth images rather than geometry.

## Operating characteristics
*   **Time Complexity:** The computational terms ($C_{Ru}/P$ and $N_R C_{Iu} I$) do not depend on the original dataset size $N$, but are linear with respect to the subsampled size $N_R$ and the number of isosurfaces $I$ [S1]. The transmission term is linear with respect to $N_R$ [S1].
*   **Memory Complexity:** The model does not explicitly define memory complexity, but the transmission size is proportional to $N_R$ [S1].
*   **Strengths:** Simple to implement and calculate. Assumes minimal communication overhead for data reduction due to uniform grid partitioning [S1]. No additional spatial or coordinate information is required [S1].
*   **Limitations:** Does not account for the costs associated with loading and processing the resulting data sets on the local graphics workstation [S1]. Assumes ideal parallelism for isosurface generation [S1]. Not suitable for non-uniform data structures where connectivity overhead is significant [S1].
*   **When to use:** When the data can be effectively subsampled onto a uniform grid and the primary bottleneck is network transmission and basic computational costs for isosurface generation [S1].
*   **When not to use:** When the data structure is non-uniform (e.g., octree) and requires transmission of connectivity information, or when the local workstation's processing capabilities are a significant bottleneck not captured by the model [S1].

## Background
The Uniform Subsampling Cost Model, also known as Model 3, was introduced in the context of comparing remote visualization strategies for interactive exploration of large data sets [S1]. It is part of a family of cost models that include Image-based Rendering (Model 1), Multiresolution Subsampling (Model 4), and others [S1]. The model was developed to provide a quantitative comparison of different remote visualization techniques, specifically focusing on the trade-offs between computational cost, transmission cost, and latency [S1].

## Sources
[S1] Comparison of remote visualization strategies for interactive exploration of large data sets, https://digital.library.unt.edu/ark:/67531/metadc710831/?q=multiresolution volume rendering, fetched 2026-10-28
