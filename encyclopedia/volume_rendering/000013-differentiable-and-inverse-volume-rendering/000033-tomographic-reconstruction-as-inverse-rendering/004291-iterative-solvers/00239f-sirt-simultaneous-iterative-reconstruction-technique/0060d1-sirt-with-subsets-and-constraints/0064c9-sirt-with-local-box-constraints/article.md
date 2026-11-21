# SIRT with Local Box Constraints

Also called: Constrained SIRT for Dynamic Tomography, Dynamic SIRT with static initialization and segmentation constraints, SIRT-LC

## Overview
SIRT-LC is an iterative tomographic reconstruction algorithm designed for dynamic multiphase flow experiments, solving the problem of reconstructing high-quality volumes from data with significantly reduced exposure times and fewer projections [S1]. It sets itself apart from standard SIRT and its variant SIRT-IC by applying local box constraints derived from the segmentation of a high-quality static reference scan, which eliminates noise in specific voxel types (rock vs. fluid) that global constraints cannot address [S1]. Unlike difference-projection methods that reconstruct only changes, SIRT-LC reconstructs the full volume at each step while enforcing physical attenuation bounds specific to the material phase identified in the static scan [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Measured projection data $b \in \mathbb{R}^m$ (radiographs) for the current time step; Forward projection operator $A \in \mathbb{R}^{m \times n}$; Segmentation mask $M$ derived from a high-quality static reconstruction; Attenuation values for rock ($\mu_{rock}$), oil ($\mu_{oil}$), and water ($\mu_{water}$); Global bounds $\mu_{min}$ and $\mu_{max}$.
*   **Output:** Reconstructed volume vector $x \in \mathbb{R}^n$ representing voxel attenuation coefficients.

**Data Structures:**
*   **Segmentation Mask ($M$):** A vector of length $n$ where each element $M_i$ classifies voxel $i$ into one of three categories:
    *   `ROCK`: Voxel is part of the static rock matrix.
    *   `FLUID`: Voxel is part of the dynamic fluid phase.
    *   `AMBIGUOUS`: Voxel value is indeterminate between rock and fluid.
*   **Convex Set ($C$):** Defined by the segmentation mask, specifying the allowed range $[L_i, U_i]$ for each voxel $i$.

**Numbered Steps:**
1.  **Static Initialization:** For the first time step ($t=0$), initialize the reconstruction volume $x_0$ using the high-quality reconstruction of the static system (the system before dynamic flow begins) [S1].
2.  **Segmentation:** Segment the static reconstruction volume by thresholding to identify rock and fluid regions [S1]:
    *   Identify **rock voxels** as those with attenuation values above a specific threshold [S1].
    *   Identify **fluid voxels** as those with values below the threshold AND between the attenuation values of oil and water ($\mu_{oil} < x < \mu_{water}$) [S1].
    *   Identify **ambiguous voxels** as those with attenuation values larger than water but smaller than rock [S1].
3.  **Dynamic Initialization:** For each subsequent time step $t > 0$, initialize the reconstruction volume $x_0^{(t)}$ with the final solution from the immediately preceding time step $x_{final}^{(t-1)}$ to maintain temporal continuity [S1].
4.  **Define Local Constraints:** Construct the convex set $C$ based on the segmentation mask $M$ from Step 2:
    *   For **rock voxels**: Fix the value to the expected constant attenuation value $\mu_{rock}$. Thus, $L_i = U_i = \mu_{rock}$.
    *   For **fluid voxels**: Constrain values to the interval between oil and water attenuation values. Thus, $L_i = \min(\mu_{oil}, \mu_{water})$ and $U_i = \max(\mu_{oil}, \mu_{water})$.
    *   For **ambiguous voxels**: Apply regular global box constraints. Thus, $L_i = \mu_{min}$ and $U_i = \mu_{max}$.
5.  **Iterative Update:** Perform the SIRT update step and project the result onto the convex set $C$ [S1]:
    $$x_{k+1} = P_C(x_k + C A^T R (b - A x_k)) = P_C(\text{SIRT}(x_k))$$
    Where:
    *   $x_k$ is the volume estimate at iteration $k$.
    *   $A^T$ is the back-projection operator.
    *   $R$ is a weighting matrix (often identity or diagonal normalization depending on the specific SIRT implementation, e.g., ASTRA toolbox) [S1].
    *   $C$ is a relaxation parameter (step size) [S1].
    *   $P_C$ is the projection operator onto the convex set $C$ [S1].
6.  **Projection Operator ($P_C$):** For each voxel $i$, compute $P_C(v)_i$ as:
    *   If $M_i = \text{ROCK}$: return $\mu_{rock}$.
    *   If $M_i = \text{FLUID}$: return $\text{clip}(v_i, \min(\mu_{oil}, \mu_{water}), \max(\mu_{oil}, \mu_{water}))$.
    *   If $M_i = \text{AMBIGUOUS}$: return $\text{clip}(v_i, \mu_{min}, \mu_{max})$.
7.  **Stopping Criterion:** Iterate until a stopping criterion is met, such as the NCP (Normalized Cyclic Projection) criterion or a fixed number of iterations [S1].

**Parameters:**
*   **$N_{proj}$:** Number of projections acquired per time step. The method is designed to work with significantly fewer projections than traditionally required (e.g., 45–360 vs. thousands) [S1].
*   **$\mu_{min}$:** The smallest attenuation value in the sample (lower limit). Typically 0 [S1].
*   **$\mu_{max}$:** The largest attenuation value in the sample (upper limit). Typically 2.5 for the tested case [S1].
*   **$C$:** Relaxation parameter for SIRT convergence [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sirt_lc_reconstruction(projections_b, forward_op_A, backproj_op_AT, 
                           static_volume, rock_threshold, mu_oil, mu_water, 
                           mu_min=0.0, mu_max=2.5, num_iterations=100, 
                           relaxation_C=1.0):
    """
    SIRT with Local Box Constraints for dynamic tomography.
    
    Args:
        projections_b: Array of projection data for current time step.
        forward_op_A: Function performing forward projection (A * x).
        backproj_op_AT: Function performing back projection (A^T * r).
        static_volume: High-quality reconstruction of the static system.
        rock_threshold: Threshold value to segment rock from fluid in static_volume.
        mu_oil: Attenuation value of oil.
        mu_water: Attention value of water.
        mu_min: Global minimum attenuation.
        mu_max: Global maximum attenuation.
        num_iterations: Number of SIRT iterations.
        relaxation_C: SIRT relaxation parameter.
        
    Returns:
        reconstructed_volume: The reconstructed volume for the current time step.
    """
    
    # Step 1 & 2: Segment static volume to create local constraints
    # Identify rock voxels: value > threshold
    is_rock = static_volume > rock_threshold
    
    # Identify fluid voxels: value < threshold AND between oil and water
    # Note: Source defines fluid as below threshold AND between oil/water values.
    # Ambiguous are those > water but < rock (implied by exclusion).
    # We assume mu_oil and mu_water define the fluid range.
    fluid_lower = min(mu_oil, mu_water)
    fluid_upper = max(mu_oil, mu_water)
    is_fluid = (static_volume < rock_threshold) & \
               (static_volume >= fluid_lower) & \
               (static_volume <= fluid_upper)
    
    # Ambiguous voxels are those not rock and not fluid
    is_ambiguous = ~is_rock & ~is_fluid
    
    # Determine fixed rock value from static volume (expected constant attenuation)
    # Using the median or mean of rock voxels in static volume as the fixed value
    rock_mask = is_rock
    if np.any(rock_mask):
        mu_rock = np.median(static_volume[rock_mask])
    else:
        # Fallback if no rock identified, though unlikely in context
        mu_rock = mu_max 
        
    # Step 3: Initialize volume
    # For t=0, use static_volume. For t>0, use previous result.
    # This function assumes it is called for a specific time step.
    # The caller should pass the previous time step's result as initial_x if t>0.
    # Here we assume initial_x is passed or derived. 
    # To make this self-contained for the "method" description, we initialize 
    # with the static volume if no previous context is provided, but the algorithm 
    # dictates linking to previous step. We will assume `initial_x` is provided 
    # by the loop in the calling code.
    # However, to be strictly self-contained as a method description:
    # We initialize with static_volume for the first call, but the logic 
    # for subsequent steps relies on external state. 
    # We will define the update step clearly.
    
    # Let's assume `current_x` is the initialization passed in or set to static_volume
    # for the purpose of this function signature representing the core solver.
    # In practice:
    # if time_step == 0: current_x = static_volume
    # else: current_x = previous_time_step_result
    
    # Since the prompt asks for the method, we define the iterative core.
    # We will initialize with static_volume for demonstration, 
    # but note the temporal linking requirement.
    current_x = static_volume.copy() 
    
    for k in range(num_iterations):
        # Standard SIRT residual update
        # r = b - A * x_k
        # x_k + C * A^T * R * r
        # Assuming R is identity or handled by the specific SIRT implementation (ASTRA)
        # Here we implement the basic SIRT update: x_new = x + C * A^T * (b - A*x)
        
        forward_proj = forward_op_A(current_x)
        residual = projections_b - forward_proj
        
        # Backprojection and update
        # Note: Standard SIRT often includes normalization in A or A^T. 
        # The formula x_{k+1} = P_C(x_k + C A^T R (b - A x_k)) is given.
        # We assume A^T handles the geometry.
        update = backproj_op_AT(residual)
        
        # Apply relaxation
        # C is the relaxation parameter
        current_x = current_x + relaxation_C * update
        
        # Step 4 & 5: Project onto Local Box Constraints
        projected_x = np.zeros_like(current_x)
        
        # Apply constraints based on segmentation
        # Rock voxels: fixed to mu_rock
        projected_x[is_rock] = mu_rock
        
        # Fluid voxels: constrained to [mu_oil, mu_water]
        projected_x[is_fluid] = np.clip(current_x[is_fluid], fluid_lower, fluid_upper)
        
        # Ambiguous voxels: global box constraints
        projected_x[is_ambiguous] = np.clip(current_x[is_ambiguous], mu_min, mu_max)
        
        current_x = projected_x
        
    return current_x
```

**Edge Cases and Pitfalls:**
*   **Underestimation of Iterations:** SIRT-LC may underestimate the number of iterations needed, leading to large regions within the fluid phase that are either overestimated or underestimated [S1].
*   **Noise in Fluid Phases:** While SIRT-LC eliminates noise compared to SIRT-IC, improper segmentation or thresholding in the static scan can lead to incorrect classification of ambiguous voxels, propagating errors [S1].
*   **Temporal Deterioration:** The $\ell_2$-norm of the residual may vary across time steps due to the initialization linking the current step to the previous one, potentially causing slight performance degradation over long sequences if the dynamic changes are significant [S1].

## Subtopics

### SIRT with Initialisation Constraints
A tomographic reconstruction method that uses a high-quality static data set to initialise the reconstruction of dynamic time steps, linking the current time step with the previous one [S1]. It initializes time step $t=0$ with the reconstruction of the static system and remaining time steps $t>0$ with the reconstruction of the previous time step $t-1$ [S1]. It applies global box constraints to truncate voxel values to $[\mu_{min}, \mu_{max}]$ [S1]. Unlike SIRT-LC, it does not apply local segmentation constraints, resulting in fair bit of noise in the reconstructed fluid phases [S1]. In the best data case ($N_{proj}=360, \rho=0.25\%$), SIRT-IC achieves an $\ell_2$-norm of $1.34 \cdot 10^{-3}$, which is slightly inferior to SIRT-LC ($1.43 \cdot 10^{-3}$) but superior to SIRT ($1.70 \cdot 10^{-3}$) [S1].

### SIRT with Global Box Constraints
A variant of SIRT that applies global box constraints to truncate voxel attenuation coefficients to the minimum and maximum values present in the sample [S1]. After the standard SIRT update step, it projects the resulting voxel values onto the convex set $C = [\mu_{min}, \mu_{max}]^n$ [S1]. For the tested case, $\mu_{min}=0$ and $\mu_{max}=2.5$ [S1]. It does not use initialization from previous time steps or local segmentation constraints [S1].

## Operating characteristics
*   **Time Complexity:** Iterative, dependent on the number of projections $N_{proj}$, the number of voxels $n$, and the number of iterations. Each iteration involves a forward and back projection.
*   **Memory Complexity:** Requires storage for the volume $x$, projection data $b$, and the system matrix $A$ (or its efficient representation via the ASTRA toolbox) [S1].
*   **Measured Performance:** SIRT-LC is considered the best algorithm in general across all data cases tested [S1]. In the worst data case ($N_{proj}=45, \rho=5.00\%$), SIRT-LC achieves an $\ell_2$-norm of $3.34 \cdot 10^{-3}$, outperforming SIRT ($11.49 \cdot 10^{-3}$) and SIRT-BC ($11.06 \cdot 10^{-3}$) [S1].
*   **Strengths:** Maintains high image quality with significantly fewer projections and higher noise levels than FBP/FDK [S1]. Eliminates noise in fluid phases better than SIRT-IC by using local constraints [S1]. Produces sharper peaks in the voxel value histogram, fitting the visual appearance of higher contrast [S1].
*   **Limitations:** Requires a high-quality static scan of the system before the dynamic process begins [S1]. Performance may degrade slightly across time steps due to initialization linking [S1].
*   **When to Use:** Dynamic multiphase flow experiments where exposure time and number of projections are limited, and a static reference scan is available [S1].
*   **When Not to Use:** Static reconstruction where FBP/FDK is sufficient, or dynamic scenes where no static reference is available or the static structure changes significantly [S1].

## Background
The method was developed to study fluid flow processes in porous media using laboratory X-ray tomography, addressing the limitations of low X-ray brilliance and prolonged exposure times [S1]. It extends the Simultaneous Iterative Reconstruction Technique (SIRT) by incorporating prior knowledge from a static scan [S1]. The algorithm was compared against FBP, FDK, SIRT, SIRT-BC, SIRT-IC, and difference-projection methods by Myers et al. and rSIRT-PWC by Van Eyndhoven et al. [S1]. The implementation utilizes the ASTRA toolbox for optimized C++ and CUDA code, callable via Python [S1].

## Sources
[S1] Improved dynamic imaging of multiphase flow by constrained tomographic reconstruction., https://doi.org/10.1038/s41598-021-91776-1, fetched 2026-10-26
