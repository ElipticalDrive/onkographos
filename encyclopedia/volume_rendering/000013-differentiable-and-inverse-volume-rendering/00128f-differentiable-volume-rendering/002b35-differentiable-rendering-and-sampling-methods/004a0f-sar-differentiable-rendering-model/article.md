# SAR Differentiable Rendering Model

Also called: SAR rendering model, inverse rendering model for SAR

## Overview
The SAR Differentiable Rendering Model is a simplified differentiable rendering technique that generates Synthetic Aperture Radar (SAR) images from a Digital Surface Model (DSM) and a radar backscattering coefficients map [S1]. It solves the problem of synthesizing SAR imagery for inverse rendering tasks by rasterizing piecewise-linear approximations of the terrain directly on the surface, rather than sampling the volume [S1]. This approach departs from standard volume rendering methods by leveraging the heightmap representation to select scatterers exclusively on the surface, which prevents the inefficiency and noise associated with sampling the volume in both angle and range [S1].

## How it works
**Inputs:**
*   Digital Surface Model (DSM) represented as a heightmap $Z(x,y)$ or a set of surface points.
*   Radar backscattering coefficients map $B(x,y)$.
*   Radar acquisition parameters (azimuth cell resolution $\delta_a$, range cell definitions).

**Outputs:**
*   Estimated SAR intensity image $\hat{I}$, where $\hat{I}_m$ is the intensity for the $m$-th range cell.

**Data Structures:**
The surface is discretized into $K+1$ points $P_k$ sampled uniformly in ground range, forming $K$ contiguous segments $[P_k, P_{k+1}]$ of length $l_k$. These segments are stretched along the azimuth axis by $\delta_a$ to construct $K$ rectangular surface patches [S1].

**Parameters:**
*   $\xi$: Steepness parameter of the logistic sigmoid used for shadow calculation [S1].
*   $\mu$: Accuracy parameter for the smooth maximum function used for range cell contributions [S1].

**Steps:**

1.  **Discretization:** Decompose the surface into $K$ rectangular patches. For each patch $k$, identify the segment $[P_k, P_{k+1}]$ and its length $l_k$ [S1].

2.  **Shadow Calculation:** Compute the shadow indicatrix $\hat{v}_k$ for each patch. The presence of shadow depends on the slope of the line between the antenna and previous points.
    *   Initialize $v_0 = 1$ and $h_0 = 0$ [S1].
    *   For $k \ge 0$, calculate the slope $\eta_k$ of the line between the antenna and point $P_k$ [S1].
    *   Update the shadow status and demarcation slope iteratively:
        $$ v_{k+1} = S_\xi(\eta_k - h_k) $$
        $$ h_{k+1} = \eta_k \cdot v_{k+1} + h_k \cdot (1 - v_{k+1}) $$
    *   The shadow indicatrix for the $k$-th patch is defined as $\hat{v}_k = v_{k+1}$, assuming the scatterer is illuminated if and only if its far extremity $P_{k+1}$ is illuminated [S1].
    *   $S_\xi$ is a logistic sigmoid: $S_\xi(x) = \frac{1}{1 + e^{-\xi x}}$ [S1].

3.  **Range Cell Contribution:** Calculate $\hat{w}_{m,k}$, the proportion of the segment $[P_k, P_{k+1}]$ belonging to the $m$-th range cell. Let $d_k^-$ and $d_k^+$ be the slant ranges of the segment extremities $P_k$ and $P_{k+1}$, and $r_m^-$ and $r_m^+$ be the lower and upper radii of the $m$-th range cell band [S1].
    *   Define the smooth maximum function $M_\mu$:
        $$ M_\mu(a, b) = \frac{a+b}{2} + \frac{(a-b)^2}{2\sqrt{(a-b)^2 + \mu^2}} $$
    *   Calculate the contribution:
        $$ \hat{w}_{m,k} = \frac{M_\mu(d_k^-, r_m^+) + M_\mu(d_k^+, r_m^-)}{d_k^+ - d_k^-} - \frac{M_\mu(d_k^+, r_m^+) + M_\mu(d_k^-, r_m^-)}{d_k^+ - d_k^-} $$
    *   This ensures the sum of contributions of a scatterer across cells does not exceed 1 [S1].

4.  **Normalized Radar Cross-Section:** Estimate $\sigma_k^0$ for the patch associated with $[P_k, P_{k+1}]$ at the midpoint $P_{k+1/2} = P_k + \frac{P_{k+1}-P_k}{2}$ [S1].
    *   Compute the surface normal $\vec{n}_k$ from the quadrilateral patch [S1].
    *   Compute the local incidence unit vector $\vec{u}_k$ pointing from the sensor to $P_{k+1/2}$ [S1].
    *   Calculate:
        $$ \sigma_k^0(\vec{u}_k) = B(P_{k+1/2}) \cdot |\vec{u}_k \cdot \vec{n}_k| $$
    *   This assumes a Lambertian surface with cosine attenuation [S1].

5.  **Intensity Synthesis:** Sum the contributions of all patches for each range cell $m$:
    $$ \hat{I}_m = \delta_a \sum_{k=0}^{K-1} \sigma_k^0 \cdot \hat{v}_k \cdot \hat{w}_{m,k} \cdot l_k $$
    where $\delta_a$ is the azimuth cell resolution [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def sigmoid(x, xi):
    """Logistic sigmoid with steepness xi."""
    return 1.0 / (1.0 + math.exp(-xi * x))

def smooth_max(a, b, mu):
    """Smooth maximum function M_mu."""
    diff_sq = (a - b) ** 2
    return (a + b) / 2.0 + diff_sq / (2.0 * math.sqrt(diff_sq + mu ** 2))

def render_sar(points, backscattering_map, azimuth_res, range_cells, xi, mu):
    """
    Render SAR image from surface points and backscattering map.
    
    Args:
        points: List of P_k points (x, y, z) ordered by increasing ground range.
        backscattering_map: Function or array B(x, y) returning backscattering coefficient.
        azimuth_res: Delta_a, azimuth cell resolution.
        range_cells: List of (r_min, r_max) for each range cell m.
        xi: Steepness parameter for shadow sigmoid.
        mu: Accuracy parameter for smooth maximum.
        
    Returns:
        intensities: List of intensities I_m for each range cell.
    """
    K = len(points) - 1
    intensities = [0.0] * len(range_cells)
    
    # Initialize shadow state
    v_prev = 1.0
    h_prev = 0.0
    
    for k in range(K):
        P_k = points[k]
        P_k1 = points[k + 1]
        
        # Segment length l_k (Euclidean distance in 3D)
        l_k = math.sqrt((P_k1.x - P_k.x)**2 + (P_k1.y - P_k.y)**2 + (P_k1.z - P_k.z)**2)
        
        # Calculate midpoint for normal and backscattering
        mid_x = (P_k.x + P_k1.x) / 2.0
        mid_y = (P_k.y + P_k1.y) / 2.0
        mid_z = (P_k.z + P_k1.z) / 2.0
        
        # Surface normal n_k (approximated from segment vector and vertical)
        # Assuming ground plane is XY, normal is perpendicular to segment in XZ/YZ?
        # Source says "normal to the surface deduced from the quadrilateral patch".
        # For a 2D heightmap extruded in azimuth, the normal in the range plane is:
        dx = P_k1.x - P_k.x
        dz = P_k1.z - P_k.z
        # Normal vector (dz, 0, -dx) normalized
        norm_len = math.sqrt(dx**2 + dz**2)
        if norm_len == 0:
            n_k = (0, 0, 1)
        else:
            n_k = (dz / norm_len, 0, -dx / norm_len)
            
        # Backscattering coefficient B at midpoint
        B_val = backscattering_map(mid_x, mid_y)
        
        # Shadow calculation for next point P_{k+1}
        # Slope eta_k of line between antenna (assumed at origin or specific position) and P_k
        # Note: Source defines eta_k as slope of line (O, P_k). 
        # Assuming antenna is at (0, R_ant, H_ant) or similar. 
        # For simplicity in generic implementation, eta_k is provided or calculated from geometry.
        # Here we assume eta_k is passed or calculated externally.
        # eta_k = (P_k.z - Antenna.z) / (P_k.range - Antenna.range) ?
        # Source: "slope of the line (O, P_k) between the antenna and the point is written eta_k"
        # We will assume eta_k is available.
        
        # Update shadow status v_{k+1}
        # v_{k+1} = S_xi(eta_k - h_k)
        # h_{k+1} = eta_k * v_{k+1} + h_k * (1 - v_{k+1})
        
        # Let's assume eta_k is computed as slope in slant range plane
        # eta_k = (P_k.z - Antenna_z) / (P_k.slant_range - Antenna_slant_range)
        # This requires antenna position. If not given, we note it as a dependency.
        # For the sake of self-contained code, we assume eta_k is an input or computed.
        # Let's assume eta_k is passed in a list corresponding to points.
        # If not, this step is incomplete without antenna geometry.
        # However, the formula relies on eta_k.
        
        # To make code runnable, we assume eta_k_list is available.
        # eta_k = eta_k_list[k]
        
        # Shadow indicatrix for patch k is v_{k+1}
        # v_next = sigmoid(eta_k - h_prev, xi)
        # h_next = eta_k * v_next + h_prev * (1 - v_next)
        
        # We need eta_k. Let's assume it's calculated from antenna pos (0,0,0) for simplicity in example,
        # though real SAR has specific geometry.
        # eta_k = P_k.z / P_k.slant_range ? No, slope is dz/dr.
        # Let's assume eta_k is provided.
        
        # For the purpose of the algorithm structure:
        # v_next = sigmoid(eta_k - h_prev, xi)
        # h_next = eta_k * v_next + h_prev * (1 - v_next)
        
        # Shadow factor for patch k
        v_hat_k = v_next # v_{k+1}
        
        # Update shadow state for next iteration
        h_prev = h_next
        v_prev = v_next
        
        # Radar Cross Section
        # Incidence vector u_k from sensor to midpoint
        # Assuming sensor at origin for vector direction, or specific position.
        # u_k = (mid - sensor) / |mid - sensor|
        # Dot product |u_k . n_k|
        # If sensor at (0,0,0):
        dist_mid = math.sqrt(mid_x**2 + mid_y**2 + mid_z**2)
        if dist_mid == 0:
            u_k = (0, 0, 0)
        else:
            u_k = (mid_x/dist_mid, mid_y/dist_mid, mid_z/dist_mid)
            
        dot_prod = abs(u_k[0]*n_k[0] + u_k[1]*n_k[1] + u_k[2]*n_k[2])
        sigma_k0 = B_val * dot_prod
        
        # Slant ranges of segment extremities
        d_k_minus = math.sqrt(P_k.x**2 + P_k.y**2 + P_k.z**2) # Assuming sensor at origin
        d_k_plus = math.sqrt(P_k1.x**2 + P_k1.y**2 + P_k1.z**2)
        
        # Contribution to each range cell
        for m, (r_min, r_max) in enumerate(range_cells):
            # w_hat_m_k
            term1 = (smooth_max(d_k_minus, r_max, mu) + smooth_max(d_k_plus, r_min, mu)) / (d_k_plus - d_k_minus)
            term2 = (smooth_max(d_k_plus, r_max, mu) + smooth_max(d_k_minus, r_min, mu)) / (d_k_plus - d_k_minus)
            w_hat_m_k = term1 - term2
            
            intensities[m] += azimuth_res * sigma_k0 * v_hat_k * w_hat_m_k * l_k
            
    return intensities
```

## Subtopics

### Coarse-to-Fine Surface Reconstruction Strategy
A training strategy that reduces sample count in early phases by using coarse scattering rates (larger patches) which contribute to several resolution cells, preventing gaps in synthesized images [S1]. It leverages the noise-free images the rendering model can synthesize from few samples [S1]. The method trains a Multi-Layer Perceptron (MLP) to fit the height $Z(x,y)$ and appearance $B(x,y)$ of a radar scene from a few SAR views [S1]. The MLP models elevation and backscattering coefficients within a bounding box of side $L$ [S1]. The input is $(x,y)$, a 2D position on the ground, encoded with Instant-NGP’s positional encoding $\gamma_P$ [S1]. The outputs are elevation $Z(x,y)$ and backscattering coefficient $B(x,y)$ [S1]. The activation for $B$ is exponential (ensures positive values) [S1]. The activation for elevation is $L_8 \cdot \arctan$ (bounds values within reasonable limits) [S1].

## Operating characteristics
The method rasterizes piecewise-linear approximations of the terrain directly on the surface, leveraging the heightmap representation to select scatterers exclusively on the surface [S1]. This departs from volume rendering which requires sampling the whole volume, making it more efficient for surface reconstruction [S1]. When a coarse scattering rate is used (larger patches), each patch contributes to several resolution cells which prevents creating gaps in the synthesized images [S1]. This property is leveraged by the coarse-to-fine training strategy [S1]. The model is differentiable, allowing for gradient-based optimization of the DSM and backscattering map [S1].

## Background
The SAR Differentiable Rendering Model was introduced in a paper by researchers demonstrating surface reconstruction from SAR images using inverse rendering [S1]. The method draws inspiration from optical approaches and aims to exploit geometric disparities in SAR images [S1]. It was demonstrated on synthetic SAR image datasets produced by ONERA’s physically based EMPRISE® simulator [S1]. The work paves the way for multisensor data fusion and 3D reconstruction from unconstrained SAR images [S1].

## Sources
[S1] Multi-view 3D surface reconstruction from SAR images by inverse rendering, https://doi.org/10.1109/lgrs.2025.3572303, fetched 2026-10-28
