# Maximum Intensity Difference Accumulation
## Overview
Maximum Intensity Difference Accumulation (MIDA) is a compositing technique that combines Direct Volume Rendering (DVR) and Maximum Intensity Projection (MIP) by modulating accumulated color and transparency based on the difference between the current maximum scalar intensity and the incoming sample's scalar intensity [S2]. It sets itself apart from standard [Back-to-Front Compositing](2078-back-to-front-compositing.md) by dynamically reducing the contribution of previously accumulated pixels when a higher-intensity feature is encountered, ensuring that high-intensity structures are not occluded [S2].
## How it works
**Inputs and Outputs:**
*   **Input:** A ray traversing a volume. For each sample $i$ along the ray: scalar intensity $s_i$, color $c_i$ (RGB), and opacity $\alpha_i$.
*   **Output:** Final accumulated color $C_{out}$ and opacity $\alpha_{out}$.
*   **State:** Running maximum scalar intensity $s_{acc}$ (initialized to the minimum possible scalar value in the dataset).

**Data Structures:**
*   `s_acc`: Floating-point scalar, stores the maximum scalar intensity encountered so far along the ray.
*   `C_dst`: Floating-point vector (RGB), stores the accumulated color.
*   `alpha_dst`: Floating-point scalar, stores the accumulated opacity.

**Steps:**

1.  **Initialization:**
    *   Set $s_{acc} \leftarrow s_{min}$ (the minimum scalar value in the dataset) [G].
    *   Set $C_{dst} \leftarrow (0, 0, 0)$.
    *   Set $\alpha_{dst} \leftarrow 0$.

2.  **Ray Marching Loop:**
    For each sample $i$ along the ray:
    
    a.  **Modulation Factor Calculation:**
        Calculate the modulation factor $\beta$ based on the difference between the incoming scalar intensity $s_i$ and the current maximum $s_{acc}$:
        $$ \beta = \begin{cases} 1 - (s_i - s_{acc}) & \text{if } s_i > s_{acc} \\ 1 & \text{otherwise} \end{cases} $$
        [S3].
        
    b.  **Modulate Accumulated State:**
        Apply $\beta$ to the currently accumulated color and opacity:
        $$ C_{dst} \leftarrow \beta \cdot C_{dst} $$
        $$ \alpha_{dst} \leftarrow \beta \cdot \alpha_{dst} $$
        [S3].
        
    c.  **Standard Compositing:**
        Composite the current sample $(c_i, \alpha_i)$ onto the modulated accumulator $(C_{dst}, \alpha_{dst})$:
        $$ C_{dst} \leftarrow C_{dst} + (1 - \alpha_{dst}) \cdot c_i $$
        $$ \alpha_{dst} \leftarrow \alpha_{dst} + (1 - \alpha_{dst}) \cdot \alpha_i $$
        [S3].
        
    d.  **Update Maximum:**
        If $s_i > s_{acc}$, update $s_{acc} \leftarrow s_i$ [S2].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def mida_composite(samples, s_min):
    """
    samples: list of tuples (s_i, c_i, alpha_i)
    s_i: scalar intensity
    c_i: color (r, g, b)
    alpha_i: opacity
    s_min: minimum scalar value in the dataset
    """
    # Initialize state
    s_acc = s_min
    c_dst = (0.0, 0.0, 0.0)
    alpha_dst = 0.0
    
    for s_i, c_i, alpha_i in samples:
        # Step 1: Calculate modulation factor beta
        if s_i > s_acc:
            beta = 1.0 - (s_i - s_acc)
        else:
            beta = 1.0
            
        # Step 2: Modulate accumulated color and transparency
        c_dst = (beta * c_dst[0], beta * c_dst[1], beta * c_dst[2])
        alpha_dst = beta * alpha_dst
        
        # Step 3: Standard front-to-back compositing
        one_minus_alpha_dst = 1.0 - alpha_dst
        c_dst = (
            c_dst[0] + one_minus_alpha_dst * c_i[0],
            c_dst[1] + one_minus_alpha_dst * c_i[1],
            c_dst[2] + one_minus_alpha_dst * c_i[2]
        )
        alpha_dst = alpha_dst + one_minus_alpha_dst * alpha_i
        
        # Step 4: Update maximum scalar intensity
        if s_i > s_acc:
            s_acc = s_i
            
    return c_dst, alpha_dst
```

**Parameters:**
*   **Scalar Intensity Range:** A range of mapped data intensity values (defined by minimum and maximum) can be specified to filter which volumetric features contribute to the final image [S2].
*   **Color Modulation:** The obtained gray-scale values can be modulated by a user-specified color value to result in more realistic visualizations [S2].

**Edge Cases and Pitfalls:**
*   **Negative Beta:** If $s_i - s_{acc} > 1$, $\beta$ becomes negative, which can cause color inversion or artifacts [G]. Implementers should clamp $\beta$ to $[0, 1]$ if necessary.
*   **Definition of "Current Maximum":** $s_{acc}$ tracks the maximum of *previous* samples along the ray. It is updated after $\beta$ is calculated to ensure the modulation factor compares against the true prior maximum [S2].
*   **Step Size Correction:** One source suggests correcting input alpha for varying step sizes using `alpha_src = 1.0 - (1.0 - alpha_i) ** (step_size * 200.0)` [S1, unofficial]. This factor is implementation-specific and may require adjustment for different resolutions or transfer functions.

## Operating characteristics
*   **Time Complexity:** $O(N)$ per ray, where $N$ is the number of samples, matching standard compositing [G].
*   **Memory Complexity:** $O(1)$ per ray, storing only $s_{acc}$, $C_{dst}$, and $\alpha_{dst}$ [G].
*   **Strengths:** Enhances visibility of high-intensity features by reducing occlusion from lower-intensity features behind them, combining the benefits of DVR and MIP [S2]. It allows users to control visualization with a single set of GUI slider controls defining an intensity range, eliminating the need for complex transfer function specification [S2].
*   **Limitations:** Sensitive to the scalar intensity range and normalization. Can produce artifacts if the difference $s_i - s_{acc}$ is large or unclamped. The step size correction factor is arbitrary and implementation-dependent [S1, unofficial].
*   **When to Use:** When high-intensity features (e.g., bones, contrast agents) are of primary interest and should not be occluded by lower-intensity structures [S2].
*   **When Not to Use:** When low-intensity features are important, or when the scalar values do not correlate with feature importance [S2].

## Background
MIDA was introduced by Bruckner et al. in 2009 [S3]. It was extended for rendering multichannel data and applied to visualization in neurobiology research [S3]. It is part of the family of importance-driven volume rendering techniques [S3].
## Sources
[S1] fargiolas/qvrc, https://github.com/fargiolas/qvrc, fetched 2026-10-26 (unofficial)
[S2] Context-Preserving Volumetric Data Set Exploration Using a 3D Painting Metaphor, https://doi.org/10.1007/978-3-642-33179-4_33, fetched 2026-10-27
[S3] Fast Volumetric Data Exploration with Importance-Based Accumulated Transparency Modulation, https://doi.org/10.2312/vg/vg10/061-068, fetched 2026-10-26
