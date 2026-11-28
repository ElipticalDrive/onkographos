# Multiresolution Subsampling Cost Model

Also called: Model 4

## Overview
The Multiresolution Subsampling Cost Model estimates the total latency for remote visualization using an octree-based data representation, where a reduced mesh is transmitted to a local workstation for isosurface generation [S1]. It differs from the Uniform Subsampling Cost Model by accounting for the variable vertex counts and connectivity overhead inherent in nonuniform octree structures, rather than assuming a fixed grid topology [S1].

## How it works
**Inputs and Outputs**
*   **Input:** Octree leaf node count ($N_R$), system computational costs ($C_{Rm}, C_{Im}$), network parameters ($P, R, L$), and rendering parameters ($I$) [S1].
*   **Output:** Total cost $MM$ in seconds [S1].

**Data Structures**
The model assumes the transmission of a reduced mesh containing:
1.  Spatial coordinates: 32-bit floats [S1].
2.  Connectivity, scalar, and error data: 32-bit values [S1].

**Formulas**
The total cost $MM$ is the sum of four terms [S1]:

$$
MM = \frac{C_{Rm}}{P} + N_R C_{Im} I + \frac{32 \cdot (22N_R)}{10^6 R} + 2 L
$$

**Symbol Definitions**
*   $MM$: Total cost (seconds) [S1].
*   $C_{Rm}$: Multiresolution subsampling cost (computational cost for data reduction on the nonuniform grid) [S1].
*   $P$: Number of processors [S1].
*   $N_R$: Number of subsampled elements (octant leaves) [S1].
*   $C_{Im}$: Multiresolution isosurface cost per element (computational cost for isosurface generation on the reduced nonuniform grid) [S1].
*   $I$: Number of isosurfaces [S1].
*   $R$: Network bandwidth in Megabits per second (Mbs) [S1].
*   $L$: Network latency in seconds [S1].

**Step-by-Step Calculation**
1.  **Compute Reduction Cost:** Calculate the time to subsample the data on the server side: $\frac{C_{Rm}}{P}$ [S1].
2.  **Compute Isosurface Cost:** Calculate the time to generate isosurfaces on the client side: $N_R \cdot C_{Im} \cdot I$ [S1].
3.  **Compute Transmission Cost:**
    *   Estimate the number of vertices $V$. The model uses an average case assumption: $V = 4 N_R$ [S1].
    *   Calculate bits for spatial coordinates: $3 \text{ coords} \times 4 \text{ vertices/leaf} \times N_R \times 32 \text{ bits} = 384 N_R$ bits [S1].
    *   Calculate bits for connectivity/scalar/error data: $10 \text{ values} \times N_R \times 32 \text{ bits} = 320 N_R$ bits [S1].
    *   Total bits: $384 N_R + 320 N_R = 704 N_R$ bits, which simplifies to $32 \cdot 22 \cdot N_R$ bits [S1].
    *   Convert to Megabits: $\frac{32 \cdot 22 \cdot N_R}{10^6}$ [S1].
    *   Divide by bandwidth $R$: $\frac{32 \cdot 22 \cdot N_R}{10^6 R}$ [S1].
4.  **Compute Latency Cost:** Add the round-trip latency: $2 L$ [S1].
5.  **Sum:** Add all four terms to get $MM$ [S1].

**Pitfalls and Edge Cases**
*   **Vertex Count Assumption:** The number of vertices in an octree mesh is not fixed [S1]. The upper bound is $8 N_R$ (no shared vertices between octants), and the lower bound is $N_R$ (maximally shared/uniform mesh) [S1]. The model explicitly assumes an average case of $4 N_R$ [S1]. Using a different vertex count requires adjusting the transmission term coefficient [S1].
*   **Nonuniform Grid Costs:** Ensure $C_{Rm}$ and $C_{Im}$ are used, not the uniform grid equivalents ($C_{Ru}, C_{Iu}$), as the data structure is an octree [S1].

## Operating characteristics
*   **Complexity:** The computational terms are linear with respect to $N_R$ and $I$ [S1]. The transmission term is linear with respect to $N_R$ and inversely proportional to $R$ [S1].
*   **Strengths:** Accurately models the overhead of transmitting mesh topology and geometry for sparse or complex datasets where uniform subsampling is inefficient [S1].
*   **Limitations:** Relies on the heuristic average of $4 N_R$ vertices [S1]. Performance may deviate if the octree structure is significantly more or less shared than the average case [S1].
*   **When to use:** When visualizing large datasets with significant empty space or complex internal structures using octree-based compression, and the client has sufficient power to generate isosurfaces from the reduced mesh [S1].

## Background
The model was introduced in the paper "Comparison of remote visualization strategies for interactive exploration of large data sets" [S1]. It is part of a series of four models comparing remote visualization strategies, specifically designed for systems using octree data representations [S1]. The work analyzes performance regimes for various values of $N, P, I,$ and $R$ to determine breakeven points between uniform and multiresolution subsampling techniques [S1].

## Sources
[S1] Comparison of remote visualization strategies for interactive exploration of large data sets, https://digital.library.unt.edu/ark:/67531/metadc710831/?q=multiresolution volume rendering, fetched 2026-10-28
