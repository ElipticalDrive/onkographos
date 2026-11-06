# Register-form kernels with staged shell data

Also called: Staged shell data optimization

## Overview
This method optimizes GPU compute shader kernels for atomic orbital (AO) evaluations by staging shell metadata and coefficient data into groupshared memory, thereby eliminating dependent global memory loads during the inner loop evaluation [S1, unofficial]. It builds on [Compute Shader Kernel](1202-compute-shader-kernel.md) by restructuring data access patterns to improve occupancy and reduce latency for AO evaluations [S1, unofficial].

## How it works
**Inputs:** A list of Gaussian-type orbital (GTO) shells, each defined by centre, cutoff, angular momentum $l$, AO count, and primitive coefficients [G]. A set of evaluation points (vertices or corners) [G].
**Outputs:** Evaluated function values and gradients at the points [G].
**Data Structures:** Exact layouts and bit packing are not given in the sources. Groupshared memory stores staged shell data (centre, cutoff, $l$, AO count, primitives) and rows of coefficient matrix $C$ for the current chunk [S1, unofficial]. Vertex values are kept in registers during computation [S1, unofficial].
**Steps:**
1. **Pack Chunks:** Group shells into chunks containing at most 256 primitives [S1, unofficial].
2. **Stage Data:** Load shell metadata and rows of $C$ into groupshared memory for the current chunk [S1, unofficial].
3. **Filter Shells:** For each evaluation point, skip shells that are out of reach in all three dimensions [S1, unofficial].
4. **Evaluate:** Execute the inner loop using the staged groupshared memory, avoiding dependent global loads per shell [S1, unofficial].
**Formulas:** Not given in the sources.
**Parameters:** Maximum primitives per chunk: 256 [S1, unofficial]. Maximum primitives per shell: 64 (shells exceeding this are refused) [S1, unofficial].
**Pitfalls:** Shells with more than 64 primitives are refused by the set [S1, unofficial]. Precision issues may arise if not handled; however, specific precision guards are not detailed in the sources [G].

## Subtopics

### Cartesian Monomials Optimization
This variant optimizes the evaluation of Cartesian monomials by computing them at once per angular momentum $l$ in `md_gto`'s order, with AO normalization folded into the staged rows of $C$, which are read as `float4` [S1, unofficial]. A single function accumulates the shell's polynomial and multiplies by the radial part once [S1, unofficial]. The groupshared rows are sized per $F$ (number of channels), ensuring that narrow kernels maintain high occupancy [S1, unofficial].

## Operating characteristics
* **Performance:** Eliminates dependent global loads in the inner loop, improving throughput [S1, unofficial].
* **Memory:** Uses groupshared memory for staging, reducing global memory bandwidth pressure [S1, unofficial].
* **Occupancy:** Row sizing per $F$ helps maintain occupancy for narrow kernels [S1, unofficial].
* **Limitations:** Shells with more than 64 primitives are unsupported [S1, unofficial].

## Background
This optimization is part of the `md_gto_iso_gpu` implementation in `mdlib`, focusing on efficient GPU isosurface extraction [S1, unofficial]. It addresses the bottleneck of loading shell data from global memory for every evaluation point [S1, unofficial].

## Sources
[S1] md_gto_iso_gpu: optimized GPU isosurface extraction (non-blocking jobs, flood, one measured path), https://github.com/scanberg/mdlib/pull/15, fetched 2026-10-28 (unofficial)
