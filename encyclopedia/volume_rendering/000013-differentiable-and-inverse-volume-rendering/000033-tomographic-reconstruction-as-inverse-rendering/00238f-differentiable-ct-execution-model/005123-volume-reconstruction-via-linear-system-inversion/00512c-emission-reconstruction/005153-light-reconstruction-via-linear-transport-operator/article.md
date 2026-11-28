# Light Reconstruction via Linear Transport Operator

Also called: Environmental Illumination Optimization, Light Source Coefficient Optimization

## Overview
This method optimizes environmental illumination (outgoing radiance from the environment) to match a desired volume appearance, assuming the volume's intrinsic properties (emission, albedo, extinction) are spatially varying but fixed during the optimization process [S1]. It differs from sibling methods like [Emission Reconstruction](14424-emission-reconstruction.md) by treating the light source coefficients as the unknown variables rather than the volume properties, solving for the lighting that best explains the observed pixel values through a linear transport operator [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $N$ constraint pixels (desired observations $o$), a volume with fixed properties, and $M$ basis functions modeling the light sources (e.g., directional lights sampled from an environment map) [S1].
*   **Outputs:** An $M$-dimensional coefficient vector $\tilde{L}_o$ representing the optimized outgoing radiance from the environment [S1].

**Data Structures:**
*   **Transport Operator $T$:** A dense matrix of size $N \times M$ [S1]. Unlike the sparse weight matrices used in volume parameter reconstruction, $T$ is dense because it encodes complete light scattering in the scene [S1].
*   **Light Volumes ($vol_i$):** For Strategy 2, $M$ precomputed volumes are stored [S1]. Each $vol_i$ contains the radiance contribution of the $i$-th light basis function at every voxel in the scene [S1].

**Formulas:**
The problem is formulated as a least-squares minimization:
$$ \min_{\tilde{L}_o} ||T \tilde{L}_o - o||^2 $$
where:
*   $o$ is the vector of desired observation values for $N$ pixels [S1].
*   $T$ is the linear light transport operator matrix of size $N \times M$ [S1].
*   $\tilde{L}_o$ is the $M$-dimensional vector of light source coefficients [S1].

**Parameters:**
*   $M$: Number of basis functions. Typical value: ~16 coefficients are sufficient to capture light influence [S1].
*   $N$: Number of constraint pixels [S1].

**Steps:**
1.  **Precomputation:** Construct the operator $T$ or the intermediate light volumes [S1]. Two strategies are available:
    *   *Strategy 1 (Store T):* Compute the full $N \times M$ matrix $T$ [S1]. For each light coefficient $i$, render the scene with only that coefficient active (value 1, others 0) to fill the $i$-th column of $T$ [S1].
    *   *Strategy 2 (Light Volumes):* For each light coefficient $i$, light the entire volume with that coefficient active to create a "light volume" $vol_i$ [S1]. Store these $M$ volumes [S1].
2.  **Optimization Loop:** Solve the linear system using an iterative solver (e.g., Conjugate Gradient) [S1]. Each iteration requires two matrix-vector multiplications:
    *   **Forward Pass ($T \tilde{L}_o$):** Compute the predicted observation [S1].
        *   *If using Strategy 1:* Perform standard matrix-vector multiplication [S1].
        *   *If using Strategy 2:* For each constraint pixel, shoot a ray through the volume [S1]. Gather values from all precomputed light volumes $vol_i$ along the ray path and sum them weighted by $\tilde{L}_o$ [S1].
    *   **Backward Pass ($T^T r$):** Distribute the residual $r = o - T \tilde{L}_o$ back to the light coefficients [S1].
        *   *If using Strategy 1:* Perform standard matrix-vector multiplication with the transpose [S1].
        *   *If using Strategy 2:* For each constraint pixel, distribute its residual value to the light coefficients [S1]. This is analogous to volume optimization back-projection, but the values are accumulated into the light coefficient vector $\tilde{L}_o$ rather than voxel properties [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def light_reconstruction(constraint_pixels, volume, num_basis_functions=16, max_iterations=100, learning_rate=0.01, epsilon=1e-6):
    """
    Optimizes light source coefficients to match constraint pixels.
    Uses Strategy 2 (Light Volumes) for memory efficiency.
    """
    M = num_basis_functions
    N = len(constraint_pixels)
    
    # Precomputation: Generate light volumes
    # vol[i] is a 3D array representing the radiance contribution 
    # of the i-th basis function at every voxel.
    light_volumes = precompute_light_volumes(volume, M)
    
    # Initialize coefficients
    L_o = [0.0] * M
    
    # Optimization loop (simplified gradient descent for clarity)
    for iteration in range(max_iterations):
        # Forward Pass: y = T * L_o
        predicted_obs = [0.0] * N
        for p_idx in range(N):
            ray = constraint_pixels[p_idx]
            accumulated = 0.0
            # Gather values from all light volumes along the ray
            for i in range(M):
                contribution = integrate_light_volume(light_volumes[i], ray)
                accumulated += L_o[i] * contribution
            predicted_obs[p_idx] = accumulated
            
        # Compute residual
        residual = [constraint_pixels[p_idx] - predicted_obs[p_idx] for p_idx in range(N)]
        
        # Backward Pass: grad = T^T * residual
        grad = [0.0] * M
        for p_idx in range(N):
            r = residual[p_idx]
            if abs(r) < epsilon:
                continue
            ray = constraint_pixels[p_idx]
            # Distribute residual to each light coefficient based on 
            # the light volume's contribution at this pixel
            for i in range(M):
                sensitivity = integrate_light_volume(light_volumes[i], ray)
                grad[i] += r * sensitivity
                
        # Update coefficients
        for i in range(M):
            L_o[i] -= learning_rate * grad[i]
            
    return L_o

def precompute_light_volumes(volume, M):
    """
    For each of the M basis functions, render the volume 
    with that light source active to create a light volume.
    """
    volumes = []
    for i in range(M):
        # Create a temporary environment map with only the i-th direction active
        temp_env = [0.0] * M
        temp_env[i] = 1.0
        # Render the volume with this lighting to get the contribution
        vol = render_volume_with_lighting(volume, temp_env)
        volumes.append(vol)
    return volumes

def integrate_light_volume(light_vol, ray):
    """
    Integrates the light volume along a ray.
    Equivalent to the transport of the specific light basis function.
    """
    total = 0.0
    for t in ray.steps:
        voxel_val = light_vol[ray.sample(t)]
        transmittance = ray.transmittance(t)
        total += voxel_val * transmittance
    return total

def render_volume_with_lighting(volume, env_map):
    """
    Renders the volume with the given environment map.
    Returns a 3D array of radiance contributions per voxel.
    """
    return [[0.0] * volume.depth for _ in range(volume.height)]
```

**Edge Cases and Pitfalls:**
*   **Sparsity:** The operator $T$ is not sparse; it encodes complete light scattering. On-the-fly evaluation is too slow, requiring precomputation [S1].
*   **Memory:** Strategy 1 (storing $T$) has memory linear in $N$ (pixels). Strategy 2 (light volumes) has memory consumption linear in the size of the volume times the number of light coefficients [S1].
*   **Constant Volume Properties:** The method assumes volume properties are constant during optimization. If they vary, the operator $T$ is no longer constant, and this linear approach fails [S1].

## Subtopics

### Local Source Term Reconstruction
Reconstruction of the local radiative source term $j(\mathbf{r})$ after the attenuation coefficient $\alpha(\mathbf{r})$ is determined, solving the remaining unknown integral term in the Radiative Transfer Equation (RTE) [S2, unofficial]. Solve the linear system $\mathbf{G} = \mathbf{H}\mathbf{J}$ using an optimization approach to reconstruct $\mathbf{J}$, then compute $I(\mathbf{r})$ using the previously determined $\alpha(\mathbf{r})$ [S2, unofficial]. The unknown term is approximated by a discretized sum: $\mathbf{G} = \mathbf{H}\mathbf{J}$, where $\mathbf{G}$ is the known vector of the RHS of the previous equation, $\mathbf{H}$ is the attenuation-weighted geometric projection matrix constructed from LODs, and $\mathbf{J}$ is the vector of the local radiative source term $j(\mathbf{r})$ [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the precomputation strategy. Strategy 1 requires $M$ full volume renders for precomputation [S1]. Strategy 2 requires $M$ full volume renders for precomputation [S1]. Optimization iterations require $O(N \cdot M)$ work per iteration for Strategy 2 (ray traversal through $M$ volumes) or $O(N \cdot M)$ for Strategy 1 (matrix-vector multiply) [S1].
*   **Memory Complexity:** Strategy 1: $O(N \cdot M)$ to store $T$ [S1]. Strategy 2: Memory consumption grows linearly in the size of the volume times the number of light coefficients [S1].
*   **Strengths:** Allows independent optimization of lighting without altering volume geometry or material properties. Efficient if $M$ is small (~16) [S1].
*   **Limitations:** Requires precomputation. Cannot handle dynamic volume properties. Dense matrix operations can be memory-intensive for high-resolution constraints [S1].
*   **When to use:** When the volume structure is known and fixed, but the lighting needs to be adjusted to match a target appearance (e.g., stylization) [S1].

## Background
This method is described in [S1]. It builds on the linearity of light transport in participating media, allowing the transport operator to be separated from the source terms [S1]. The approach is analogous to tomographic reconstruction but applied to the light source coefficients rather than the volume density [S1].

## Sources
[S1] Property and Lighting Manipulations for Static Volume Stylization Using a Painting Metaphor, https://doi.org/10.1109/TVCG.2014.13, fetched 2026-10-26
[S2] Tomographic Reconstruction of Infrared Radiance from Axisymmetric Sprays in a Hot Environment Using Multi-Parameter Regularization, https://doi.org/10.21203/rs.3.rs-10907544/v1, fetched 2026-10-26 (unofficial)
