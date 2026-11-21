# Extinction-based Ambient Occlusion

Also called: AO via extinction summation, Local AO

## Overview
Extinction-based Ambient Occlusion (AO) approximates the attenuation of diffusely reflected ambient light through local volume occlusion by summing extinction coefficients rather than tracing shadow rays [S1, unofficial]. It differs from standard DVR by replacing the constant ambient term with a spatially varying occlusion factor derived from a local neighborhood integral, and from full global illumination by ignoring multi-bounce scattering and relying on a precomputed aggregate structure for speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume dataset with extinction coefficients $\tau(t)$ and color $C_{RGB}(t)$; ambient light intensity $I_A$; local neighborhood definition $N(s)$ (typically a sphere or box centered at sample point $s$).
*   **Output:** Occluded ambient intensity $I_{AO}(s)$ and final reflected light $E_i$ for each sample point.

**Data Structures:**
*   **3D Summed Area Table (SAT):** A precomputed 3D array storing the cumulative sum of extinction coefficients. Let $SAT[x][y][z]$ store the sum of $\tau$ values in the box from $(0,0,0)$ to $(x,y,z)$. This allows $O(1)$ retrieval of the sum of $\tau$ over any axis-aligned bounding box [S1, unofficial].
*   **Weighted SAT (Optional):** If the weighting function $w(t)$ varies spatially (e.g., $1/|s-t|^2$), a separate SAT for weighted extinction may be required, or the distance weighting must be applied post-aggregation if the SAT only stores raw sums [S1, unofficial]. The source notes use a weighting function $w_{A,L}$ for the aggregate query, implying the SAT stores raw $\tau$, and weights are applied during the query or via a specific weighted aggregation strategy. Given the formula involves $1/|s-t|^2$, a simple box-sum SAT is insufficient for exact integration unless the neighborhood is discretized into weighted bins or the approximation assumes constant weight over the aggregate query volume. The source states $w_{A,L} = r^{-2}$ is the inverse square of the radius of the aggregate query, suggesting a simplified approximation where the distance is approximated by the query radius [S1, unofficial].

**Numbered Steps:**
1.  **Precompute SAT:** Construct a 3D Summed Area Table from the volume's extinction coefficient field $\tau$. For each voxel $(x,y,z)$, $SAT[x][y][z] = \tau(x,y,z) + SAT[x-1][y][z] + SAT[x][y-1][z] + SAT[x][y][z-1] - SAT[x-1][y-1][z] - SAT[x-1][y][z-1] - SAT[x][y-1][z-1] + SAT[x-1][y-1][z-1]$ [S1, unofficial].
2.  **Define Neighborhood:** For a sample point $s$, define the local neighborhood $N(s)$. In the implementation described, this is approximated by an aggregate query volume $\Omega_{A,L}$ (e.g., a sphere or box) [S1, unofficial].
3.  **Aggregate Extinction:** Query the SAT to compute the sum of extinction coefficients $\sum_{\Omega_{A,L}} \Delta^\tau_j$ over the neighborhood $\Omega_{A,L}$ [S1, unofficial].
4.  **Compute Occlusion Factor:** Calculate the local ambient occlusion intensity $I_{AO}(s)$ using the exponential attenuation model:
    $$I_{AO}(s) \approx I_A \cdot e^{-\int_{t \in \mathbb{N}(s)} \frac{\tau(t)}{|s-t|^2} dt}$$
    The source approximates the integral $\int_{t \in \mathbb{N}(s)} \frac{\tau(t)}{|s-t|^2} dt$ using the aggregated sum. Specifically, it uses a weighting factor $w_{A,L} = r^{-2}$ (inverse square of the radius of the aggregate query) applied to the sum:
    $$\text{Exponent} = w_{A,L} \cdot \sum_{\Omega_{A,L}} \Delta^\tau_j \cdot k$$
    where $k$ is a normal/gradient-dependent local illumination model factor [S1, unofficial]. Note: The source formula (8) combines this with color and light intensity.
5.  **Compute Reflected Light:** Calculate the final reflected light $E_i$ for the voxel:
    $$E_i = I_{A,L} \cdot e^{-w_{A,L} \sum_{\Omega_{A,L}} \Delta^\tau_j \cdot k} \cdot C_{RGB}$$
    where $I_{A,L}$ is the ambient or light intensity, and $C_{RGB}$ is the voxel color [S1, unofficial].

**Formulas:**
*   **Local AO Intensity:** $I_{AO}(s) \approx I_A \cdot e^{-\int_{t \in \mathbb{N}(s)} \frac{\tau(t)}{|s-t|^2} dt}$ [S1, unofficial].
*   **Reflected Light:** $E_i = I_{A,L} \cdot e^{-w_{A,L} \sum_{\Omega_{A,L}} \Delta^\tau_j \cdot k} \cdot C_{RGB}$ [S1, unofficial].
*   **Weighting Function:** $w_{A,L} = r^{-2}$, where $r$ is the radius of the aggregate query [S1, unofficial].

**Parameters:**
*   **Neighborhood Size ($r$):** The radius of the aggregate query $\Omega_{A,L}$. Not given in the sources.
*   **Ambient Intensity ($I_A$):** Proportional to the sum of all lights. Not given in the sources.
*   **Illumination Factor ($k$):** Normal/gradient dependent. Not given in the sources.

**Edge Cases and Pitfalls:**
*   **Precision:** The exponential function can underflow if the extinction sum is large, resulting in black pixels. [G] Clamp the exponent or use log-space arithmetic if necessary.
*   **Boundary Conditions:** The SAT query must handle cases where the neighborhood extends beyond the volume bounds. [G] Clamp the SAT query coordinates to the volume dimensions.
*   **Approximation Error:** The use of a constant weight $r^{-2}$ for the entire neighborhood introduces error, especially for large neighborhoods or non-spherical shapes. The source notes "virtually no difference in image quality" compared to individual sampling, but this is a specific trade-off [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_extinction_ao(volume, sat, s, r, I_A, k):
    """
    Compute extinction-based ambient occlusion for sample point s.
    
    Args:
        volume: 3D array of extinction coefficients tau.
        sat: 3D Summed Area Table of tau.
        s: Sample point (x, y, z).
        r: Radius of the aggregate query neighborhood.
        I_A: Ambient light intensity.
        k: Illumination factor.
        
    Returns:
        E_i: Reflected light intensity.
    """
    # Define the bounding box of the neighborhood Omega_A,L
    # Assuming a spherical neighborhood approximated by a bounding box for SAT query
    x_min, x_max = max(0, s.x - r), min(volume.width - 1, s.x + r)
    y_min, y_max = max(0, s.y - r), min(volume.height - 1, s.y + r)
    z_min, z_max = max(0, s.z - r), min(volume.depth - 1, s.z + r)
    
    # Query SAT to get sum of tau in the box
    # SAT query function for 3D box sum
    sum_tau = query_sat_3d(sat, x_min, y_min, z_min, x_max, y_max, z_max)
    
    # Calculate weighting factor w_A,L = r^-2
    # Note: If r is 0, handle division by zero. [G]
    if r == 0:
        w_A_L = 0.0
    else:
        w_A_L = 1.0 / (r * r)
        
    # Calculate exponent
    exponent = w_A_L * sum_tau * k
    
    # Calculate reflected light E_i
    # I_A,L is assumed to be I_A for ambient occlusion
    I_A_L = I_A
    C_RGB = volume[s.x, s.y, s.z] # Voxel color
    
    E_i = I_A_L * math.exp(-exponent) * C_RGB
    
    return E_i

def query_sat_3d(sat, x_min, y_min, z_min, x_max, y_max, z_max):
    """
    Compute sum of elements in 3D box using SAT.
    """
    def get_val(x, y, z):
        # Handle boundary conditions
        x = max(0, min(x, sat.width - 1))
        y = max(0, min(y, sat.height - 1))
        z = max(0, min(z, sat.depth - 1))
        return sat[x, y, z]
        
    sum_box = (get_val(x_max, y_max, z_max) 
               - get_val(x_min - 1, y_max, z_max) 
               - get_val(x_max, y_min - 1, z_max) 
               - get_val(x_max, y_max, z_min - 1) 
               + get_val(x_min - 1, y_min - 1, z_max) 
               + get_val(x_min - 1, y_max, z_min - 1) 
               + get_val(x_max, y_min - 1, z_min - 1) 
               - get_val(x_min - 1, y_min - 1, z_min - 1))
    return sum_box
```

## Subtopics

### TVAM Integrator
Uses a different integration strategy for volume rendering, focusing on texture-based volume accumulation. [TVAM Integrator](6814-tvam-integrator.md)

### Cinematic Rendering
Focuses on high-fidelity global illumination and soft shadows, often using path tracing, which is more expensive than extinction-based AO. [Cinematic Rendering](9473-cinematic-rendering.md)

### Compositional Rendering
Renders distinct material styles into separate layers and composites them post-hoc, rather than integrating lighting during ray casting. [Compositional Rendering](11182-compositional-rendering.md)

### Shading and Occlusion Models
A broader category that includes extinction-based AO, cone-traced shadows, and other local illumination approximations. [Shading and Occlusion Models](15337-shading-and-occlusion-models.md)

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample point for the AO calculation after $O(N)$ precomputation of the SAT, where $N$ is the number of voxels. Aggregate sampling is over 45 times faster than individually sampled neighborhoods (0.22s vs 10s) [S1, unofficial].
*   **Memory Complexity:** $O(N)$ for the SAT, where $N$ is the number of voxels.
*   **Measured Performance:** On the tested hardware and resolution, aggregate sampling took 0.22s compared to 10s for individual sampling, with virtually no difference in image quality [S1, unofficial].
*   **Strengths:** Fast approximation of local occlusion; suitable for interactive volume rendering; simple to implement with SAT.
*   **Limitations:** Approximation error due to constant weight assumption; does not account for multi-bounce scattering or complex light transport; quality depends on neighborhood size and shape approximation.
*   **When to use:** When interactive performance is required and local occlusion effects are desired without the cost of full global illumination.
*   **When not to use:** When physical accuracy is required, or when global illumination effects like color bleeding (beyond local approximation) are critical.

## Background
Extinction-based Ambient Occlusion was introduced as a fast approximation for local illumination in volume rendering, avoiding the cost of shadow ray tracing [S1, unofficial]. It builds on the concept of ambient occlusion in surface rendering but adapts it to volumes by using extinction coefficients as a proxy for opacity. The method was developed to provide a useful approximation of the effect of occlusion on diffusely reflected ambient light, which is otherwise computationally expensive to simulate with full global illumination models [S1, unofficial].

## Sources
[S1] PDF Extinction-based Shading and Illumination in GPU Volume Ray-Casting - UZH, https://www.ifi.uzh.ch/dam/jcr:ffffffff-9e7f-2af5-ffff-ffffcf9259bb/FastShadows.pdf, fetched 2026-10-30 (unofficial)
