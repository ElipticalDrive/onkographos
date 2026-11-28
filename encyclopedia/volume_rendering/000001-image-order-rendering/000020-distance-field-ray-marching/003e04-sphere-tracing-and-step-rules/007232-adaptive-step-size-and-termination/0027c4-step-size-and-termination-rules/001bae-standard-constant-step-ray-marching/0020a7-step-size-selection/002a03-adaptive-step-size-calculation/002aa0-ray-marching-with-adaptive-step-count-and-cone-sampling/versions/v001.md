# Ray Marching with Adaptive Step Count and Cone Sampling

Also called: Adaptive Step Count with Cone Sampling

## Overview
This method performs volume rendering of volumetric clouds by adaptively selecting the number of ray march steps based on the viewing angle and using cone sampling to approximate light transport within the volume. It solves the problem of balancing rendering performance and visual quality in real-time atmospheric clouds, where top-down views require fewer steps than horizon views, and single-sample lighting causes banding artifacts. It differs from standard constant step ray marching by varying the step count dynamically and from sibling methods by using a fixed kernel of 6 offset vectors rotated towards the light source for density estimation along the light path.

## How it works

### Inputs and Outputs
**Inputs:**
- Ray origin $O$ and direction $D$ (normalized).
- Ray segment bounds $t_{start}$ and $t_{end}$ in world space.
- Pixel identifier (integer) for jittering.
- Time variable $T$ for wind animation.
- Light direction $L$ (normalized).
- Earth center position $C_{earth}$ (typically $(0, R_{earth}, 0)$).
- Parameters: `baseDensityFactor`, `jitterfactor`, `eccentricity`, `silver_intensity`, `silver_spread`, `brightness`.

**Outputs:**
- Accumulated color $C_{out}$ (RGB).
- Accumulated density $\alpha_{accum}$ (scalar, clamped to 1.0).

### Data Structures
The method uses a fixed kernel of 6 offset vectors. These are defined in local space and rotated to align with the light direction.
- **Kernel Offsets (Local):**
  1. $(0.1, 0.25, -0.15)$
  2. $(0.2, 0.5, 0.2)$
  3. $(-0.2, 0.1, -0.1)$
  4. $(-0.05, 0.75, 0.05)$
  5. $(-0.1, 1.0, 0.0)$
  6. $(0.0, 3.0, 0.0)$

### Algorithm Steps

1.  **Calculate Adaptive Step Count:**
    Compute the dot product of the ray direction $D$ and the world up vector $U = (0, 1, 0)$.
    $$ d = \text{dot}(D, U) $$
    Calculate the number of steps $N_{steps}$ using linear interpolation (mix) between 35 and 60 steps based on the viewing angle. Top-down views ($d \approx 1$) use 35 steps; horizon views ($d \approx 0$) use 60 steps.
    $$ N_{steps} = \lfloor \text{mix}(35.0, 60.0, 1.0 - d) \rfloor $$
    [S1, unofficial]

2.  **Initialize Variables:**
    Compute the step size:
    $$ \Delta t = \frac{t_{end} - t_{start}}{N_{steps}} $$
    Initialize transmittance $T_{trans} = 1.0$ and accumulated density $\alpha_{accum} = 0.0$. Initialize output color $C_{out} = (0, 0, 0)$.
    [S1, unofficial]

3.  **Construct Light Rotation Matrix (TBN):**
    To rotate the cone sampling kernel towards the light direction $L$, construct a rotation matrix.
    Find the axis with the largest absolute component in $L$:
    $$ \text{maxComp} = \text{argmax}_i |L_i| $$
    Define unit vectors for the axes:
    $$ V_x = (1, 0, 0), \quad V_y = (0, 1, 0), \quad V_z = (0, 0, 1) $$
    Select $V_{max}$ corresponding to $\text{maxComp}$.
    Compute the cross product to find an orthogonal vector $Z_{comp}$:
    $$ Z_{comp} = \text{cross}(L, V_{max}) $$
    Compute the X component:
    $$ X_{comp} = \text{cross}(Z_{comp}, L) $$
    Normalize $X_{comp}$ and $Z_{comp}$ if necessary (though cross products of orthogonal vectors are often normalized, the source implies direct usage). Form the rotation matrix $M_{sun}$ with columns $X_{comp}$, $L$, and $Z_{comp}$:
    $$ M_{sun} = \begin{bmatrix} X_{comp} & L & Z_{comp} \end{bmatrix} $$
    [S1, unofficial]

4.  **Precompute Cone Sampling Offsets:**
    Transform the 6 local kernel offsets $K_{local}[i]$ into world space aligned with the light:
    $$ K_{world}[i] = M_{sun} \times K_{local}[i], \quad \text{for } i \in [0, 5] $$
    [S1, unofficial]

5.  **Ray Marching Loop:**
    Iterate $j$ from $0$ to $N_{steps} - 1$:
    
    a. **Calculate Sample Position:**
       Compute current $t$:
       $$ t = t_{start} + j \cdot \Delta t $$
       
       Generate jitter offset using pixel ID and $t$ to select an index from a pre-defined or pseudo-random set (mod 16). Let this be `jitterIndex`. Retrieve `jitterLocation` (a 2D vector or similar structure derived from the index).
       
       Calculate the perturbed ray direction component:
       $$ \Delta D_x = \text{jitterLocation}.x $$
       $$ \Delta D_y = (\text{jitterLocation}.x + \text{jitterLocation}.y) \cdot \text{jitterfactor} $$
       $$ \Delta D_z = \text{jitterLocation}.y $$
       
       Update position $P$:
       $$ P = O + t \cdot (D + (\Delta D_x, \Delta D_y, \Delta D_z)) $$
       [S1, unofficial]

    b. **Sample Density:**
       Transform $P$ to relative atmospheric coordinates. Divide by 8.0 to control noise frequency.
       Skew the sample point using wind direction and time $T$:
       $$ P_{skewed} = \text{Skew}(P, \text{wind}, T) $$
       
       Calculate relative height $h$ from $P$ and $C_{earth}$.
       
       Sample low-frequency density:
       $$ \rho_{base} = \text{SampleLowFreq}(P_{skewed}, P, h, C_{earth}) \cdot \text{baseDensityFactor} $$
       
       If $\rho_{base} > 0.0$, erode with high-frequency noise:
       $$ \rho_{high} = \text{ErodeCloud}(\rho_{base} \cdot 1.4, D, P_{skewed}, h) $$
       
       Update accumulated density:
       $$ \alpha_{accum} \leftarrow \alpha_{accum} + \rho_{high} \cdot 0.5 $$
       
       Check for early termination:
       If $\alpha_{accum} \ge 1.0$, set $\alpha_{accum} = 1.0$ and break the loop.
       [S1, unofficial]

    c. **Cone Sampling for Lighting:**
       Initialize `densityAlongLight` = 0.0.
       Loop $i$ from $0$ to $5$:
       
       Calculate light sample position:
       $$ P_{light} = P + \Delta t \cdot K_{world}[i] \cdot (i + 1) $$
       *(Note: Source says `float(i)`, implying multiplication by index. If $i=0$, offset is 0.)*
       
       Sample density at $P_{light}$:
       $$ \rho_{light\_base} = \text{SampleLowFreq}(P_{light}, \dots) $$
       
       If $\rho_{light\_base} > 0.0$:
       $$ \rho_{light\_high} = \text{ErodeCloud}(\rho_{light\_base} \cdot 1.4, D, P_{light}, h) $$
       $$ \text{densityAlongLight} \leftarrow \text{densityAlongLight} + \rho_{light\_high} $$
       
       [S1, unofficial]

    d. **Compute Light Energy and Update Color:**
       Calculate total light energy using a function `GetLightEnergy` with parameters:
       - Relative height $h$
       - `densityAlongLight`
       - $\rho_{base}$
       - Henyey-Greenstein phase function value $p(\theta)$ with eccentricity `eccentricity`
       - Cosine of the angle between light and view direction
       - Step size $\Delta t$
       - Brightness multiplier `brightness`
       
       $$ E_{light} = \text{GetLightEnergy}(h, \text{densityAlongLight}, \rho_{base}, p(\theta), \Delta t, \text{brightness}) $$
       
       Update transmittance and color:
       $$ T_{trans} = \text{mix}(T_{trans}, E_{light}, 1.0 - \alpha_{accum}) $$
       $$ C_{out} \leftarrow C_{out} + T_{trans} $$
       
       [S1, unofficial]

### Parameters
- `baseDensityFactor`: Controls cloud density center intensity. Default: 0.380. [S1, unofficial]
- `jitterfactor`: Multiplier for jitter offset in Y component. Default: 1.180. [S1, unofficial]
- `eccentricity`: Henyey-Greenstein scattering parameter. Default: 0.6. [S1, unofficial]
- `silver_intensity`: Parameter for modified Henyey-Greenstein function. Default: 0.7. [S1, unofficial]
- `silver_spread`: Parameter for modified Henyey-Greenstein function. Default: 0.1. [S1, unofficial]
- `brightness`: Lighting brightness multiplier. Default: 5.0. [S1, unofficial]

### Edge Cases and Pitfalls
- **Precision:** The mixing of transmittance and light energy requires careful handling of floating-point precision to avoid banding, especially at low densities.
- **Boundaries:** The ray segment bounds $t_{start}$ and $t_{end}$ must correctly clip against the volume boundaries (e.g., sphere intersection with atmosphere).
- **Degenerate Input:** If the ray direction is parallel to the world up vector, the dot product is 1 or -1. The formula handles $d \approx 1$ (top-down) well. If $d$ is negative (viewing from below), the mix parameter $1.0-d$ exceeds 1, potentially exceeding 60 steps. The source specifies "horizon views" for $d$ near 0, implying the method is optimized for viewing from above or the horizon. [S1, unofficial]
- **Kernel Alignment:** The TBN matrix construction assumes $L$ is not parallel to the chosen `maxComp` axis. If $L$ is parallel, the cross product becomes zero. A fallback axis selection is not explicitly detailed in the source but is a standard implementation requirement. [G]

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity:** $O(N_{steps} \cdot K)$, where $N_{steps}$ is the adaptive step count (35-60) and $K=6$ is the number of cone samples. The complexity is linear with respect to the number of steps and constant with respect to the scene size (assuming texture lookups are $O(1)$).
- **Memory Complexity:** $O(1)$ auxiliary memory per pixel, excluding the noise texture/sampler state.

### Measured Performance
- Specific performance metrics (FPS, resolution) are not provided in the sources. The method is designed for real-time rendering, as indicated by the use of jittering and adaptive steps. [S1, unofficial]

### Scaling
- The method scales well with resolution as it is a per-pixel operation. The adaptive step count reduces computational load for top-down views, providing a performance benefit in scenes with significant vertical viewing angles.

### Strengths and Limitations
- **Strengths:** Reduces artifacts from single-sample lighting via cone sampling. Adaptive steps optimize performance based on viewing angle.
- **Limitations:** Fixed kernel size (6 samples) may not capture complex light transport in very dense volumes. The method relies on procedural noise, which may lack the detail of texture-based approaches. [S1, unofficial]

### When to Use
- Use for real-time volumetric cloud rendering where performance is critical and visual quality needs to be maintained across varying camera angles.

### When Not to Use
- Not suitable for offline rendering where higher fidelity light transport (e.g., multiple scattering) is required. Not suitable for scenes with dynamic light sources that change rapidly, as the TBN matrix must be recomputed per ray.

## Background

### Origin
The method is derived from the implementation in the open-source project "realtime-volumetric-cloudscapes" by erickTornero. [S1, unofficial]

### History
The technique combines adaptive step sizing, which is common in ray marching to optimize performance, with cone sampling, a technique used to approximate the integration of light through a volume by sampling along a cone aligned with the light direction. This approach helps to reduce the "staircase" artifacts and noise associated with single-sample lighting in volumetric rendering. [S1, unofficial]

### Usage
The method is used in real-time graphics applications, specifically for rendering atmospheric clouds in games or simulations. [S1, unofficial]

## Sources

[S1] erickTornero/realtime-volumetric-cloudscapes, https://github.com/erickTornero/realtime-volumetric-cloudscapes, fetched 2026-10-27 (unofficial)
