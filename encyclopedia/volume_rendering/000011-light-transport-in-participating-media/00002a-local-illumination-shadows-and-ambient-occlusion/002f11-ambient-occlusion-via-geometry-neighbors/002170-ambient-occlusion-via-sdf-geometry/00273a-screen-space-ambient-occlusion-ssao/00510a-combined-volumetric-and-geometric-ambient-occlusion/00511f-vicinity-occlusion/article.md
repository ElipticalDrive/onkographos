# Vicinity Occlusion

Also called: geometric occlusion term, high-frequency geometric details occlusion

## Overview
Vicinity occlusion is an approximate geometric ambient occlusion term designed to capture high-frequency surface-to-surface occlusion effects on tubular geometric structures (such as DTI fiber tracts) within a volume rendering pipeline [S1]. It computes occlusion by rendering a full screen quad and comparing the depth of each pixel with neighboring pixels in a geometric depth buffer [S1]. Unlike standard Screen-Space Ambient Occlusion (SSAO) techniques that cover the entire screen, this method is computed only for pixels containing geometric structures, leveraging the sparsity of the geometry to minimize performance impact [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A geometric linear depth buffer (`geom_linear_depth`) containing view-space depth values for the geometric structures [S1].
*   **Output:** A scalar occlusion value per pixel, ranging from 0 (fully occluded) to 1 (fully visible) [S1].

**Data Structures**
*   `geom_linear_depth`: A 2D texture storing linear view-space depth values for the geometric primitives [S1].
*   Sample positions $\vec{p}_i$: Offsets within the vicinity occlusion extent, typically generated using a Poisson disk distribution [S1].

**Parameters**
*   `depth_bias`: A scalar value added to the surface depth to prevent self-occlusion artifacts [S1].
*   $l_{vicinity}$: A "virtual" slice distance specified by the user to compute the vicinity occlusion extent, decoupling the qualitative appearance from the volumetric occlusion effects [S1]. Typical value: $0.005$ [S1].
*   $\theta$: The cone opening angle for the Depth-of-Structure (DOS) model, determining the radius of influence for occluders [S1]. Typical values: $25^\circ, 45^\circ, 65^\circ$ [S1].
*   $N$: The number of samples used to approximate the occlusion. Up to 118 samples using a Poisson distribution were used in the source implementation [S1].

**Formulas**
The view space radius of influence $r_{vicinity\_occl}$ is calculated based on the virtual slice distance and the cone angle:
$$ r_{vicinity\_occl} = l_{vicinity} \cdot \tan(\theta) $$
[S1]

The radius of influence is expressed as a homogeneous point in view space and projected into clip space using the projection matrix $P$, then transformed into texture coordinate space using a scale and bias matrix $T$ [S1]. The resulting texture coordinate space radius of influence $\vec{R}_c$ is:
$$ \vec{R}_c = \begin{pmatrix} x_t \\ y_t \\ z_t \\ w_t \end{pmatrix} = T \cdot P \cdot \begin{pmatrix} r_{vicinity\_occl}(z_{surface}) \\ r_{vicinity\_occl}(z_{surface}) \\ z_{surface} \\ 1 \end{pmatrix} $$
[S1]

The texture space radius of influence $\vec{R}_t$ is obtained via perspective division:
$$ \vec{R}_t = \begin{pmatrix} x_t/w_t \\ y_t/w_t \end{pmatrix} $$
[S1]

The texture space sample position $\vec{t}_i$ for the $i$-th sample is computed by adding the sample offset $\vec{p}_i$ to the projected texture coordinate of the current fragment $\vec{f}_t$:
$$ \vec{t}_i = \vec{f}_t + \vec{p}_i $$
[S1]

**Algorithm Steps**
1.  Initialize the occlusion counter to 0 [S1].
2.  Load the surface depth $z_{surface}$ from the geometric linear depth texture at the current fragment's texture coordinate [S1].
3.  Calculate the biased depth by adding `depth_bias` to $z_{surface}$ to avoid self-occlusion [S1].
4.  Generate $N$ sample offsets $\vec{p}_i$ within the vicinity occlusion extent $\vec{R}_t$ [S1].
5.  For each sample offset $\vec{p}_i$:
    a. Project the sample position into texture space $\vec{t}_i$ using $z_{surface}$ and the precomputed $\vec{R}_t$ (via $\vec{f}_t + \vec{p}_i$) [S1].
    b. Sample the geometric linear depth buffer at $\vec{t}_i$ to obtain `depth_sample` [S1].
    c. If `depth_sample` > 0 and `depth_sample` < biased depth, increment the occlusion counter [S1].
6.  Return the final occlusion value as $1 - \frac{\text{occlusion}}{N}$ [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

def VicinityOcclusion(geom_linear_depth_texture, fragment_texture_coord, z_surface, depth_bias, R_t, num_samples):
    """
    Computes the vicinity occlusion term for a geometric surface.
    R_t is the texture space radius of influence (vec2), precomputed per fragment.
    """
    occlusion_count = 0
    biased_depth = z_surface + depth_bias

    for _ in range(num_samples):
        # Generate sample offset p_i uniformly within the extent R_t
        # R_t defines an ellipse in texture space if aspect ratios differ
        angle = random.uniform(0, 2 * math.pi)
        r = math.sqrt(random.uniform(0, 1))
        p_i_x = r * R_t[0] * math.cos(angle)
        p_i_y = r * R_t[1] * math.sin(angle)

        # Compute texture coordinate t_i = f_t + p_i
        t_i_x = fragment_texture_coord[0] + p_i_x
        t_i_y = fragment_texture_coord[1] + p_i_y

        # Sample depth buffer
        depth_sample = geom_linear_depth_texture.sample(t_i_x, t_i_y)

        # Check occlusion: depth_sample > 0 and depth_sample < biased_depth
        if depth_sample > 0 and depth_sample < biased_depth:
            occlusion_count += 1

    return 1.0 - (occlusion_count / num_samples)
```

**Edge Cases and Pitfalls**
*   **Self-occlusion:** Must be avoided by offsetting the depth value of the current pixel with a `depth_bias` [S1].
*   **Aspect Ratio Distortion:** The transformation of a circular radius of influence from view space into image space results in an ellipse if the projection matrix aspect ratio does not correspond to the viewport aspect ratio [S1].
*   **Performance:** While SSAO techniques often cover the entire screen, this method is only computed for pixels containing geometric structures, reducing the performance impact due to the sparsity of DTI fiber tracts [S1].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of samples $N$ per pixel [G]. The source implementation used up to 118 samples [S1].
*   **Memory Complexity:** Requires storage for the geometric linear depth buffer [S1].
*   **Performance:** The method is less performance-limiting than full-screen SSAO because it is only computed for pixels containing geometric structures, which are sparse in the context of DTI fiber tracts [S1].
*   **Strengths:** Captures high-frequency surface-to-surface occlusion effects on tubular structures; decouples qualitative appearance from volumetric occlusion via $l_{vicinity}$ [S1].
*   **Limitations:** Only applicable to geometric structures; relies on the availability of a geometric depth buffer [S1].

## Background
Vicinity occlusion was introduced in the context of combined volumetric and geometric ambient occlusion for visualizing Diffusion Tensor Imaging (DTI) fiber tracts [S1]. It builds on the Depth-of-Structure (DOS) model, using a cone opening angle $\theta$ to define the radius of influence for occluders [S1]. The method uses a "virtual" slice distance $l_{vicinity}$ to couple the vicinity occlusion appearance with volumetric occlusion effects, rather than letting users specify the radius directly [S1].

## Sources
[S1] Ambient Occlusion Effects for Combined Volumes and Tubular Geometry, https://doi.org/10.1109/tvcg.2012.306, fetched 2026-11-08
