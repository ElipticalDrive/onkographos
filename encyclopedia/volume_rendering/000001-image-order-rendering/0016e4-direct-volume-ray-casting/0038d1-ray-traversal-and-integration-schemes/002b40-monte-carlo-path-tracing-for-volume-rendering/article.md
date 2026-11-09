# Monte Carlo Path Tracing for Volume Rendering

Also called: MCSRender, Monte Carlo Scattering Rendering

## Overview
Monte Carlo Path Tracing for Volume Rendering (MCSRender) is a stochastic method for simulating light transport within participating media by tracing individual scattering paths rather than integrating along fixed rays [S1, unofficial]. It differs from deterministic ray marching by using distance sampling to select scattering events and accumulating results over multiple frames via an exponential moving average to reduce noise [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
*   Camera parameters: Model-View-Projection (MVP) matrix and its inverse.
*   Volume data: A voxel grid or spatial structure supporting trilinear interpolation of density/scalar values.
*   Transfer function: Maps scalar density values to color $C$ and opacity $\alpha$.
*   Environment map: Provides incident light color for scattering directions.
*   Uniform parameters:
    *   $u_{\sigma_{max}}$: Maximum scattering coefficient for distance sampling [S1, unofficial].
    *   $u_{\alpha_{corr}}$: Alpha correction factor [S1, unofficial].
    *   $u_{offset}$: Random seed offset [S1, unofficial].
    *   $u_{inv\_frame\_num}$: Inverse of the current frame number for accumulation [S1, unofficial].

**Outputs:**
*   Final pixel color $oColor$ accumulated over frames [S1, unofficial].

### Data Structures
*   **Ray**: Defined by origin $\mathbf{v}_{RayFrom}$ and direction/end $\mathbf{v}_{RayTo}$ [S1, unofficial].
*   **Bounds**: Intersection interval $[t_{bounds.x}, t_{bounds.y}]$ with the volume domain (typically an AABB) [S1, unofficial].
*   **Accumulator**: A texture buffer storing the running average color, initialized to zero [S1, unofficial].

### Algorithm Steps

1.  **Ray Generation**: Unproject the screen pixel position using the inverse MVP matrix to obtain the ray origin $\mathbf{v}_{RayFrom}$ and a point on the far plane $\mathbf{v}_{RayTo}$ [S1, unofficial].
2.  **Domain Intersection**: Intersect the ray with the volume bounding box to compute $t_{bounds}$ [S1, unofficial].
    *   If $t_{bounds.x} \ge t_{bounds.y}$, the ray misses the volume. Sample the environment map at the ray direction and output as the frame color [S1, unofficial].
3.  **Segment Setup**: Calculate the ray segment within the volume: $\mathbf{from} = \mathbf{v}_{RayFrom} + t_{bounds.x} \cdot (\mathbf{v}_{RayTo} - \mathbf{v}_{RayFrom})$ and $\mathbf{to} = \mathbf{v}_{RayFrom} + t_{bounds.y} \cdot (\mathbf{v}_{RayTo} - \mathbf{v}_{RayFrom})$. Compute $maxDistance = \|\mathbf{to} - \mathbf{from}\|$ [S1, unofficial].
4.  **Seed Initialization**: Generate a random seed based on pixel position and frame offset: $\mathbf{seed} = \mathbf{v}_{Position} + \text{rand}(\mathbf{u}_{offset})$ [S1, unofficial].
5.  **First Scattering Event**:
    *   Call `sampleDistance(from, to, seed)` to find the distance $dist$ to the first scattering event [S1, unofficial].
    *   If $dist > maxDistance$, sample the environment map at the ray direction and output [S1, unofficial].
    *   Update the current position: $\mathbf{from} = \text{mix}(\mathbf{from}, \mathbf{to}, dist / maxDistance)$ [S1, unofficial].
6.  **Scattering Direction**: Determine the scattering direction $\mathbf{u}_{ScatteringDirection}$ (e.g., diffuse reflection) [S1, unofficial].
7.  **Secondary Ray Setup**: Intersect a new ray starting at $\mathbf{from}$ in direction $\mathbf{u}_{ScatteringDirection}$ with the volume bounds to get new $t_{bounds}$ and $\mathbf{to}$ [S1, unofficial].
8.  **Lighting Calculation**:
    *   Sample volume color at $\mathbf{from}$: $diffuseColor = \text{sampleVolumeColor}(\mathbf{from})$ [S1, unofficial].
    *   Sample light color from the environment map in direction $\mathbf{u}_{ScatteringDirection}$: $lightColor = \text{sampleEnvironmentMap}(\mathbf{u}_{ScatteringDirection})$ [S1, unofficial].
    *   Calculate transmittance from $\mathbf{from}$ to $\mathbf{to}$ using `sampleTransmittance(from, to, seed)` [S1, unofficial].
9.  **Frame Color**: Compute the contribution for this frame: $oColor = diffuseColor \times lightColor \times transmittance$ [S1, unofficial].
10. **Accumulation**: Update the accumulator using an exponential moving average:
    $$ acc = \text{texture}(u_{Accumulator}, v_{Position}) $$
    $$ frame = oColor $$
    $$ oColor = acc + (frame - acc) \times u_{inv\_frame\_num} $$
    [S1, unofficial].

### Subroutines

#### `sampleDistance(from, to, seed)`
Determines the distance to the next scattering event using inhomogeneous medium sampling [S1, unofficial].

1.  Initialize $dist = 0.0$, $invSigmaMax = 1.0 / u_{\sigma_{max}}$, $invMaxDistance = 1.0 / maxDistance$ [S1, unofficial].
2.  Loop:
    a. Generate random number $r_1$ from seed [S1, unofficial].
    b. Update distance: $dist \leftarrow dist - \ln(1.0 - r_1) \times invSigmaMax$ [S1, unofficial].
    c. If $dist > maxDistance$, break and return $dist$ [S1, unofficial].
    d. Calculate sampling position: $\mathbf{samplingPosition} = \text{mix}(\mathbf{from}, \mathbf{to}, dist \times invMaxDistance)$ [S1, unofficial].
    e. Sample opacity: $\alpha_{sample} = \text{transferSample}(\mathbf{samplingPosition}).a \times u_{\alpha_{corr}}$ [S1, unofficial].
    f. Generate random number $r_2$ from seed [S1, unofficial].
    g. If $r_2 < \alpha_{sample} \times invSigmaMax$, break and return $dist$ [S1, unofficial].
3.  Return $dist$ [S1, unofficial].

#### `sampleTransmittance(from, to, seed)`
Calculates the transmittance along the ray segment using the same distance sampling logic [S1, unofficial].

1.  Initialize $transmittance = 1.0$, $dist = 0.0$, $invSigmaMax = 1.0 / u_{\sigma_{max}}$, $invMaxDistance = 1.0 / maxDistance$ [S1, unofficial].
2.  Loop:
    a. Generate random number $r_1$ from seed [S1, unofficial].
    b. Update distance: $dist \leftarrow dist - \ln(1.0 - r_1) \times invSigmaMax$ [S1, unofficial].
    c. If $dist > maxDistance$, break [S1, unofficial].
    d. Calculate sampling position: $\mathbf{samplingPosition} = \text{mix}(\mathbf{from}, \mathbf{to}, dist \times invMaxDistance)$ [S1, unofficial].
    e. Sample opacity: $\alpha_{sample} = \text{transferSample}(\mathbf{samplingPosition}).a \times u_{\alpha_{corr}}$ [S1, unofficial].
    f. Update transmittance: $transmittance \leftarrow transmittance \times (1.0 - \alpha_{sample} \times invSigmaMax)$ [S1, unofficial].
    g. Generate random number $r_2$ from seed [S1, unofficial].
    h. If $r_2 < \alpha_{sample} \times invSigmaMax$, break [S1, unofficial].
3.  Return $transmittance$ [S1, unofficial].

### Edge Cases and Pitfalls
*   **Precision**: The use of $\ln(1.0 - r)$ requires $r$ to be strictly less than 1.0 to avoid $\ln(0)$. Standard random number generators should be clamped or handled to ensure $r \in [0, 1)$ [S1, unofficial].
*   **Boundary Conditions**: If the ray origin is inside the volume, $t_{bounds.x}$ may be negative. The algorithm assumes intersection with the AABB; care must be taken to clamp $t$ to $[0, \infty)$ if the ray starts inside [S1, unofficial].
*   **Noise**: As a Monte Carlo method, single-frame results are noisy. Convergence requires multiple frames and proper accumulation [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: Per frame, complexity depends on the number of scattering events per ray, which is stochastic. It is generally slower than deterministic ray marching for single-frame accuracy but converges to the correct solution with more samples [S1, unofficial].
*   **Memory Complexity**: Requires storage for the volume data, environment map, and an accumulator buffer [S1, unofficial].
*   **Strengths**: Handles complex scattering and lighting interactions naturally; unbiased result with sufficient samples [S1, unofficial].
*   **Limitations**: High noise in early frames; requires temporal accumulation; performance varies with scene density and scattering coefficients [S1, unofficial].
*   **When to use**: When visual fidelity and correct light transport in participating media are prioritized over single-frame performance [S1, unofficial].
*   **When not to use**: When real-time single-frame performance is critical without temporal accumulation [S1, unofficial].

## Background
The method is implemented in the `vpt-conductor` project by UL-FRI-LGM, representing a practical application of Monte Carlo path tracing for volume rendering using WebGL/OpenGL shaders [S1, unofficial]. It adapts standard path tracing concepts (distance sampling, transmittance calculation) to the specific constraints of real-time volume rendering [S1, unofficial].

## Sources
[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
