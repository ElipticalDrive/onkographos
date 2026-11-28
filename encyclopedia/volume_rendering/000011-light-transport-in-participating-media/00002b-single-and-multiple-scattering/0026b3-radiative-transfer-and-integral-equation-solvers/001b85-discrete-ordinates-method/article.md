# Discrete Ordinates Method

Also called: Discrete Directions Method, Discrete Ordinates Method with Direction Bins, Languénou et al. Method, S-N method

## Overview
The Discrete Ordinates Method (DOM) solves the Radiative Transfer Equation (RTE) by discretizing the continuous solid angle of directions into a finite set of $M$ discrete bins [S1][S2, book][S3, unofficial][S4][S5, unofficial]. Unlike the [Principal-Ordinates Propagation](4121-principal-ordinates-propagation.md) which tracks specific rays, or [Spherical Harmonics and Spherical Gaussians for Radiance Transfer](5599-spherical-harmonics-and-spherical-gaussians-for-radiance-tra.md) which uses global basis functions, DOM approximates the angular intensity distribution using local directional bins [S3, unofficial][S5, unofficial]. It differs from the [Multiple Scattering Model (Integral Equation)](1177-multiple-scattering-model-integral-equation.md) by avoiding the explicit computation of dense form factors between all voxel pairs, instead using iterative flux propagation [S1][S3, unofficial]. A primary drawback is the "ray effect," where energy propagates in narrow beams along discrete directions, missing regions between them [S1][S2, book][S3, unofficial][S4][S5, unofficial].

## How it works
**Inputs:**
*   Voxel grid of size $n \times n \times n$ (total $N = n^3$ voxels) [S3, unofficial][S4][S5, unofficial].
*   Medium properties per voxel: extinction coefficient $\sigma_t$, scattering coefficient $\sigma_s$, albedo $a = \sigma_s / \sigma_t$, emission coefficient $\sigma_e$ [G].
*   Phase function $p(\boldsymbol{\omega}', \boldsymbol{\omega})$ approximated by an $M \times M$ bin-to-bin matrix [S3, unofficial][S4][S5, unofficial].
*   Number of direction bins $M$ [S3, unofficial][S4][S5, unofficial].

**Outputs:**
*   Directional intensity $I(\mathbf{X}_i, \boldsymbol{\omega}_j)$ for each voxel $i$ and direction $j$ [S3, unofficial][S4][S5, unofficial].
*   Pixel radiances via a final gathering pass [S3, unofficial][S4][S5, unofficial].

**Data Structures:**
*   `Intensity[N][M]`: Stores $I(\mathbf{X}_i, \boldsymbol{\omega}_j)$.
*   `FluxReceived[N][M]`: Temporary array for accumulated flux at voxel $i$ from direction $j$ [S3, unofficial][S4][S5, unofficial].
*   `PhaseMatrix[M][M]`: Approximation of the phase function, where entry $(j, l)$ represents scattering from bin $j$ to bin $l$ [S3, unofficial][S4][S5, unofficial].
*   `Transparency`: Precomputed or computed on-the-fly transmittance between voxels [S3, unofficial][S4][S5, unofficial].

**Steps:**
1.  **Discretize Directions:** Choose $M$ directions $\{\boldsymbol{\omega}_1, \dots, \boldsymbol{\omega}_M\}$ to provide optimal Gaussian quadrature for solid angle integrals [S3, unofficial][S4][S5, unofficial]. A common implementation arranges bins on the 96 exterior faces of a $4 \times 4 \times 4$ block of cubes, resulting in unequal solid angles [S3, unofficial][S4][S5, unofficial].
2.  **Initialize:** Set initial intensities based on boundary conditions and internal emission sources [S2, book][S3, unofficial].
3.  **Iterative Propagation (Shooting):** Repeat until convergence:
    a.  For each direction bin $j \in \{1, \dots, M\}$:
        i.  **Layer-wise Propagation:** Propagate flux from all voxels in the grid simultaneously along direction $\boldsymbol{\omega}_j$. For each voxel, multiply the incoming flux by the voxel's transparency and distribute it to the four adjacent voxels determined by the direction bin $\boldsymbol{\omega}_j$ [S3, unofficial][S4][S5, unofficial]. This approximates the form factors $F_{klij}$ without precomputing them [S3, unofficial][S4][S5, unofficial].
        ii. **Accumulation:** Deposit the received flux into `FluxReceived` at the target voxel [S3, unofficial][S4][S5, unofficial].
    b.  **Scattering:** For each voxel $k$ and each incoming direction bin $j$:
        i.  Scatter the accumulated flux in `FluxReceived[k][j]` to all $M$ outgoing direction bins $l$ using the phase matrix: $I_{new}(\mathbf{X}_k, \boldsymbol{\omega}_l) += \text{FluxReceived}[k][j] \times \text{PhaseMatrix}[j][l]$ [S3, unofficial][S4][S5, unofficial].
    c.  **Update:** Update the intensity field with the new scattered values [S1][S2, book].
4.  **Final Gathering:** Perform a rendering pass along viewing rays using the computed intensity distribution $I(\mathbf{X}, \boldsymbol{\omega})$ to compute pixel radiances, providing one final directional scattering bounce [S3, unofficial][S4][S5, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def discrete_ordinates_method(voxels, phase_matrix, num_directions, max_iterations, convergence_threshold, dim):
    """
    Solves radiative transfer using Discrete Ordinates Method.
    voxels: List of voxel properties (sigma_t, sigma_s, sigma_e, size)
    phase_matrix: M x M matrix for scattering between direction bins
    num_directions: M
    max_iterations: Maximum number of scattering bounces
    convergence_threshold: Stop if change in total energy is below this
    dim: Grid dimensions (nx, ny, nz)
    """
    N = len(voxels)
    M = num_directions
    
    # Initialize intensities
    I = [[0.0 for _ in range(M)] for _ in range(N)]
    
    # Initialize with emission and boundary conditions
    for i in range(N):
        for j in range(M):
            I[i][j] = voxels[i].sigma_e 
            
    for iteration in range(max_iterations):
        prev_total_energy = sum(sum(row) for row in I)
        flux_received = [[0.0 for _ in range(M)] for _ in range(N)]
        
        # Propagate flux for each direction bin
        for j in range(M):
            omega = get_direction_vector(j)
            
            # Layer-wise propagation: distribute flux to 4 adjacent voxels
            for i in range(N):
                flux_out = I[i][j]
                sigma_t = voxels[i].sigma_t
                transparency = math.exp(-sigma_t * voxels[i].size)
                
                adjacent_voxels = get_adjacent_voxels(i, dim, omega)
                share = transparency / len(adjacent_voxels)
                
                for k in adjacent_voxels:
                    flux_received[k][j] += flux_out * share
            
            # Scatter received flux to all M directions
            for k in range(N):
                for l in range(M):
                    I[k][l] += flux_received[k][j] * phase_matrix[j][l]
                    
        # Check convergence
        curr_total_energy = sum(sum(row) for row in I)
        if abs(curr_total_energy - prev_total_energy) < convergence_threshold:
            break
            
    return I

def get_direction_vector(bin_index):
    # Returns a representative unit vector for the direction bin
    # Implementation depends on the specific bin arrangement (e.g., 96 faces of 4x4x4 cubes)
    return [0.0, 0.0, 0.0] # [G] Standard direction lookup

def get_adjacent_voxels(idx, dim, omega):
    # Returns the 4 adjacent voxels in the octant defined by omega
    x, y, z = idx // (dim[1]*dim[2]), (idx // dim[2]) % dim[1], idx % dim[2]
    dx = 1 if omega[0] > 0 else -1
    dy = 1 if omega[1] > 0 else -1
    dz = 1 if omega[2] > 0 else -1
    neighbors = []
    # The 4 neighbors in the propagation octant
    candidates = [
        (x, y, z),
        (x + dx, y, z),
        (x, y + dy, z),
        (x, y, z + dz)
    ]
    for nx, ny, nz in candidates:
        if 0 <= nx < dim[0] and 0 <= ny < dim[1] and 0 <= nz < dim[2]:
            neighbors.append(nx * dim[1]*dim[2] + ny * dim[2] + nz)
    return neighbors
```

**Parameters:**
*   $M$: Number of direction bins. Typical values include 6 or 26 directions as used by Patmore [S1][S2, book], or 96 bins arranged on the 96 exterior faces of a $4 \times 4 \times 4$ block of cubes as implemented by Max [S3, unofficial][S4][S5, unofficial].
*   Albedo $a$: Ratio of scattering to extinction. In tests, $a=0.99$ was used [S3, unofficial][S4][S5, unofficial].
*   Convergence: Iterations stop when energy change is below a threshold. In tests, 15 iterations were sufficient for convergence on a $24 \times 24 \times 18$ voxel cloud [S3, unofficial][S4][S5, unofficial].

**Edge Cases and Pitfalls:**
*   **Ray Effects:** The primary pitfall is the "ray effect," where energy is propagated in narrow beams along discrete directions, missing regions between them [S1][S2, book][S3, unofficial][S4][S5, unofficial]. This can be mitigated by spreading intensity into solid angle bins rather than discrete rays [S3, unofficial][S4][S5, unofficial].
*   **Precision:** Computing all form factors $F_{klij}$ explicitly is prohibitively expensive ($O(n^7 + n^6 M)$) [S3, unofficial][S4][S5, unofficial]. The iterative approximation is preferred when iterations are small compared to $N$ [S3, unofficial][S4][S5, unofficial].

## Subtopics

### [Direction Bin Method for Multiple Scattering](1690-direction-bin-method-for-multiple-scattering.md)
The Direction Bin Method computes higher-order (multiple) scattering in volume rendering by discretizing the sphere of directions into bins arranged on the exterior faces of a 3D grid of cubes and solving for flux transfer between these bins. It differs from [Light Propagation for Multiple Anisotropic Volume Scattering](6627-light-propagation-for-multiple-anisotropic-volume-scattering.md) by using a discrete bin-based flux spreading model rather than continuous light transport volumes.

### Discrete Ordinates Extension
An extension of the discrete ordinates method that includes objects and sources inside the participating medium, handles highly anisotropic phase functions, and considers volumetric inelastic processes, particularly fluorescence [S1]. It has been applied to the rendering of underwater scenes where the sea is treated as a participating medium characterized by real experimental medium parametrizations [S1].

### m > 1 Propagation
Handles direction bins where $m > 1$ by maintaining work, edge, and corner arrays only for separations $s$ divisible by $m$, and spreading them for non-divisible $s$ [S6, book]. For $m=2$ in 2-D, weights for cells are computed from integrals of the phase function and segment length, assigning weight in cell $c$ to cell $b$ to reduce spread, and normalizing $a$ and $b$ to sum to 1 [S6, book]. For 3-D $m>1$, Monte Carlo integration is used to determine weights due to complex ray/cell intersection geometry [S6, book]. On odd-numbered steps, special weights are used to propagate flux [S6, book].

## Operating characteristics
**Time Complexity:**
*   Computing all form factors $F_{klij}$ explicitly costs $O(n^7 + n^6 M)$ time [S3, unofficial][S4][S5, unofficial].
*   The iterative approximation propagates flux from all voxels in a layer simultaneously, effectively computing $N^2$ interactions in time $O(N \log N)$ per direction bin [S3, unofficial][S4][S5, unofficial].
*   One iteration through all $M$ shooting bins takes $O(M N \log N + M^2 N) = O(M n^3 \log n + M^2 n^3)$ time [S3, unofficial][S4][S5, unofficial].
*   The scattering step takes $O(MN)$ time per iteration [S3, unofficial][S4][S5, unofficial].
*   Cost per iteration is $O(Mv)$ where $v$ is the number of voxels [S1][S2, book].

**Memory Complexity:**
*   Stores $N \times M$ unknown intensities [S3, unofficial][S4][S5, unofficial].
*   The matrix $F_{klij}$ has at least $N^2 M$ non-zero coefficients if computed explicitly [S3, unofficial][S4][S5, unofficial].

**Measured Performance:**
*   On an SGI Personal Iris 4D/35 with Mips 3000 processor, a $24 \times 24 \times 18$ voxel cloud volume required 15 minutes per iteration for 15 iterations [S3, unofficial][S4][S5, unofficial].
*   Final rendering at $512 \times 384$ resolution took 5 minutes [S3, unofficial][S4][S5, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Avoids the prohibitive cost of computing all form factors explicitly [S3, unofficial][S4][S5, unofficial]. Can handle multiple scattering efficiently with few iterations if albedo is high but flux exits edges [S3, unofficial][S4][S5, unofficial].
*   **Limitations:** Suffers from ray effects [S1][S2, book][S3, unofficial][S4][S5, unofficial]. Computationally expensive for large $N$ and $M$ [S3, unofficial][S4][S5, unofficial].

**When to Use:**
*   Use when multiple scattering is important and ray effects can be mitigated by binning [S3, unofficial][S4][S5, unofficial].
*   Use when the number of iterations required for convergence is small compared to $N$ [S3, unofficial][S4][S5, unofficial].

**When Not to Use:**
*   Avoid when ray effects are unacceptable and cannot be mitigated by binning [S3, unofficial][S4][S5, unofficial].
*   Avoid when $N$ is very large and $M$ is large, leading to high computational cost [S3, unofficial][S4][S5, unofficial].

## Background
Also known as the S-N method, the Discrete Ordinates Method discretizes the continuous solid angle of directions into a finite set of $M$ discrete bins [S1][S2, book][S3, unofficial][S4][S5, unofficial]. It is mathematically equivalent to the P-N method if $M$ directions are properly placed [S3, unofficial][S4][S5, unofficial]. The method was adapted for volume rendering by Patmore [65] for local solutions using 6 or 26 directions [S1][S2, book]. Languénou et al. [49, 48] applied the method to cloud rendering with a progressive refinement approach [S1][S2, book]. Max [55] further developed the method for cloud rendering, introducing the layer-wise propagation approximation to avoid computing form factors [S1][S3, unofficial][S4][S5, unofficial]. Cerezo and Serón [14] extended the method to include internal objects, sources, and fluorescence [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
[S3] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S4] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S5] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
[S6] Efficient Light Propagation for Multiple Anisotropic Volume Scattering, https://doi.org/10.1007/978-3-642-87825-1_7, fetched 2026-10-26 (book)
