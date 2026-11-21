# Motion Field Error (MFE)

## Overview
Motion Field Error (MFE) is a quantitative metric used to evaluate the accuracy of motion modeling in dynamic volume rendering, specifically within deformable Neural Radiance Fields (NeRFs) and particle-based dynamic models [S1]. It measures the average Euclidean-norm difference between two velocity fields over a region of interest, serving as a ground-truth validation for how well a model captures physically relevant dynamics [S1]. Unlike structural metrics such as PSNR or SSIM which assess visual fidelity, MFE isolates the temporal consistency and kinematic accuracy of the scene's motion [S1]. It distinguishes itself from other error metrics by focusing exclusively on the vector field of velocities rather than scalar intensity or color values [S1].

## How it works
**Inputs and Outputs:**
*   **Input 1:** $F_1(x)$, the estimated velocity field from a dynamic rendering model (e.g., TiNeuVox or DAP-NeRF).
*   **Input 2:** $F_2(x)$, the ground-truth velocity field derived from explicit object trajectories or design software [S1].
*   **Input 3:** $V$, the volume of the region of interest (typically the cubic bounding box of the scene) [S1].
*   **Input 4:** $N$, the number of voxels used for spatial approximation [S1].
*   **Input 5:** $\delta t$, the small time increment for velocity approximation [S1].
*   **Input 6:** $t$, the specific time steps for evaluation [S1].
*   **Output:** A scalar value representing the MFE.

**Data Structures:**
The method operates on voxel grids. The region of interest is discretized into $N$ voxels. Each voxel $n$ has a center coordinate $x_n \in \mathbb{R}^3$. The velocity fields $F_1$ and $F_2$ are functions mapping $\mathbb{R}^3 \to \mathbb{R}^3$.

**Parameters:**
*   $N$: Number of voxels. Typical value: $30^3$ (27,000) evenly distributed within the cubic bounding box [S1].
*   $\delta t$: Time increment. Typical value: $0.01$ [S1].
*   $t$: Sampled time steps. Typical values: $[0.1, 0.3, 0.5, 0.7, 0.9]$ [S1].

**Steps:**
1.  **Discretize the Region:** Generate $N$ voxel centers $x_n$ evenly distributed within the cubic bounding box of the scene [S1].
2.  **Compute Ground-Truth Velocity ($F_2$):** Determine the velocity of the object at each voxel center $x_n$ at time $t$. If the voxel is occupied by a moving object, $F_2(x_n)$ is the object's velocity; otherwise, it is zero [S1].
3.  **Compute Estimated Velocity ($F_1$):** Calculate the velocity field predicted by the dynamic model. The method differs based on the model architecture:
    *   *For Deformation-Field Models (e.g., TiNeuVox):* Compute the voxel velocity by finite differencing the deformation field $d_f(x, t)$, which maps canonical 3D locations to deformed locations [S1].
        $$ F_{df}(x) \approx \frac{d_f(x, t) - d_f(x, t + \delta t)}{\delta t} $$
        Zero-out velocities in empty areas (where no object occupies the voxel) to prevent incorrect values from the deformation field [S1].
    *   *For Particle-Based Models (e.g., DAP-NeRF):* Compute the velocity directly from particle motions. Calculate the velocity of each particle $p$ as $v_p = \frac{p_{t+\delta t} - p_t}{\delta t}$ [S1]. Then, $F_1(x_n)$ is the average velocity of all particles affecting the voxel region at time $t$ [S1].
4.  **Calculate Error:** Compute the Euclidean norm difference between $F_1(x_n)$ and $F_2(x_n)$ for each voxel [S1].
5.  **Aggregate:** Average the errors over all $N$ voxels to obtain the MFE [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_mfe(estimated_velocity_field, ground_truth_velocity_field, voxel_centers):
    """
    Computes the Motion Field Error between two velocity fields.
    
    Args:
        estimated_velocity_field: Function or lookup returning F1(x) for any x.
        ground_truth_velocity_field: Function or lookup returning F2(x) for any x.
        voxel_centers: List of N 3D coordinates (x_n).
        
    Returns:
        float: The MFE value.
    """
    total_error = 0.0
    n_voxels = len(voxel_centers)
    
    for x_n in voxel_centers:
        # Get velocity vectors at the voxel center
        f1 = estimated_velocity_field(x_n)
        f2 = ground_truth_velocity_field(x_n)
        
        # Compute Euclidean norm difference
        # f1 and f2 are assumed to be 3D vectors
        diff = [f1[i] - f2[i] for i in range(3)]
        norm_diff = sqrt(diff[0]**2 + diff[1]**2 + diff[2]**2)
        
        total_error += norm_diff
        
    mfe = total_error / n_voxels
    return mfe

def get_tiNeuvox_velocity(deformation_field, x, t, delta_t):
    """
    Computes velocity for TiNeuVox-style models.
    """
    # Get deformation offsets
    d_f_t = deformation_field(x, t)
    d_f_t_plus_dt = deformation_field(x, t + delta_t)
    
    # Finite difference
    velocity = [(d_f_t[i] - d_f_t_plus_dt[i]) / delta_t for i in range(3)]
    
    # Zero-out if empty area (requires occupancy check not detailed in source)
    # [G] Assuming occupancy is determined by density threshold or explicit mask
    if not is_occupied(x):
        velocity = [0.0, 0.0, 0.0]
        
    return velocity

def get_dapnerf_velocity(particles, x, t, delta_t):
    """
    Computes velocity for DAP-NeRF-style models.
    """
    # Find particles affecting voxel x at time t
    # [G] Logic to determine "affecting" particles (e.g., distance threshold)
    affecting_particles = get_affecting_particles(particles, x, t)
    
    if not affecting_particles:
        return [0.0, 0.0, 0.0]
        
    total_velocity = [0.0, 0.0, 0.0]
    count = 0
    
    for p in affecting_particles:
        p_t = p.position_at(t)
        p_t_plus_dt = p.position_at(t + delta_t)
        v_p = [(p_t_plus_dt[i] - p_t[i]) / delta_t for i in range(3)]
        
        for i in range(3):
            total_velocity[i] += v_p[i]
        count += 1
        
    avg_velocity = [v / count for v in total_velocity]
    return avg_velocity
```

**Edge Cases and Pitfalls:**
*   **Empty Areas:** Deformation fields may produce incorrect velocity values in empty space. These must be explicitly zeroed out to avoid inflating the error metric [S1].
*   **Time Increment:** The choice of $\delta t$ affects the accuracy of the finite difference approximation. A value of $0.01$ is used to balance stability and precision [S1].
*   **Voxel Distribution:** The approximation assumes uniform distribution of voxels within the bounding box. Non-uniform sampling may bias the result [S1].

## Operating characteristics
**Time and Memory Complexity:**
The complexity is linear with respect to the number of voxels $N$ and the number of particles (for DAP-NeRF) or deformation field evaluations (for TiNeuVox) per voxel [S1]. The memory overhead is minimal, requiring storage for the voxel centers and the velocity fields [S1].

**Measured Performance:**
On the D-NeRF dataset (Scenes 1-3) at $t=1.0$, TiNeuVox achieved MFE values of 0.0197, 0.0363, and 0.0376, respectively. DAP-NeRF achieved lower errors of 0.0029, 0.0104, and 0.0136 for the same scenes [S1].

**Strengths and Limitations:**
*   **Strengths:** Provides a direct, quantitative measure of motion accuracy, complementing visual metrics like PSNR/SSIM [S1]. Effective in distinguishing between deformation-based and particle-based motion modeling [S1].
*   **Limitations:** Requires ground-truth motion data, which is not always available in real-world scenarios [S1]. The metric is sensitive to the choice of $\delta t$ and voxel resolution [S1].

**When to Use:**
Use MFE when evaluating the kinematic accuracy of dynamic NeRF models, particularly when comparing deformation-based methods against particle-based methods [S1]. It is most useful in synthetic datasets where ground-truth trajectories are known [S1].

## Background
Motion Field Error (MFE) was introduced in the context of Dynamic Appearance Particle Neural Radiance Field (DAP-NeRF) [S1]. The metric was designed to address the limitation of existing evaluation methods that focused primarily on visual quality (PSNR, SSIM) and user studies, without quantitatively assessing the physical accuracy of motion [S1]. It builds on the need to evaluate how well dynamic models capture physically relevant dynamics, such as object trajectories in 3D design software [S1]. The method compares dynamic NeRFs (like TiNeuVox) with particle-based models (like DAP-NeRF), highlighting the latter's ability to yield lower MFE in dynamic scenes by computing velocity directly from explicit particle motions rather than implying it from deformation fields [S1].

## Sources
[S1] Dynamic Appearance Particle Neural Radiance Field, https://doi.org/10.1109/tcsvt.2025.3540792, fetched 2026-11-05
