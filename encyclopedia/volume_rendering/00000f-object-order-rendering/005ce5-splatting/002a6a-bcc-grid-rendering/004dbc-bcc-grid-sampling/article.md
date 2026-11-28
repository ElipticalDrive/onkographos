# BCC Grid Sampling

Also called: BCC Lattice Sampling, Body Centered Cartesian

## Overview
BCC Grid Sampling is a 3D sampling method that uses a Body-Centered Cartesian (BCC) lattice to store volume data, requiring fewer sample points than a standard Cubic Cartesian (CC) grid for spherically bandlimited signals [S1, unofficial][S2, book]. It is distinguished from its sibling [FCC Lattice Sampling](13653-fcc-lattice-sampling.md) by its application: BCC is best used with point-based object-order methods employing spherical filters, whereas FCC is optimal when the signal spectrum is spherical [S1, unofficial][S2, book].

## How it works
**Inputs and Outputs:**
*   **Input:** A continuous volumetric signal $f(\mathbf{x})$ assumed to be bandlimited with a spherical frequency spectrum [S1, unofficial][S2, book].
*   **Output:** A discrete set of sample values located at the nodes of a BCC lattice [G].

**Data Structures:**
The BCC lattice is defined as the dual of the Face-Centered Cartesian (FCC) lattice in the spatial domain [S1, unofficial][S2, book]. In the spatial domain, this structure gives rise to two interleaved CC grids [S1, unofficial][S2, book]. The lattice consists of two interpenetrating simple cubic sublattices. The sources describe the sampling interval separation as involving $\sqrt{2}$ and $1/\sqrt{2}$, which dictates the density relationship between the lattices [S1, unofficial][S2, book].

**Numbered Steps:**
1.  Define the spatial bounding box and a base lattice spacing parameter.
2.  Generate the first sublattice by sampling a simple cubic grid at intervals of `base_spacing`.
3.  Generate the second sublattice by sampling a simple cubic grid at intervals of `base_spacing`, offset by half the spacing along each axis.
4.  Discard any generated points that fall outside the bounding box.
5.  During reconstruction, evaluate a radially symmetric filter at the distance between the query point and each sample point.

**Formulas:**
*   **Sample Count Reduction:** $N_{\text{BCC}} = \frac{\sqrt{2}}{2} N_{\text{CC}} \approx 0.71 N_{\text{CC}}$ [S1, unofficial][S2, book].
    *   $N_{\text{BCC}}$: Number of samples required for the BCC grid.
    *   $N_{\text{CC}}$: Number of samples required for an equivalent-resolution CC grid.
    *   $\sqrt{2}/2$: Density ratio derived from the lattice geometry for equivalent spherical bandlimiting [S1, unofficial][S2, book].
*   **4D Extension:** $N_{\text{BCC,4D}} = 0.50 N_{\text{CC,4D}}$ [S1, unofficial][S2, book].

**Parameters:**
*   `base_spacing`: Controls the lattice constant. Typical values are not given in the sources. To achieve the 71% reduction relative to a target CC grid, `base_spacing` is typically scaled by $\sqrt{2}$ relative to the CC grid spacing [G].

**Edge Cases and Pitfalls:**
*   **Aliasing with Trilinear Filters:** Reconstruction of a BCC grid with trilinear filters leads to aliasing [S1, unofficial][S2, book]. This occurs because the trilinear filter's frequency response is not radially symmetric, causing it to include higher spectra during interpolation [S1, unofficial][S2, book].
*   **Boundary Handling:** Points generated near the bounding box edges must be explicitly clamped or discarded to prevent out-of-bounds memory access [G].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def generate_bcc_samples(x_min, x_max, y_min, y_max, z_min, z_max, base_spacing):
    """
    Generates sample points for a BCC lattice within a bounding box.
    The BCC lattice is constructed as two interleaved CC grids.
    """
    samples = []
    step = base_spacing
    half_step = step / 2.0

    # Sublattice 1: Simple cubic grid
    x = x_min
    while x <= x_max:
        y = y_min
        while y <= y_max:
            z = z_min
            while z <= z_max:
                samples.append((x, y, z))
                z += step
            y += step
        x += step

    # Sublattice 2: Offset by (step/2, step/2, step/2)
    x = x_min + half_step
    while x <= x_max:
        y = y_min + half_step
        while y <= y_max:
            z = z_min + half_step
            while z <= z_max:
                samples.append((x, y, z))
                z += step
            y += step
        x += step

    return samples

def reconstruct_bcc(samples, query_point, filter_func):
    """
    Reconstructs the value at query_point using BCC samples.
    Must use a radially symmetric filter to avoid aliasing [S1, unofficial][S2, book].
    """
    value = 0.0
    for sample_pos, sample_val in samples:
        dx = query_point[0] - sample_pos[0]
        dy = query_point[1] - sample_pos[1]
        dz = query_point[2] - sample_pos[2]
        dist = math.sqrt(dx*dx + dy*dy + dz*dz)
        weight = filter_func(dist)
        value += sample_val * weight
    return value
```

## Subtopics

### BCC Lattice Storage
BCC Lattice Storage refers to the specific data organization of the BCC grid, which arises as the dual of the FCC lattice in the spatial domain [S2, book]. It consists of two interleaved CC grids with a sampling interval separation involving $\sqrt{2}$ and $1/\sqrt{2}$ [S2, book]. This structure requires 71% of the samples of a usual cubic Cartesian (CC) grid [S2, book]. For 4D volumes (time-varying), a 4D BCC storage requires only 50% of the 4D CC samples [S2, book]. It is best used in conjunction with point-based object-order methods using spherical (radially symmetric) filters to preserve the spherical shape of the frequency spectrum [S2, book]. Reconstruction with trilinear filters leads to aliasing due to the lack of radial symmetry in the filter's frequency response [S2, book].

## Operating characteristics
**Time and Memory Complexity:**
*   **Memory:** BCC sampling requires approximately 71% of the memory of a CC grid for the same volume resolution [S1, unofficial][S2, book]. In 4D, it requires 50% [S1, unofficial][S2, book].
*   **Time:** BCC rendering took 70% of the time of CC rendering for the Foot dataset [S1, unofficial].

**Measured Performance:**
*   On the Foot dataset, BCC rendering achieved 70% of the time required by CC rendering [S1, unofficial].
*   The UltraVis system, which optimizes cache performance, can render a $256^3$ volume at 10 frames/s [S1, unofficial][S2, book]. While not exclusively BCC, the system highlights that cache management is key to fast volume rendering due to massive data sizes [S1, unofficial][S2, book].

**Strengths and Limitations:**
*   **Strengths:** Significant reduction in sample count (and thus memory and potentially computation) for spherically bandlimited signals [S1, unofficial][S2, book].
*   **Limitations:** Trilinear interpolation causes aliasing because the filter is not radially symmetric [S1, unofficial][S2, book]. It is less common than CC grids [S1, unofficial].

**When to Use:**
*   Use BCC grids with point-based object-order methods that employ spherical (radially symmetric) filters [S1, unofficial][S2, book].
*   Do not use with trilinear filters if aliasing is a concern [S1, unofficial][S2, book].

## Background
The BCC lattice is the dual of the Face-Centered Cartesian (FCC) lattice in the spatial domain [S1, unofficial][S2, book]. The spacing of samples in the BCC lattice is the reciprocal of that in the frequency domain, according to the Fourier scaling theorem [S1, unofficial][S2, book]. The concept extends to higher dimensions, such as 4D volumes [S1, unofficial][S2, book]. The efficiency of BCC grids is particularly relevant for signals with spherical frequency spectra, which are common in volume generation due to the sampling kernels used [S1, unofficial][S2, book].

## Sources
[S1] Overview of Volume Rendering, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
[S2] PDF Volume Visualization and Volume Graphics, https://www3.cs.stonybrook.edu/~mueller/teaching/cse332/volvisChapter.pdf, fetched 2026-10-28 (book)
