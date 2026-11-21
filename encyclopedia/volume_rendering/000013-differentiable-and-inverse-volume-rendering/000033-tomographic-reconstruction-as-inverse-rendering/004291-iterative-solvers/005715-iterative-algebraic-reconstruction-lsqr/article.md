# Iterative Algebraic Reconstruction (LSQR)

Also called: 3D Helical CT Reconstruction via LSQR, 3D Helical Cone-Beam CT Image Reconstruction using LSQR, Helical CT LSQR, Iterative Backprojection Reconstruction, LSQR Reconstruction

## Overview
Iterative Algebraic Reconstruction using LSQR solves the linear inverse problem $\mathbf{A}\mathbf{X} = \mathbf{B}$ for 3D helical cone-beam CT, reconstructing a 3D voxel density grid $\mathbf{X}$ from 2D X-ray projection measurements $\mathbf{B}$ [S1, unofficial]. Unlike direct methods such as [Back-projection](2450-back-projection.md) or [FastFDK](3315-fastfdk.md), LSQR iteratively minimizes the residual error by leveraging the adjoint of the forward projection operator, making it suitable for large-scale, sparse systems where explicit matrix storage is infeasible [S1, unofficial]. It distinguishes itself from other iterative solvers like [Iterative Solvers](10781-iterative-solvers.md) by specifically employing the Least Squares QR algorithm to handle the ill-conditioned nature of helical ray-voxel intersection matrices [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Measured projection data $\mathbf{B}$ (sinogram) of size $N_{pa} \times N_{det} \times N_{slice}$, and system geometry parameters (source-to-object distance, source-to-detector distance, detector pitch, helical shift, angular span).
*   **Output:** Reconstructed 3D volume $\mathbf{X}$ of size $N_{vox} \times N_{vox} \times N_{vox}$ (typically $512^3$).

**Data Structures:**
*   **Volume Grid $\mathbf{X}$:** A contiguous 1D array of `float32` values representing the voxel densities, mapped to a 3D grid of resolution $N_{vox} \times N_{vox} \times N_{vox}$. In the reference implementation, $N_{vox} = 512$, occupying ~536.87 MB [S1, unofficial].
*   **Projection Data $\mathbf{B}$:** A contiguous 1D array of `float32` values representing line integrals for each detector pixel at each angle. Size: $N_{pa} \times N_{det} \times N_{slice}$ [S1, unofficial].
*   **Ray Geometry:** Implicitly defined by the helical trajectory. For angle $i$, the source position $(x_1, y_1, z_1)$ and detector ray endpoints are calculated using trigonometric functions of the angular span and helical shift [S1, unofficial].

**Parameters (Reference Values):**
*   $N_{vox}$ (npix): 512
*   $N_{pa}$: 473 (projections)
*   $N_{det}$: 512 (detector columns)
*   $N_{slice}$: 512 (detector rows)
*   Span: $640^\circ$
*   Detector Pitch ($d$): 0.8 mm
*   Source-to-Object Distance ($scob$): 166.641 mm
*   Source-to-Detector Distance ($scd$): 1238.157 mm
*   Helical Shift: 0.136 mm per angle
*   Relaxation Factor: 0.05 (used in backprojection scaling) [S1, unofficial]
*   Max Iterations: 20 [S1, unofficial]

**Algorithm Steps:**
The reconstruction follows an iterative loop where each iteration processes all projection angles to update the volume estimate.

1.  **Initialization:** Initialize the volume buffer $\mathbf{X}$ to zero.
2.  **Iteration Loop:** For $iter = 0$ to $max\_iterations - 1$:
    a.  **Angle Loop:** For each projection angle $i$ from $0$ to $N_{pa} - 1$:
        i.  **Data Transfer:** Copy the projection slice $\mathbf{B}_i$ (size $N_{det} \times N_{slice}$) from host memory to device memory.
        ii. **Adjoint Kernel Launch:** Launch the `adjoint_kernel` CUDA kernel.
            *   **Grid Dimensions:** $\left(\frac{N_{det}+15}{16}, \frac{N_{slice}+15}{16}\right)$
            *   **Block Dimensions:** $(16, 16)$
            *   **Operation:** For each detector element $(j, k)$ at angle $i$, calculate the ray path through the volume. Accumulate the measured projection value $p$ into the intersected voxels. The contribution is scaled by the relaxation factor and the pixel length.
            *   **Ray Calculation:**
                *   Source position: $x_1 = scob \cdot \sin(\theta_i)$, $y_1 = -scob \cdot \cos(\theta_i)$, $z_1 = z_{start} - i \cdot shift$ (where $\theta_i = \frac{span \cdot \pi}{N_{pa} \cdot 180}$).
                *   Detector point: $p_1 = (\frac{N_{det}}{2} - (0.5 + k)) \cdot d$, $p_3 = z_1 + (\frac{N_{slice}}{2} - (j + 0.5)) \cdot d$.
                *   Ray direction components derived from source-to-detector vector.
                *   Intersection with voxel grid boundaries determines the path length and voxel indices $(m_1, m_2, m_3)$.
    b.  **Synchronization:** After all angles are processed, synchronize the device to ensure all atomic updates to $\mathbf{X}$ are complete.
3.  **Normalization:** Multiply every voxel in $\mathbf{X}$ by the normalization factor $\frac{1}{N_{pa}}$ (or $1/473$) to scale the accumulated backprojected values [S1, unofficial].
4.  **Output:** Save $\mathbf{X}$ as a binary file.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def lsqr_helical_ct_reconstruction(projection_data, geometry_params):
    """
    Reconstructs 3D volume from helical CT projections using LSQR-like iterative backprojection.
    Note: The source implementation uses a simplified iterative backprojection with relaxation,
    often referred to as LSQR in the context of this specific codebase, though strictly it 
    performs weighted backprojection iterations.
    """
    npix = geometry_params['npix']  # 512
    Npa = geometry_params['Npa']    # 473
    Ndet = geometry_params['Ndet']  # 512
    Nslice = geometry_params['Nslice'] # 512
    span = geometry_params['span']  # 640
    d = geometry_params['d']        # 0.8
    scob = geometry_params['scob']  # 166.641
    scd = geometry_params['scd']    # 1238.157
    shift = geometry_params['shift'] # 0.136
    max_iterations = 20
    relaxation_factor = 0.05
    
    # Initialize volume buffer to zero
    volume = np.zeros((npix, npix, npix), dtype=np.float32)
    
    # Pre-calculate constants
    pixlen = 80.86 / npix  # Object diameter / npix
    dia = 80.86
    
    for iteration in range(max_iterations):
        for i in range(Npa):
            # Get projection data for angle i
            # Shape: (Nslice, Ndet)
            proj_slice = projection_data[i] 
            
            # Calculate angle theta
            theta = (span * np.pi) / (Npa * 180) * i
            
            # Source position
            x1 = scob * np.sin(theta)
            y1 = -scob * np.cos(theta)
            z1 = 51.63 - i * shift / Npa  # Approximation based on source code logic
            
            # Process each detector element
            for j in range(Nslice):
                for k in range(Ndet):
                    val = proj_slice[j, k]
                    
                    # Skip negative or zero values
                    if val <= 0.0:
                        continue
                        
                    # Detector point coordinates relative to isocenter
                    p1 = (Ndet / 2 - (0.5 + k)) * d
                    p3 = z1 + (Nslice / 2 - (j + 0.5)) * d
                    
                    # Ray vector components
                    # Source to Detector vector
                    # x2, y2 are detector plane coordinates transformed to world space
                    # Note: The source code calculates x2, y2 based on detector offset p1 and distance p2
                    p2 = scd - scob
                    x2 = p1 * np.cos(theta) - p2 * np.sin(theta)
                    y2 = p1 * np.sin(theta) + p2 * np.cos(theta)
                    z2 = p3
                    
                    # Ray intersection with volume bounds
                    # The source code iterates through voxel indices along the ray
                    # Simplified: Calculate ray direction and step through voxels
                    
                    # Direction vector
                    dx = x2 - x1
                    dy = y2 - y1
                    dz = z2 - z1
                    
                    # Avoid division by zero
                    if abs(dy) < 1e-9:
                        continue
                        
                    # Determine voxel indices intersected by the ray
                    # The source code uses a specific mapping for helical geometry
                    # Iterating through the volume along the ray path
                    # For implementation, one would typically use a ray-marching algorithm
                    
                    # Reference implementation logic for voxel indexing:
                    # It calculates intersection points with voxel boundaries
                    # and accumulates the value 'val' into the intersected voxels
                    
                    # Simplified accumulation logic based on source 'adjoint_kernel' description:
                    # For each voxel m1, m2, m3 intersected by the ray:
                    #   volume[m1, m2, m3] += val * relaxation_factor * pixlen
                    
                    # Note: Exact ray-voxel traversal (e.g., DDA or Voxel Grid Algorithm) 
                    # is required here as the source code uses CUDA atomicAdd for concurrent writes.
                    # The following is a conceptual placeholder for the ray traversal:
                    
                    voxels_hit = ray_march_volume(x1, y1, z1, x2, y2, z2, npix, dia, pixlen)
                    
                    for (m1, m2, m3) in voxels_hit:
                        # m1: x-index, m2: y-index, m3: z-index (based on source wt array mapping)
                        # Source mapping: wt[total+t] = m1 * npix * npix + m3 * npix + m2
                        # This implies m1 is X, m3 is Z, m2 is Y in the linear index
                        
                        if 0 <= m1 < npix and 0 <= m2 < npix and 0 <= m3 < npix:
                            volume[m1, m2, m3] += val * relaxation_factor * pixlen

    # Normalize volume
    volume *= (1.0 / Npa)
    
    return volume

def ray_march_volume(x1, y1, z1, x2, y2, z2, npix, dia, pixlen):
    """
    Conceptual ray marching to find intersected voxel indices.
    The source code calculates specific indices m1, m2, m3 based on 
    intersection with the bounding box [-dia/2, dia/2].
    """
    # This function would implement a standard voxel traversal algorithm
    # returning a list of (m1, m2, m3) tuples.
    # The source code's weightgen function calculates these indices explicitly.
    pass
```

**Edge Cases and Pitfalls:**
*   **Division by Zero:** If the ray direction component $y_2 - y_1$ is near zero, the kernel returns immediately to avoid NaNs or infinite loops [S1, unofficial].
*   **Out-of-Bounds Access:** Voxel indices $m_1$ and $m_3$ must be checked against $0$ and $N_{vox}$ before accessing the volume array [S1, unofficial].
*   **Negative Projections:** If the projection value is $\le 0.0$, backprojection is skipped for that ray [S1, unofficial].
*   **Atomic Contention:** On GPU, concurrent writes to the same voxel from different rays require `atomicAdd`. The CPU reference uses a pre-computed sparse weight matrix to avoid this but is slower [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(I \cdot N_{pa} \cdot N_{det} \cdot N_{slice} \cdot L)$, where $I$ is the number of iterations, and $L$ is the average number of voxels intersected per ray. The reference implementation uses 20 iterations [S1, unofficial].
*   **Memory Complexity:** $O(N_{vox} + N_{proj})$. The volume grid ($512^3$ floats) occupies ~536.87 MB, and the projection data ($473 \times 512 \times 512$ floats) occupies ~495.98 MB [S1, unofficial].
*   **Performance:** The CUDA implementation is significantly faster than the CPU reference due to parallel ray processing and atomic operations. Convergence metrics on the reference dataset show MSE dropping from 24.86 to 0.4478 in 10 iterations [S1, unofficial].
*   **Strengths:** Handles large, sparse systems efficiently without storing the full projection matrix $\mathbf{A}$. Suitable for helical geometry where direct inversion is complex.
*   **Limitations:** Requires many iterations for high fidelity. Sensitive to noise in projection data. The "LSQR" label in the source refers to the iterative framework, but the specific implementation uses a weighted backprojection with relaxation rather than the full LSQR linear algebra solver (which would require matrix-vector products $A^T A x$).

## Background
The method originates from a GitHub repository by Siddharthpo03, providing an end-to-end system for 3D Helical Cone-Beam CT reconstruction [S1, unofficial]. It combines ray-casting geometry with an iterative solver approach. The term "LSQR" is used in the project title to denote the iterative least-squares nature of the reconstruction, although the core loop implements a backprojection-based iterative update. The geometry parameters (512x512x512 volume, 473 angles, 640-degree span) are specific to a simulated helical CT dataset.

## Sources
[S1] Siddharthpo03/3D-Helical-CT-LSQR-Reconstruction, https://github.com/Siddharthpo03/3D-Helical-CT-LSQR-Reconstruction, fetched 2026-10-27 (unofficial)
