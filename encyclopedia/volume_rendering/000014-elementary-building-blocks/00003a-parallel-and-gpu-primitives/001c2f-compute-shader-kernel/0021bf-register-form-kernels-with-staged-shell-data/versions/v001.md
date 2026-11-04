# Register-form kernels with staged shell data

Also called: Staged shell data optimization

## Overview
This method optimizes GPU compute shader kernels for volume rendering by staging shell metadata and coefficient data into groupshared memory, thereby eliminating dependent global memory loads during the inner loop evaluation. It builds on [Compute Shader Kernel](1202-compute-shader-kernel.md) by restructuring data access patterns to improve occupancy and reduce latency for atomic orbital (AO) evaluations.

## How it works
**Inputs:** A list of Gaussian-type orbital (GTO) shells, each defined by centre, cutoff, angular momentum $l$, AO count, and primitive coefficients. A set of evaluation points (vertices or corners).
**Outputs:** Evaluated function values and gradients at the points.

**Data Structures:**
*   **Chunk:** A subset of shells packed to at most 256 primitives [S1, S2].
*   **Groupshared Memory:** Stores staged shell data (centre, cutoff, $l$, AO count, primitives) and rows of coefficient matrix $C$ for the current chunk [S1, S2].
*   **Register Form:** Vertex values are kept in registers during computation [S1, unofficial].

**Steps:**
1.  **Pack Chunks:** Group shells into chunks containing at most 256 primitives [S1, S2]. Shells exceeding 64 primitives are refused by the set [S1, S2].
2.  **Stage Data:** Load shell metadata and rows of $C$ into groupshared memory for the current chunk [S1, S2].
3.  **Filter Shells:** For each evaluation point, skip shells that are out of reach in all three dimensions (based on cutoff) [S1, S2].
4.  **Evaluate:** Execute the inner loop using the staged groupshared memory. This avoids dependent global loads per shell [S1, S2].
5.  **Accumulate:** Accumulate contributions from primitives to the final value/gradient [S1, S2].

**Parameters:**
*   Maximum primitives per chunk: 256 [S1, S2].
*   Maximum primitives per shell: 64 (shells exceeding this are refused) [S1, S2].

**Pitfalls:**
*   Shells with more than 64 primitives are refused [S1, S2].
*   Precision issues may arise if not handled; however, specific precision guards are not detailed in the sources [G].

## Subtopics

### Cartesian Monomials Optimization
This variant optimizes the evaluation of Cartesian monomials by computing them at once per angular momentum $l$ in `md_gto`'s order, with AO normalization folded into the staged rows of $C$, which are read as `float4` [S1, unofficial]. A single function accumulates the shell's polynomial and multiplies by the radial part once [S1, unofficial]. The groupshared rows are sized per $F$ (number of channels), ensuring that narrow kernels maintain high occupancy [S1, unofficial].

## Operating characteristics
*   **Performance:** Eliminates dependent global loads in the inner loop, improving throughput [S1, S2].
*   **Memory:** Uses groupshared memory for staging, reducing global memory bandwidth pressure [S1, S2].
*   **Occupancy:** Row sizing per $F$ helps maintain occupancy for narrow kernels [S1, unofficial].
*   **Limitations:** Shells with >64 primitives are unsupported [S1, S2].

## Background
This optimization is part of the `md_gto_iso_gpu` implementation in `mdlib`, focusing on efficient GPU isosurface extraction [S1, S2]. It addresses the bottleneck of loading shell data from global memory for every evaluation point [S1, S2].

## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
