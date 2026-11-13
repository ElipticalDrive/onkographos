# Motion Blur via Sub-frame Optimization
Also called: ExBluRF Blur Formulation, Motion Blur via Subframe Accumulation, Subframe Accumulation, Temporal Accumulation for Motion Blur

## Overview
ExBluRF models motion blur in radiance fields by approximating the continuous integral of ray accumulation over the shutter open time as a finite sum of $N$ intermediate sub-frames, jointly optimizing sub-frame camera poses with the radiance field [S1]. It differs from prior works that model blur using global camera trajectories or light fields by optimizing each training view’s latent camera trajectory independently, allowing for arbitrary but smooth local motion [S1].

## How it works
**Inputs and Outputs**
*   **Inputs:** Training views with known sharp images, shutter open time $\tau_o$, shutter close time $\tau_c$, number of sub-frames $N$, and Bézier curve order $M$ [S1].
*   **Outputs:** Blurred RGB color $\hat{\mathbf{C}}_B(\mathbf{x})$ for a pixel at location $\mathbf{x}$, and optimized control points $\hat{\mathbf{p}}_j$ defining the camera trajectory [S1].

**Data Structures**
*   **Control Points:** A set of $M+1$ control points $\{\hat{\mathbf{p}}_j\}_{j=0}^{M}$, where each $\hat{\mathbf{p}}_j$ is an element of the Lie algebra $se(3)$, representing the 6-DOF camera motion parameters [S1].
*   **Sub-frame Times:** A set of $N$ time steps $\tau_i$ for $i=1 \dots N$ [S1].

**Parameters**
*   $N$: Number of intermediate sub-frames. Typical value: 21 [S1].
*   $M$: Order of the Bézier curve. Typical value: 7 [S1].

**Algorithm**
1.  **Initialization:** Initialize the control points $\hat{\mathbf{p}}_j$ using camera poses estimated by COLMAP [S1].
2.  **Time Discretization:** Compute the time $\tau_i$ for each sub-frame $i$ using linear interpolation between shutter open $\tau_o$ and close $\tau_c$:
    $$ \tau_i = \tau_o + \frac{i-1}{N-1} (\tau_c - \tau_o) $$
    where $\tau_i$ is the time of the $i$-th sub-frame [S1].
3.  **Normalized Time:** Compute the normalized time $\tau'_i$ for each sub-frame:
    $$ \tau'_i = \frac{\tau_i - \tau_o}{\tau_c - \tau_o} $$
    where $\tau'_i \in [0, 1]$ [S1].
4.  **Pose Interpolation:** Compute the Lie algebra representation of the camera pose $\mathbf{p}_{\tau_i}$ at time $\tau_i$ using the Bézier curve formulation:
    $$ \mathbf{p}_{\tau_i} = \sum_{j=0}^{M} \binom{M}{j} (1-\tau'_i)^{M-j} (\tau'_i)^j \cdot \hat{\mathbf{p}}_j $$
    where $\mathbf{p}_{\tau_i}$ is the pose at $\tau_i$, $M$ is the curve order, and $\hat{\mathbf{p}}_j$ are the control points [S1].
5.  **Ray Generation:** For each sub-frame $i$, generate rays $\mathbf{r}_{\tau_i}(\mathbf{x})$ originating from the 3D location of the camera at pose $\mathbf{p}_{\tau_i}$ and passing through pixel $\mathbf{x}$ with direction $\mathbf{d}_{\tau_i}$ [S1].
6.  **Accumulation:** Compute the blurred RGB color $\hat{\mathbf{C}}_B(\mathbf{x})$ by accumulating radiance along the rays for all $N$ sub-frames:
    $$ \hat{\mathbf{C}}_B(\mathbf{x}) = \frac{1}{N} \sum_{i=1}^{N} \hat{\mathbf{C}}(\mathbf{r}_{\tau_i}(\mathbf{x})) $$
    where $\hat{\mathbf{C}}(\mathbf{r}_{\tau_i}(\mathbf{x}))$ is the radiance accumulated along ray $\mathbf{r}_{\tau_i}(\mathbf{x})$ using the radiance field [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def factorial(n):
    """Compute factorial of n."""
    result = 1
    for k in range(2, n + 1):
        result *= k
    return result

def exp_map(se3_vec):
    """Convert a 6-DOF Lie algebra vector se(3) to an SE(3) 4x4 transformation matrix [G]."""
    # Standard Lie group exponential map implementation
    return [[1, 0, 0, 0], [0, 1, 0, 0], [0, 0, 1, 0], [0, 0, 0, 1]]

def generate_ray(pose_matrix, pixel_coords):
    """Generate a ray origin and direction from a camera pose and pixel coordinates [G]."""
    # Standard pinhole camera ray generation
    return {"origin": [0, 0, 0], "direction": [0, 0, 1]}

def compute_blur_color(x, tau_o, tau_c, N, M, control_points, radiance_field):
    """Compute the blurred color for pixel x using ExBluRF formulation."""
    if tau_o == tau_c:
        raise ValueError("Shutter open and close times must differ to avoid division by zero.")
    blurred_color = 0.0
    for i in range(1, N + 1):
        tau_i = tau_o + (i - 1) / (N - 1) * (tau_c - tau_o)
        tau_prime_i = (tau_i - tau_o) / (tau_c - tau_o)
        pose_se3 = [0.0] * 6
        for j in range(M + 1):
            binom = factorial(M) / (factorial(j) * factorial(M - j))
            bernstein = binom * ((1 - tau_prime_i) ** (M - j)) * (tau_prime_i ** j)
            for k in range(6):
                pose_se3[k] += bernstein * control_points[j][k]
        pose_matrix = exp_map(pose_se3)
        ray = generate_ray(pose_matrix, x)
        radiance = radiance_field(ray)
        blurred_color += radiance
    return blurred_color / N
```

**Edge Cases and Pitfalls**
*   **Precision:** The accumulation of $N$ rays increases memory consumption and training time by $O(N)$ [S1]. Extreme motion blur requires larger $N$ to keep the latent 3D scene sharp, exacerbating this cost [S1].
*   **Boundaries:** The normalized time $\tau'_i$ is defined over $[0, 1]$. If $\tau_o = \tau_c$, division by zero occurs in the normalized time calculation [G].
*   **Initialization:** While no prior assumption is strictly required, initializing control points with COLMAP estimates ensures convergence [S1].

## Operating characteristics
*   **Time Complexity:** Training time increases by a factor of $O(N)$ compared to training radiance fields on sharp images, as $N$ rays are shot to synthesize a single blurry pixel [S1].
*   **Memory Complexity:** Memory consumption increases by $O(N)$ due to the accumulation of gradients for $N$ sub-frames [S1].
*   **Performance:** Tested on a single NVIDIA RTX Quadro RTX 8000 with 200k iterations and a batch size of 25k rays [S1].
*   **Strengths:** Handles arbitrary but smooth camera motion without global trajectory assumptions; independently optimizes latent trajectories per view [S1].
*   **Limitations:** High computational cost for extreme motion blur; requires large $N$ for sharp reconstruction under heavy blur [S1].
*   **When to use:** When reconstructing radiance fields from extreme camera motion blur where global trajectory models are insufficient [S1].
*   **When not to use:** When computational resources are limited, as training requires $O(N)$ more memory and time [S1].

## Background
ExBluRF was introduced in the paper "ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images" [S1]. The method addresses the challenge of training MLP-based radiance fields from extreme camera motion, which requires heavy computation [S1]. It builds on the concept of differentiable rendering and Bézier curves for trajectory parameterization [S1].

## Sources
[S1] ExBluRF: Efficient Radiance Fields for Extreme Motion Blurred Images, https://arxiv.org/abs/2309.08957v3, fetched 2026-10-27
