# Discrete Ordinates Method

Also called: Discrete Directions Method, Discrete Ordinates Method with Direction Bins, Languénou et al. Method, S-N method

## Overview
The Discrete Ordinates Method (DOM) solves the Radiative Transfer Equation (RTE) by discretizing the continuous solid angle of directions into a finite set of $M$ discrete bins [S1][S2, book][S3, unofficial][S4][S5, unofficial]. Unlike the [Principal-Ordinates Propagation](4121-principal-ordinates-propagation.md) which tracks specific rays, or [Spherical Harmonics and Spherical Gaussians for Radiance Transfer](5599-spherical-harmonics-and-spherical-gaussians-for-radiance-tra.md) which uses global basis functions, DOM approximates the angular intensity distribution using local directional bins [S3, S5, unofficial]. It differs from the [Multiple Scattering Model (Integral Equation)](1177-multiple-scattering-model-integral-equation.md) by avoiding the explicit computation of dense form factors between all voxel pairs, instead using iterative flux propagation [S1][S3, unofficial]. A primary drawback is the "ray effect," where energy propagates in narrow beams along discrete directions, missing regions between them [S1][S2, book][S3, unofficial][S4][S5, unofficial].

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
def discrete_ordinates_method(voxels, phase_matrix, num_directions, max_iterations, convergence_threshold):
    """
    Solves radiative transfer using Discrete Ordinates Method.
    voxels: List of voxel properties (sigma_t, sigma_s, sigma_e, albedo)
    phase_matrix: M x M matrix for scattering between direction bins
    num_directions: M
    max_iterations: Maximum number of scattering bounces
    convergence_threshold: Stop if change in total energy is below this
    """
    N = len(voxels)
    M = num_directions
    
    # Initialize intensities
    # I[i][j] stores intensity at voxel i in direction j
    I = [[0.0 for _ in range(M)] for _ in range(N)]
    
    # Initialize with emission and boundary conditions
    for i in range(N):
        for j in range(M):
            I[i][j] = voxels[i].sigma_e # Simplified initialization
            
    for iteration in range(max_iterations):
        prev_total_energy = sum(sum(row) for row in I)
        
        # Temporary array to accumulate flux received by each voxel from each direction
        flux_received = [[0.0 for _ in range(M)] for _ in range(N)]
        
        # Propagate flux for each direction bin
        for j in range(M):
            # Propagate along direction j
            # In a full implementation, this requires sorting voxels by depth along omega_j
            # and sweeping. Here we approximate the layer-wise propagation described in sources.
            # Note: The source describes distributing to 4 adjacent voxels. 
            # A precise implementation needs a grid traversal algorithm.
            
            # Simplified simulation of the propagation step described in [S3, unofficial][S4][S5, unofficial]:
            # "flux entering each voxel is multiplied by the voxel's transparency 
            # and then distributed to four adjacent voxels, determined by the direction bin"
            
            # Create a copy for simultaneous update
            new_flux = [[0.0 for _ in range(M)] for _ in range(N)]
            
            for i in range(N):
                # Get flux leaving voxel i in direction j
                flux_out = I[i][j]
                
                # Calculate transparency of voxel i
                # T = exp(-sigma_t * voxel_size)
                # Assuming voxel_size = 1 for simplicity
                sigma_t = voxels[i].sigma_t
                transparency = math.exp(-sigma_t)
                
                # Distribute to 4 adjacent voxels based on direction bin j
                # This is a simplification of the geometric projection described in [S3, unofficial][S4][S5, unofficial]
                # In reality, this depends on the specific geometry of the direction bin
                # and the voxel grid.
                adjacent_voxels = get_adjacent_voxels(i, j) 
                
                for k in adjacent_voxels:
                    if 0 <= k < N:
                        # Add contribution to flux_received at voxel k
                        # The source says "distributed to four adjacent voxels"
                        # We assume equal distribution for this simplified model
                        flux_received[k][j] += flux_out * transparency / len(adjacent_voxels)
            
            # After propagation, scatter received flux to all M directions
            for k in range(N):
                for l in range(M):
                    # Scatter flux received in direction j to direction l
                    # Using the phase matrix
                    scattering_coeff = phase_matrix[j][l]
                    I[k][l] += flux_received[k][j] * scattering_coeff

        # Check convergence
        curr_total_energy = sum(sum(row) for row in I)
        if abs(curr_total_energy - prev_total_energy) < convergence_threshold:
            break
            
    # Final gathering pass
    # Render pixels by integrating I(X, omega) along viewing rays
    # This step is not detailed in the sources' algorithmic steps but mentioned as a final pass [S3, unofficial][S4][S5, unofficial]
    return I
```

**Parameters:**
*   $M$: Number of direction bins. Typical values include 6, 26, or 96 (from $4 \times 4 \times 4$ cube faces) [S1][S2, book][S3, unofficial][S4][S5, unofficial].
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
The Discrete Ordinates Method originates from neutron transport theory, where it is also known as the S-N method [S5, unofficial]. It was adapted for volume rendering by Patmore [65] for local solutions and Languénou et al. [49, 48] for progressive refinement [S1][S2, book]. Max [55] further developed the method for cloud rendering, introducing the layer-wise propagation approximation to avoid computing form factors [S1][S3, unofficial][S4][S5, unofficial]. Cerezo and Serón [14] extended the method to include internal objects, sources, and fluorescence [S1].

## Sources
[S1] A survey on participating media rendering techniques, https://doi.org/10.1007/s00371-005-0287-1, fetched 2026-10-26
[S2] Global Illumination Techniques for the Simulation of Participating Media, https://inria.hal.science/inria-00510096v1, fetched 2026-10-26 (book)
[S3] [PDF] Optical Models for Direct Volume Rendering - Duke Computer Science, https://courses.cs.duke.edu/spring03/cps296.8/papers/max95opticalModelsForDirectVolumeRendering.pdf, fetched 2026-10-26 (unofficial)
[S4] Optical models for direct volume rendering (PDF) - Academia.edu, https://www.academia.edu/49467168/Optical_models_for_direct_volume_rendering, fetched 2026-10-27
[S5] Mathematical models for volume rendering and neutron transport, https://digital.library.unt.edu/ark:/67531/metadc1274949/?q=volume rendering equation, fetched 2026-10-28 (unofficial)
