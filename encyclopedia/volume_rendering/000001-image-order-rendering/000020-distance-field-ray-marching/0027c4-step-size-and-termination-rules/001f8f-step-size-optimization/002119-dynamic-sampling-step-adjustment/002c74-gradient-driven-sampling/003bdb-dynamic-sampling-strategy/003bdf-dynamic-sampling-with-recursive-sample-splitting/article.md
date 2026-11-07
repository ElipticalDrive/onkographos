# Dynamic Sampling with Recursive Sample Splitting

Also called: Recursive Sample Spliting, Adaptive Resampling with Gradient-Based Error Metric

## Overview
Dynamic Sampling with Recursive Sample Splitting is an adaptive resampling scheme that dynamically places additional samples in areas with greater shading error to accurately model sharp shading variations (e.g., light shafts) that sparse sampling misses [S1]. It builds on [Dynamic Sampling Strategy](9081-dynamic-sampling-strategy.md) by using a specific gradient-based error metric to drive a recursive splitting process that replaces high-error samples with finer grid vertices [S1]. Unlike static distance-field marching, this method operates on a set of valid spheres (typically derived from Gaussian centers) and refines them iteratively based on local radiance approximation errors [S1].

## How it works
### Inputs and Outputs
**Inputs:**
*   Density field $D(x)$ [S1].
*   Initial sample set $Q_0$, typically consisting of Gaussian centers $\{c_j\}$ of the density representation [S1].
*   Threshold $\epsilon$ for local shading error [S1].
*   Grid resolutions for splitting levels $k$: $G_1$ ($16^3$), $G_2$ ($32^3$), $G_3$ ($64^3$) [S1].
*   Maximum grid resolution, limited to half the density field resolution (e.g., $128^3$ for a $256^3$ field) [S1].

**Outputs:**
*   Final set of sample points $Q_{final}$ and their associated valid radii and shading data [S1].

### Data Structures
**Renderable 3D Grid Information Buffer:**
A 3D texture or buffer corresponding to the current grid $G_k$. For each vertex in $G_k$, it stores:
1.  A boolean indicator: whether the vertex is currently in the sample set $Q_k$ [S1].
2.  The local shading error $E_j$ for the corresponding sample [S1].

### Parameters
*   **Sample points for error metric ($x_{ij}$):** 6 points along the axes scaled by the radius: $\{x_j \pm R_j \mathbf{X}, x_j \pm R_j \mathbf{Y}, x_j \pm R_j \mathbf{Z}\}$ [S1]. Thus, $n=6$ [S1].
*   **Valid Radius ($R_j$):** For initial Gaussian centers, $R_j = 3r_j$ where $r_j$ is the culling radius of the Gaussian [S1]. For split samples from grid $G_k$, the valid radius is equal to the grid interval of $G_k$ [S1].
*   **Transmittance Approximation ($\tau_{x_j v}$):** Approximated as the transmittance from the sphere center $x_j$ to the viewer for all points in the sphere to ensure computational efficiency [S1]. For descendant spheres, the $\tau_{x_j v}$ of the original parent sphere is inherited [S1].
*   **Threshold ($\epsilon$):** Not given in the sources.

### Formulas
**Local Shading Error ($E_j$):**
$$E_j = R_j^3 \sum_{i=1}^n \frac{|\tilde{L}_{x_{ij}} - L_{x_{ij}}|}{n} D(x_{ij}) \tau_{x_j v} \quad (5)$$
Where:
*   $R_j$: Valid radius of sample $j$ [S1].
*   $n$: Number of sample points within the sphere (6) [S1].
*   $\tilde{L}_{x_{ij}}$: Approximate source radiance at point $x_{ij}$ [S1].
*   $L_{x_{ij}}$: Actual source radiance at point $x_{ij}$ [S1].
*   $D(x_{ij})$: Density at point $x_{ij}$ [S1].
*   $\tau_{x_j v}$: Transmittance from sphere center to viewer [S1].

**Approximate Source Radiance ($\tilde{L}_{x_{ij}}$):**
Computed using a first-order Taylor expansion from the single sample point $j$:
$$\tilde{L}_{x_{ij}} \approx L_{x_j} + (x_{ij} - x_j) \nabla L_{x_j}$$
Where:
*   $L_{x_j}$: Source radiance at sample center $x_j$ [S1].
*   $\nabla L_{x_j}$: Gradient of source radiance at $x_j$ [S1].

**Source Radiance ($L_x$):**
Computed via volume tracing along the ray from $x$ to light source $s$:
$$L_x = \frac{I_0}{4\pi d_{sv}^2} \exp\left(-\sigma_t \Delta_1 \sum_{u \in \mathbb{U}} D(u)\right)$$
Where:
*   $I_0$: Light intensity [S1].
*   $d_{sv}$: Distance from source to viewer (or relevant distance metric) [S1].
*   $\sigma_t$: Total extinction coefficient [S1].
*   $\Delta_1$: Step size for volume tracing [S1].
*   $\mathbb{U}$: Set of points $u_k = x + v k \Delta_1$ along the ray direction $v = (s-x)/\|s-x\|$ [S1].

### Algorithm Steps
1.  **Initialization:** Set $Q_0 = \{c_j\}$ (Gaussian centers). Compute valid radius $R_j = 3r_j$ for each [S1].
2.  **Iteration Loop:** For each level $k = 1, 2, \dots$ up to max resolution:
    a.  **Sampling:** For each sample $j \in Q_{k-1}$:
        i.  Compute $L_{x_j}$ and $\nabla L_{x_j}$ using volume tracing [S1].
        ii. Compute $L_{x_{ij}}$ for the 6 axis-aligned points $x_{ij}$ using volume tracing [S1].
        iii. Compute $D(x_{ij})$ from the density field [S1].
        iv. Compute $\tau_{x_j v}$ (inherited from parent if applicable) [S1].
        v.  Compute $\tilde{L}_{x_{ij}}$ using the Taylor expansion [S1].
        vi. Compute $E_j$ using Eq. (5) [S1].
    b.  **Filtering:** Identify samples where $E_j > \epsilon$ [S1].
    c.  **Splitting:** For each sample $j$ with $E_j > \epsilon$:
        i.  Define grid $G_k$ (e.g., $16^3, 32^3, 64^3$) [S1].
        ii. Create set $Q^1_j = \{q : q \in G_k \land \|q - x_j\| < R_{x_j}\}$ containing grid vertices within the valid sphere [S1].
        iii. Assign valid radius $R_{new}$ equal to the grid interval of $G_k$ to all $q \in Q^1_j$ [S1].
        iv. Remove original sample $j$ from $Q_{k-1}$ [S1].
    d.  **Update:** Collect all split vertices into $Q_k = \bigcup_j Q^1_j$ [S1].
    e.  **Termination:** If $k$ is max level or no samples have $E_j > \epsilon$, stop. Else, increment $k$ and repeat [S1].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dynamic_sampling_with_recursive_splitting(density_field, gaussian_centers, epsilon, max_level):
    """
    Implements Dynamic Sampling with Recursive Sample Splitting.
    
    Args:
        density_field: Function D(x) returning density at x.
        gaussian_centers: List of initial sample centers c_j.
        epsilon: Threshold for local shading error.
        max_level: Maximum recursion depth (grid resolution level).
        
    Returns:
        final_samples: List of sample objects with position, radius, and shading data.
    """
    
    # Grid resolutions for each level
    grid_resolutions = [16, 32, 64] # G1, G2, G3
    
    # Initialize Q_0
    current_samples = []
    for c in gaussian_centers:
        r_j = 3 * c.culling_radius # Assuming c has culling_radius attribute
        sample = {
            'pos': c.position,
            'radius': r_j,
            'level': 0,
            'parent_tau': None, # To be computed
            'error': 0.0
        }
        current_samples.append(sample)
        
    final_samples = []
    
    for k in range(1, max_level + 1):
        if k > len(grid_resolutions):
            break
            
        grid_res = grid_resolutions[k-1]
        grid_interval = 1.0 / grid_res # Assuming normalized space, adjust as needed
        next_samples = []
        
        # 1. Sampling and Error Computation
        for sample in current_samples:
            x_j = sample['pos']
            R_j = sample['radius']
            
            # Compute L_xj and gradient
            L_xj, grad_L_xj = compute_source_radiance_and_gradient(x_j, density_field)
            
            # Compute transmittance tau_xjv (approximated from center to viewer)
            # In practice, this requires viewer position and ray marching
            tau_xjv = compute_transmittance_to_viewer(x_j, density_field)
            sample['tau'] = tau_xjv
            
            # Define 6 sample points
            axes = [(1,0,0), (-1,0,0), (0,1,0), (0,-1,0), (0,0,1), (0,0,-1)]
            sum_error = 0.0
            
            for dx, dy, dz in axes:
                x_ij = x_j + R_j * vector(dx, dy, dz)
                
                # Actual radiance
                L_xij = compute_source_radiance(x_ij, density_field)
                
                # Approximate radiance via Taylor expansion
                # L_tilde = L_xj + (x_ij - x_j) . grad_L_xj
                diff = vector(x_ij.x - x_j.x, x_ij.y - x_j.y, x_ij.z - x_j.z)
                L_tilde_xij = L_xj + dot(diff, grad_L_xj)
                
                # Density
                D_xij = density_field(x_ij)
                
                # Error contribution
                # |L_tilde - L| / n * D * tau
                # Note: The formula sums over n, so we divide by n at the end or accumulate
                sum_error += abs(L_tilde_xij - L_xij) * D_xij * tau_xjv
                
            # E_j = R_j^3 * (sum_error / n)
            n = 6
            E_j = (R_j ** 3) * (sum_error / n)
            sample['error'] = E_j
            
            # 2. Filtering and Splitting
            if E_j > epsilon:
                # Split this sample
                # Find grid vertices within valid sphere
                # Grid vertices are spaced by grid_interval
                # Center is x_j. Range is [x_j.x - R_j, x_j.x + R_j] etc.
                
                min_x = floor((x_j.x - R_j) / grid_interval) * grid_interval
                max_x = ceil((x_j.x + R_j) / grid_interval) * grid_interval
                min_y = floor((x_j.y - R_j) / grid_interval) * grid_interval
                max_y = ceil((x_j.y + R_j) / grid_interval) * grid_interval
                min_z = floor((x_j.z - R_j) / grid_interval) * grid_interval
                max_z = ceil((x_j.z + R_j) / grid_interval) * grid_interval
                
                for gx in range(min_x, max_x + grid_interval, grid_interval):
                    for gy in range(min_y, max_y + grid_interval, grid_interval):
                        for gz in range(min_z, max_z + grid_interval, grid_interval):
                            q = vector(gx, gy, gz)
                            if distance(q, x_j) < R_j:
                                new_sample = {
                                    'pos': q,
                                    'radius': grid_interval,
                                    'level': k,
                                    'parent_tau': tau_xjv, # Inherit transmittance
                                    'error': 0.0
                                }
                                next_samples.append(new_sample)
            else:
                # Keep sample if error is low
                final_samples.append(sample)
                
        current_samples = next_samples
        if not current_samples:
            break
            
    # Add remaining samples from last iteration if they didn't split (or if max level reached)
    # The algorithm says "output final set" after max resolution or no splits
    final_samples.extend(current_samples)
    
    return final_samples

def compute_source_radiance_and_gradient(x, density_field):
    """
    Computes L_x and gradient using volume tracing.
    This is a placeholder for the specific volume tracing implementation details
    which involve marching from x to light source s.
    """
    # Ray direction to light source s
    v = normalize(light_source_position - x)
    distance_to_light = length(light_source_position - x)
    
    # Discrete integration
    # L_x = (I_0 / (4 * pi * d_sv^2)) * exp(-sigma_t * Delta_1 * sum(D(u)))
    # Note: d_sv in source is likely distance from source to viewer or similar, 
    # but the formula uses d_sv^2 in denominator. Assuming standard point light attenuation.
    
    # For gradient, multiple samples around x are needed to estimate grad L_x
    # The source states "measures source radiance gradients using multiple samples in a valid local neighborhood"
    # but the error metric uses grad L_xj computed at x_j.
    # We assume a central difference or similar finite difference method is used here.
    
    # Simplified: Return dummy values for structure
    L_x = 0.0
    grad_L_x = vector(0, 0, 0)
    return L_x, grad_L_x

def compute_source_radiance(x, density_field):
    """
    Computes L_x using volume tracing.
    """
    # Same logic as above but for a single point
    return 0.0

def compute_transmittance_to_viewer(x, density_field):
    """
    Computes tau_xv.
    """
    return 0.0
```

### Edge Cases and Pitfalls
*   **Precision:** The error metric involves summing small differences multiplied by density and transmittance. Floating-point precision may affect the accuracy of $E_j$ in low-density regions [S1].
*   **Boundaries:** The splitting process relies on grid vertices. If a valid sphere is near the boundary of the density field, the grid vertices may be clipped. The sources do not specify boundary handling for the grid splitting [S1].
*   **Degenerate Input:** If the density field is empty, no samples will be generated. If the threshold $\epsilon$ is too high, no splitting occurs; if too low, excessive samples are generated [S1].
*   **Transmittance Approximation:** Using $\tau_{x_j v}$ for all points in the sphere ignores transmittance variations within the sphere, which may lead to inaccurate error estimates in highly absorbing media [S1].

## Operating characteristics
*   **Time Complexity:** The method is iterative. Each iteration involves volume tracing for $O(N)$ samples, where $N$ is the number of samples. Splitting increases $N$ by a factor related to the grid density. The GPU implementation uses scan primitives for efficient filtering and splitting [S1].
*   **Memory Complexity:** Requires storage for the sample set and the 3D grid information buffer. The buffer size is proportional to the grid resolution (e.g., $64^3$) [S1].
*   **Measured Performance:** In a spot light example, starting from 541 samples, the method produced 2705 samples to capture sharp shading variations [S1].
*   **Strengths:** Accurately captures sharp shading variations (e.g., light shafts) that sparse sampling misses. Adaptive nature reduces rendering artifacts like jittering [S1].
*   **Limitations:** Computationally expensive due to volume tracing for each sample and its neighbors. The approximation of transmittance may reduce accuracy in dense media [S1].
*   **When to Use:** When rendering inhomogeneous media with sharp shading variations and sufficient computational resources for adaptive sampling [S1].
*   **When Not to Use:** For simple, homogeneous media or when real-time performance is critical and adaptive sampling is too slow [S1].

## Background
*   **Origin:** The method was introduced in "Gradient-based Interpolation and Sampling for Real-time Rendering of Inhomogeneous, Single-scattering Media" [S1].
*   **History:** It builds on previous work in gradient-based interpolation [S1]. The method operates in a top-down manner, similar to [Ramamoorthi et al. 2007], but differs by measuring gradients using multiple samples in a valid local neighborhood [S1].
*   **Usage:** Used in real-time rendering of inhomogeneous, single-scattering media [S1].

## Sources
[S1] Gradient‐based Interpolation and Sampling for Real‐time Rendering of Inhomogeneous, Single‐scattering Media, https://doi.org/10.1111/j.1467-8659.2008.01343.x, fetched 2026-10-27
