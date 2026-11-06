# Cancellation Error Reduction via Local Coordinates

Also called: Local Coordinate System for Moment Calculation, Shifted Integration Domain for Moments

## Overview
This method mitigates catastrophic cancellation errors in volume and moment calculations for simplicial decomposition by shifting the integration domain into a local coordinate system near the origin before calculation [S1, unofficial]. It sets itself apart from naive global-coordinate integration by changing the error scaling from cubic to linear with respect to the ratio of the integration domain size to its absolute coordinate position [S1, unofficial].

## How it works
**Inputs:** A voxel $V$ with geometric definition (e.g., clipped simplicial mesh), its center coordinate $x_0$ in the global frame, and the desired moment order.
**Outputs:** The volume $V_{vol}$ and coordinate moments $\int_V x^k dV$ (or higher-order tensor equivalents) computed with reduced floating-point error.

**Data Structures:**
The method operates on the geometric primitives defining the voxel. No specific bit-packing is mandated, but the implementation must support the precision required for the target hardware (single or double). The key data is the local coordinate representation of the voxel vertices.

**Steps:**
1.  **Local Coordinate Transformation:** For each voxel, define a local coordinate system where the voxel center $x_0$ lies at the origin. Transform all vertex coordinates $x$ of the voxel to local coordinates $x' = x - x_0$.
2.  **Local Moment Calculation:** Compute the volume $V$ and the "local" moments $\int_V (x - x_0)^k dV$ using the standard analytic voxelization formulas (e.g., from [Polynomial Field Integration via Coordinate Moments](3791-polynomial-field-integration-via-coordinate-moments.md)) applied to the transformed geometry.
3.  **Global Moment Reconstruction (First Order):** For the first moment in $x$, apply the binomial expansion correction. The global moment is reconstructed by adding the offset correction to the local moment:
    $$ \int_V x \, dV = \int_V (x - x_0) \, dV + x_0 \, V $$
    This step involves only addition, avoiding cancellation errors in the correction term [S1, unofficial].
4.  **Higher-Order Moments:** For higher-order moments, calculate the local moments as if the voxel were centered on the origin. Then, add correction terms for the coordinate offset. These cross-terms involve only addition and thus do not contribute to cancellation errors [S1, unofficial].

**Formulas:**
The fractional error $E$ without coordinate shifting scales as:
$$ E \approx 2^{-p} \left( \frac{\Delta x}{x} \right)^{-3} $$
where $p$ is the number of bits in the significand (53 for double, 24 for single), $\Delta x$ is the size of the integration domain, and $x$ is the absolute coordinate position [S1, unofficial].

After shifting the domain to the local coordinate system, the error scaling becomes:
$$ E \approx 2^{-p} \left( \frac{\Delta x}{x} \right)^{-1} $$
This changes the error scaling from cubic to linear with respect to $x/\Delta x$ [S1, unofficial].

**Parameters:**
*   $p$: Significand bits. Typical values: 24 (single precision), 53 (double precision) [S1, unofficial].

**Pitfalls:**
*   **Volume Error Propagation:** Any numerical error in the calculation of the volume $V$ is scaled by a factor of $x_0$ and propagated into the final result for moments. This is unavoidable [S1, unofficial].
*   **Higher-Order Cross-Terms:** Errors in volume or lower-order moments will similarly scale with $x_0$ when reconstructing higher-order moments via binomial expansion [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_moments_local_coords(voxel_vertices, voxel_center, precision_bits):
    """
    Computes volume and moments for a voxel using local coordinates to reduce cancellation error.
    
    Args:
        voxel_vertices: List of 3D coordinates [x, y, z] in global space.
        voxel_center: 3D coordinate [x0, y0, z0] of the voxel center.
        precision_bits: Number of bits in the significand (24 or 53).
        
    Returns:
        volume: Computed volume.
        moments: Dictionary of moments, e.g., {'m1_x': ..., 'm2_xx': ...}.
    """
    # Step 1: Transform to local coordinates centered at origin
    local_vertices = []
    for v in voxel_vertices:
        local_v = [v[i] - voxel_center[i] for i in range(3)]
        local_vertices.append(local_v)
        
    # Step 2: Compute local volume and local moments
    # Note: The actual analytic integration of the clipped simplicial mesh 
    # is performed here using the local_vertices. 
    # This function represents the standard analytic voxelization routine.
    local_volume = compute_analytic_volume(local_vertices)
    local_moments = compute_analytic_moments(local_vertices)
    
    # Step 3: Reconstruct global moments
    global_volume = local_volume
    
    # First moment reconstruction
    # m1_x_global = integral(x dV) = integral((x-x0) dV) + x0 * volume
    global_moments = {}
    global_moments['m1_x'] = local_moments['m1_x_local'] + voxel_center[0] * global_volume
    global_moments['m1_y'] = local_moments['m1_y_local'] + voxel_center[1] * global_volume
    global_moments['m1_z'] = local_moments['m1_z_local'] + voxel_center[2] * global_volume
    
    # Higher-order moments require binomial expansion
    # Example for m2_xx: integral(x^2 dV) = integral((x-x0)^2 dV) + 2*x0*integral((x-x0)dV) + x0^2*volume
    global_moments['m2_xx'] = local_moments['m2_xx_local'] + \
                              2 * voxel_center[0] * local_moments['m1_x_local'] + \
                              voxel_center[0]**2 * global_volume
                              
    # ... (similar for other higher-order terms)
    
    return global_volume, global_moments
```

## Operating characteristics
*   **Time Complexity:** Adds a constant-time coordinate transformation per vertex and simple arithmetic for moment reconstruction. Negligible overhead compared to the analytic integration itself.
*   **Memory Complexity:** Requires storing local vertex coordinates temporarily. No additional large data structures.
*   **Performance:** Significantly improves accuracy for voxels far from the origin, especially in single precision. The error scaling improves from $O((x/\Delta x)^3)$ to $O(x/\Delta x)$ [S1, unofficial].
*   **Strengths:** Drastically reduces cancellation errors in high-precision requirements or single-precision environments for distant objects.
*   **Limitations:** Does not eliminate error; volume calculation errors are amplified by $x_0$ [S1, unofficial].

## Background
This technique is described in the context of an exact general remeshing scheme for conservative voxelization [S1, unofficial]. It addresses the numerical instability inherent in computing moments for geometric primitives located far from the coordinate origin.

## Sources
[S1] An exact general remeshing scheme applied to conservative voxelization, https://arxiv.org/pdf/1412.4941, fetched 2026-10-26 (unofficial)
