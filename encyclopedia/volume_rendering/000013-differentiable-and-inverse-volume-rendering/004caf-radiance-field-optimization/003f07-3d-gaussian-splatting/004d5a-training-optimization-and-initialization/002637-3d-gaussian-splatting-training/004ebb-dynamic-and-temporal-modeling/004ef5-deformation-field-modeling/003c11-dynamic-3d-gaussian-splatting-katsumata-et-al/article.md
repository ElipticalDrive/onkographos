# Dynamic 3D Gaussian Splatting (Katsumata et al.)
Also called: Dynamic 3DGS, Fourier-based Dynamic 3DGS, Katsumata et al

## Overview
Dynamic 3D Gaussian Splatting (Katsumata et al.) models dynamic scenes by explicitly parameterizing the temporal evolution of 3D Gaussian positions and orientations, while keeping other Gaussian attributes (opacity, scale, spherical harmonics) static [S1, S2]. It distinguishes itself from implicit deformation networks by using continuous parametric functions (Fourier series for position, linear for rotation), ensuring that the number of learnable parameters remains independent of the number of time steps [S2]. This approach balances computational efficiency with robustness for periodic or smooth motions [S2].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of 3D Gaussians $\{G_i\}$ initialized from static reconstruction, and a sequence of images captured at various timestamps $t$.
*   **Outputs:** A dynamic radiance field where each Gaussian $G_i$ has time-dependent position $\mathbf{p}_i(t)$ and rotation $\mathbf{q}_i(t)$.

**Data Structures:**
Each Gaussian $i$ stores the following parameters, which are optimized:
1.  **Static Attributes:** Scale $\mathbf{s}_i$, Opacity $\alpha_i$, Spherical Harmonics coefficients $\mathbf{c}_i$. These are constant over time.
2.  **Dynamic Position Parameters:** Fourier series coefficients for $x, y, z$ coordinates. For frequency order $K$, each coordinate requires $2K+1$ coefficients (bias + $K$ cosine + $K$ sine terms).
3.  **Dynamic Rotation Parameters:** Linear coefficients for quaternion components. A quaternion $\mathbf{q}$ has 4 components, each modeled as a linear function of time.

**Formulas:**
The position $\mathbf{p}_i(t) = [x_i(t), y_i(t), z_i(t)]$ is modeled using a Fourier series [S2]:
$$
x_i(t) = a_{i,0}^x + \sum_{k=1}^{K} \left( a_{i,k}^x \cos(k \omega t) + b_{i,k}^x \sin(k \omega t) \right)
$$
$$
y_i(t) = a_{i,0}^y + \sum_{k=1}^{K} \left( a_{i,k}^y \cos(k \omega t) + b_{i,k}^y \sin(k \omega t) \right)
$$
$$
z_i(t) = a_{i,0}^z + \sum_{k=1}^{K} \left( a_{i,k}^z \cos(k \omega t) + b_{i,k}^z \sin(k \omega t) \right)
$$
where $\omega$ is the base frequency, and $a, b$ are learnable coefficients.

The rotation $\mathbf{q}_i(t)$ is modeled as a linear function of time [S2]:
$$
\mathbf{q}_i(t) = \mathbf{q}_{i,0} + \mathbf{m}_i t
$$
where $\mathbf{q}_{i,0}$ is the initial quaternion and $\mathbf{m}_i$ is the linear velocity quaternion. Note: The resulting quaternion must be normalized after computation to maintain validity [G].

**Numbered Steps:**
1.  **Initialization:** Initialize 3D Gaussians using standard static 3DGS initialization [S2].
2.  **Parameter Definition:** For each Gaussian, define the Fourier coefficients for position and linear coefficients for rotation. The total number of parameters is fixed regardless of the number of time steps [S2].
3.  **Time Evaluation:** For a given camera view at time $t$, compute $\mathbf{p}_i(t)$ and $\mathbf{q}_i(t)$ for all Gaussians using the formulas above.
4.  **Rendering:** Rasterize the Gaussians using the computed time-dependent position and rotation, along with static scale, opacity, and color [S2].
5.  **Loss Computation:**
    *   **Photometric Loss:** L1 or LPIPS loss between rendered and ground truth images [S2].
    *   **Optical Flow Loss:** A bidirectional optical flow loss is applied to enforce temporal consistency and guide motion [S2].
6.  **Optimization:** Update the Fourier and linear coefficients via gradient descent [S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_dynamic_gaussian(gaussian, time_t, freq_order_K, base_freq):
    """
    Computes the time-dependent position and rotation for a single Gaussian.
    gaussian: dict containing static attributes and dynamic coefficients
    time_t: float, current time
    freq_order_K: int, number of Fourier harmonics
    base_freq: float, omega
    """
    # 1. Compute Position via Fourier Series
    pos = [0.0, 0.0, 0.0]
    for dim in range(3):
        # Extract coefficients for this dimension (x=0, y=1, z=2)
        # Assume gaussian['pos_coeffs'] is a list of [a_0, a_1, b_1, a_2, b_2, ...]
        # Structure: [bias] + [cos1, sin1, cos2, sin2, ..., cosK, sinK]
        coeffs = gaussian['pos_coeffs'][dim]
        val = coeffs[0]  # a_0
        for k in range(1, freq_order_K + 1):
            a_k = coeffs[2 * k - 1]
            b_k = coeffs[2 * k]
            val += a_k * math.cos(k * base_freq * time_t) + b_k * math.sin(k * base_freq * time_t)
        pos[dim] = val

    # 2. Compute Rotation via Linear Function
    # gaussian['rot_coeffs'] contains [q0_x, q0_y, q0_z, q0_w, m_x, m_y, m_z, m_w]
    q0 = gaussian['rot_coeffs'][0:4]
    m = gaussian['rot_coeffs'][4:8]
    
    rot = [q0[i] + m[i] * time_t for i in range(4)]
    
    # Normalize quaternion [G]
    norm = math.sqrt(sum(r*r for r in rot))
    rot = [r / norm for r in rot]
    
    return pos, rot

def render_dynamic_scene(gaussians, cameras, time_t, freq_order_K, base_freq):
    """
    Renders a frame for a specific time t.
    """
    # Update Gaussian attributes for time t
    dynamic_gaussians = []
    for g in gaussians:
        pos, rot = compute_dynamic_gaussian(g, time_t, freq_order_K, base_freq)
        # Create a temporary Gaussian object with updated pos/rot but static scale/opacity/color
        temp_g = {
            'pos': pos,
            'rot': rot,
            'scale': g['scale'],
            'opacity': g['opacity'],
            'sh_coeffs': g['sh_coeffs']
        }
        dynamic_gaussians.append(temp_g)
    
    # Standard 3DGS rasterization
    image = rasterize(dynamic_gaussians, cameras)
    return image
```

**Edge Cases and Pitfalls:**
*   **Quaternion Normalization:** The linear interpolation of quaternions does not preserve unit length; normalization is required after each time step [G].
*   **Frequency Selection:** The choice of $K$ and $\omega$ affects the ability to model periodic vs. non-periodic motion. High $K$ may lead to overfitting [S2].
*   **Temporal Continuity:** The parametric nature ensures smooth motion, but discontinuities may arise if the Fourier series does not capture high-frequency motion components [S2].

## Subtopics

### Gaussian-Flow
Combines polynomial fitting (for smooth motion) and Fourier approximation (for violent motion) in a Dual-Domain Deformation Model (DDDM). It also models color changes over time and uses adaptive timestep scaling. [Gaussian-Flow](8797-gaussian-flow.md)

## Operating characteristics
*   **Time Complexity:** Rendering is $O(N)$ per frame, where $N$ is the number of Gaussians, similar to static 3DGS. The overhead of computing Fourier/linear functions is negligible.
*   **Memory Complexity:** Storage is $O(N \cdot K)$ for position coefficients and $O(N)$ for rotation coefficients, where $K$ is the Fourier order. This is significantly less than implicit deformation networks which require neural network weights [S2].
*   **Strengths:** Fast training and rendering; parameters are independent of time steps; robust for periodic motions (e.g., human animation) [S2].
*   **Limitations:** May struggle with highly complex, non-periodic deformations compared to neural deformation fields [S2].
*   **When to use:** Dynamic scenes with smooth or periodic motion; when computational efficiency is critical [S2].

## Background
Dynamic 3D Gaussian Splatting was proposed by Katsumata et al. as an explicit deformation method for 3DGS [S2]. It was inspired by the observation that human and articulated object motions are often periodic, making Fourier series a suitable representation for position changes [S2]. It contrasts with implicit methods like D-NeRF by avoiding large neural networks, thus reducing overfitting risks and training time [S2].

## Sources
[S1] A Survey on 3D Gaussian Splatting, https://doi.org/10.1145/3807511, fetched 2026-10-27
[S2] Recent advances in 3D Gaussian splatting, https://doi.org/10.1007/s41095-024-0436-y, fetched 2026-10-26
