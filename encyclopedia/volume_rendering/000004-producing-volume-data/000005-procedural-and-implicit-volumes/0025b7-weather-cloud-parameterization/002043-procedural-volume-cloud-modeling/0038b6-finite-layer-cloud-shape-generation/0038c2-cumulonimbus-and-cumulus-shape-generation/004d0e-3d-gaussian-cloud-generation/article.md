# 3D Gaussian Cloud Generation
Also called: Gaussian pseudo-spheroid generation
## Overview
An algorithm for generating Cumulus (type 9) and Cumulonimbus (type 10) clouds using a sparse set of 3D Gaussian-distributed pseudo-spheroid primitives [S1]. It differs from particle-based systems by requiring significantly fewer primitives (typically 35) to achieve realistic results, and from hierarchical implicit blobs by using explicit geometric primitives with pre-processing filters to optimize rendering performance [S1].
## How it works
**Inputs:** Cloud center $C$; number of primitives $N$ (typically 35); scale constants $t_x, t_y, t_z$; maximum radius $R_{max}$; radius spread $\sigma_r$ [S1].
**Outputs:** List of $N$ pseudo-spheroids, each defined by position $P_i$, orientation (implied by axis scaling), and radius $r_i$ [S1].

**Data Structures:**
*   `PseudoSpheroid`: struct containing `position` (vec3), `radius` (float), `mean` (vec3), `std_dev` (vec3).

**Parameters:**
*   $N = 35$ [S1].
*   $R_{max}$: Maximum possible radius of a sphere in the variant [S1].
*   $\sigma_r$: Radius spread constant [S1].
*   $t_x, t_y, t_z$: Scale constants for the normal distribution axes [S1].
*   Typical values for means and standard deviations are provided in Table 2 of the source, but exact numerical values are not listed in the text [S1].

**Steps:**
1.  **Generate Positions:** For each primitive $i$ from 1 to $N$:
    *   Sample a random variable $\mathcal{N}$ with normal distribution for each axis $x, y, z$ using the specific means and standard deviations defined for the cloud type [S1].
    *   Calculate position $P_i$ relative to cloud center $C$:
        $$ P_i = C + \mathcal{N}(\mu, \sigma) $$
        Where $\mu$ and $\sigma$ are the mean and standard deviation vectors for the respective axes [S1].
    *   *Note:* The source mentions using clamping similar to Huang et al. [1], but does not specify the clamp bounds [S1]. [G] Implementation should clamp sampled values to prevent outliers from extending the cloud too far, typically within $\pm 3\sigma$.

2.  **Calculate Radii:** For each primitive $i$:
    *   Calculate distance $d_i$ from $P_i$ to $C$:
        $$ d_i = \| P_i - C \| $$
    *   Compute radius $r_i$ using Equation (16) for real cumulus generation [S1]:
        $$ r_i = R_{max} \cdot e^{-\frac{d_i^2}{2\sigma_r^2}} $$
        Where $R_{max}$ is the maximum radius and $\sigma_r$ is the radius spread constant [S1].
    *   *Note:* Equation (15) is a variant, but Equation (16) is specified for the experiments shown [S1].

3.  **Pre-processing Filter 1 (Hollow Cloud Removal):**
    *   Identify spheroids within a specific range from the center that would create a hollow appearance.
    *   Remove approximately 20% of spheroids in the nucleus [S1].
    *   *Note:* The exact range threshold is not given in the text [S1]. [G] A common approach is to remove primitives where $d_i < \epsilon$ for a small threshold $\epsilon$, or based on a density threshold.

4.  **Pre-processing Filter 2 (Containment Removal):**
    *   For every pair of spheroids $A$ and $B$:
        *   Calculate distance $d_{AB}$ between centers $P_A$ and $P_B$.
        *   Check if sphere $A$ is inside sphere $B$:
            $$ d_{AB} + r_A < r_B $$
        *   If true, remove sphere $A$ [S1].
    *   Repeat for all pairs to ensure no primitive is fully contained within another [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def generate_3d_gaussian_cloud(center, num_primitives=35, r_max=1.0, sigma_r=0.5, means=None, std_devs=None):
    """
    Generates a cloud using 3D Gaussian distributed pseudo-spheroids.
    
    Args:
        center: (x, y, z) tuple for cloud center C.
        num_primitives: Number of spheroids (typically 35).
        r_max: Maximum radius constant.
        sigma_r: Radius spread constant.
        means: List of 3 tuples (mean_x, mean_y, mean_z). Not given in sources. [G]
        std_devs: List of 3 tuples (std_x, std_y, std_z). Not given in sources. [G]
        
    Returns:
        List of dicts with 'position' and 'radius'.
    """
    # Default values if not provided (placeholders for missing Table 2 data)
    if means is None:
        means = [(0, 0, 0), (0, 0, 0), (0, 0, 0)] # Placeholder
    if std_devs is None:
        std_devs = [(1, 1, 1), (1, 1, 1), (1, 1, 1)] # Placeholder

    primitives = []

    # Step 1: Generate Positions
    for i in range(num_primitives):
        # Sample from normal distribution for each axis
        # Note: Source references Table 2 for specific means/std_devs per axis
        # Implementation assumes means/std_devs are applied per axis
        x = random.gauss(means[0][0], std_devs[0][0])
        y = random.gauss(means[0][1], std_devs[0][1])
        z = random.gauss(means[0][2], std_devs[0][2])
        
        # Apply clamping as mentioned in source (similar to Huang et al.)
        # [G] Clamp bounds not specified; typical implementation uses ±3σ or similar
        # For this implementation, we assume raw sampling or simple clamping
        # position = center + (x, y, z)
        position = (center[0] + x, center[1] + y, center[2] + z)
        
        # Step 2: Calculate Radius
        dx = position[0] - center[0]
        dy = position[1] - center[1]
        dz = position[2] - center[2]
        distance = math.sqrt(dx*dx + dy*dy + dz*dz)
        
        # Equation (16) for real cumulus generation
        radius = r_max * math.exp(-(distance**2) / (2 * sigma_r**2))
        
        primitives.append({
            'position': position,
            'radius': radius
        })

    # Step 3: Pre-processing Filter 1 (Hollow Cloud Removal)
    # Source states ~20% of nucleus spheroids are removed.
    # Exact range not given. [G] Removing primitives very close to center
    # to create a hollow effect.
    nucleus_threshold = 0.1 * r_max # Placeholder threshold
    filtered_primitives = []
    for p in primitives:
        dx = p['position'][0] - center[0]
        dy = p['position'][1] - center[1]
        dz = p['position'][2] - center[2]
        dist = math.sqrt(dx*dx + dy*dy + dz*dz)
        if dist > nucleus_threshold:
            filtered_primitives.append(p)
    
    # If filtering didn't remove ~20%, adjust threshold or logic
    # Source does not specify exact algorithm for this filter.
    # [G] In practice, this might involve sorting by distance and removing
    # the innermost 20% of primitives.
    if len(filtered_primitives) > 0.8 * len(primitives):
        # Fallback: Remove innermost 20%
        filtered_primitives.sort(key=lambda p: math.sqrt(
            (p['position'][0]-center[0])**2 + 
            (p['position'][1]-center[1])**2 + 
            (p['position'][2]-center[2])**2
        ))
        filtered_primitives = filtered_primitives[int(0.2 * len(filtered_primitives)):]

    # Step 4: Pre-processing Filter 2 (Containment Removal)
    final_primitives = []
    for i, p_a in enumerate(filtered_primitives):
        is_contained = False
        for j, p_b in enumerate(filtered_primitives):
            if i == j:
                continue
            # Check if A is inside B
            dx = p_a['position'][0] - p_b['position'][0]
            dy = p_a['position'][1] - p_b['position'][1]
            dz = p_a['position'][2] - p_b['position'][2]
            dist_ab = math.sqrt(dx*dx + dy*dy + dz*dz)
            
            # Condition: distance + radius_A < radius_B
            if dist_ab + p_a['radius'] < p_b['radius']:
                is_contained = True
                break
        if not is_contained:
            final_primitives.append(p_a)

    return final_primitives
```

## Operating characteristics
*   **Complexity:** The pre-processing computation (comparing radii and distances) is not a drawback due to the reduced number of spheres (e.g., 35) required in the Gaussian model [S1].
*   **Performance:** The pre-processing filter 1 yields a ~30% average improvement in frame rate without affecting the look of the cloud [S1].
*   **Strengths:** Requires significantly fewer primitives (typically 35) compared to particle systems (thousands) [S1].
*   **Limitations:** Relies on specific parameter tuning (means, standard deviations) which are not fully detailed in the source [S1].

## Background
The method is part of a procedural volume cloud modeling approach for real-time GPU volumetric rendering [S1]. It builds on the concept of using 3D Gaussian distributions to model cumuliform cloud shapes, which naturally resemble normal distributions [S1]. The position generation uses clamping similar to the method by Huang et al. [1], but adapts it for explicit pseudo-spheroid primitives rather than particles [S1].

## Sources
[S1] Efficient Algorithms for Real-Time GPU Volumetric Cloud Rendering with Enhanced Geometry, https://doi.org/10.3390/sym10040125, fetched 2026-10-26
