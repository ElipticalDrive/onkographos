# Incremental Blending for In-scattering

Also called: Incremental In-scattering Blending

## Overview
Incremental Blending for In-scattering is a method for approximating volumetric light scattering (in-scattering) in direct volume rendering by accumulating scattered light contributions from multiple discrete directions [S1]. It solves the problem of computing global illumination effects in participating media without the prohibitive cost of full path tracing or expensive pre-computed light fields [S1]. It sets itself apart from standard compositing by treating the scattering integral as a sum of directional components that are blended incrementally, allowing for interactive updates as the number of scattering directions increases [S1]. Unlike [Intermixing](5139-intermixing.md), which focuses on spatial mixing of properties, this method focuses on the temporal accumulation of lighting estimates across rendering passes or directions [S1].

## How it works
The method discretizes the scattering sphere into $J$ directions. For each direction $j$ (from 1 to $J$), it computes the scattered intensity contribution and blends it with the previously accumulated intensity [S1].

**Inputs and Outputs:**
*   **Input:** Volume data (opacity $\alpha$, color $c$), transfer function parameters, light source direction, scattering sphere discretization (number of rays $J$), bias parameter $I_{bias}$.
*   **Output:** Final in-scattered intensity image $I_{eye}$ or accumulated intensity buffer $I_{stored}$.

**Data Structures:**
*   $I_{stored}$: A buffer (volume or image) storing the accumulated in-scattered intensity. Initialized to zero.
*   $I_{final}$: A temporary buffer or variable storing the result of the current directional computation.
*   $j$: Integer counter for the current scattering direction (1 to $J$).

**Steps:**

1.  **Initialization:** Initialize $I_{stored}$ to zero for all pixels/voxels. Set $j = 1$.
2.  **Directional Computation:** For the current direction $j$:
    a. Compute the scattered intensity $I_s(s_k)$ for each sample point $s_k$ along the ray. This typically involves evaluating the phase function and light transport from the source to the sample [S1].
    b. Compute the emitted color contribution $g_k$ using the bias parameter:
       $$g_k = (I_{bias} + (1 - I_{bias}) \cdot I_s(s_k)) \cdot c(s_k) \cdot \alpha(s_k)$$
       where $I_{bias} \in [0, 1]$ adjusts the minimum light level (default white light assumption) [S1].
    c. Compute the total in-scattered intensity $I_{final}$ for this direction by integrating along the ray using the standard volume rendering equation:
       $$I_{final} = \sum_{k=0}^n g_k \prod_{p=0}^{k-1} (1 - \alpha(s_p))$$
       where $n$ is the number of samples, $c(s_k)$ is the color from the transfer function, and $\alpha(s_p)$ is the opacity at sample $p$ [S1].
3.  **Incremental Blending:** Blend the new result $I_{final}$ with the stored value $I_{stored}$ using the incremental blending formula:
    $$I_{blended} = \frac{1}{j} \cdot I_{final} + \left(1 - \frac{1}{j}\right) \cdot I_{stored}$$
    where $I_{blended}$ is the new value to store [S1].
4.  **Update:** Store $I_{blended}$ as the new $I_{stored}$.
5.  **Iteration:** Increment $j$. If $j \le J$, return to Step 2. Otherwise, the final in-scattered intensity is $I_{stored}$.

**Parameters:**
*   $J$: Number of scattering directions. Higher values yield better accuracy but increase computation time [S1].
*   $I_{bias}$: Minimum light level bias, range $[0, 1]$. Default is often 0 or 1 depending on implementation, but the formula assumes a white light source if not extended [S1].

**Edge Cases and Pitfalls:**
*   **Precision:** Accumulating many small values can lead to precision loss. Using floating-point buffers for $I_{stored}$ is recommended [S1].
*   **Memory:** Storing intermediate results for each voxel for multiple passes can be memory-intensive. The source notes that for a $512^3$ volume, three additional volumes are needed for intermediate computations ($\alpha l_p$, $\alpha g$, $I_{final}$) [S1].
*   **Light Color:** The base formula assumes a white light source. Extending to colored light requires per-channel computation or vector operations [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def incremental_blending_in_scattering(volume, transfer_function, light_direction, J, I_bias):
    """
    Computes in-scattered light using incremental blending.
    
    Args:
        volume: 3D array of opacity values alpha.
        transfer_function: Function to map density to color c.
        light_direction: Direction vector for light source.
        J: Number of scattering directions.
        I_bias: Bias parameter for minimum light level [0, 1].
        
    Returns:
        I_stored: Final accumulated in-scattered intensity.
    """
    # Initialize stored intensity buffer
    I_stored = zeros_like(volume)
    
    # Discretize scattering sphere into J directions
    # For simplicity, assume directions are pre-computed or sampled uniformly
    scattering_directions = sample_scattering_sphere(J)
    
    for j_idx, direction in enumerate(scattering_directions):
        j = j_idx + 1  # Current direction index (1-based)
        
        # Compute I_final for the current direction
        I_final = compute_directional_in_scattering(
            volume, transfer_function, direction, I_bias
        )
        
        # Incremental blending
        # I_blended = (1/j) * I_final + (1 - 1/j) * I_stored
        weight_new = 1.0 / j
        weight_old = 1.0 - weight_new
        
        I_stored = weight_new * I_final + weight_old * I_stored
        
    return I_stored

def compute_directional_in_scattering(volume, transfer_function, direction, I_bias):
    """
    Computes the in-scattered intensity for a single direction.
    """
    # Ray march along the direction
    # This is a simplified representation; actual implementation requires
    # ray casting through the volume
    I_final = 0.0
    transmittance = 1.0
    
    for sample in ray_cast(volume, direction):
        alpha = sample.opacity
        c = transfer_function(sample.density)
        
        # Compute scattered intensity I_s
        # This depends on the specific scattering model (e.g., Henyey-Greenstein)
        # and light transport from source to sample
        I_s = compute_scattered_intensity(sample.position, direction, volume)
        
        # Compute emitted color g_k
        g_k = (I_bias + (1.0 - I_bias) * I_s) * c * alpha
        
        # Accumulate
        I_final += g_k * transmittance
        transmittance *= (1.0 - alpha)
        
    return I_final
```

## Subtopics

### [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md)
Incremental Lighting for In-Scattering is a screen-space technique that approximates volumetric light scattering by propagating and blurring indirect lighting components across volume slices. It extends the base incremental blending method by introducing per-channel indirect attenuation parameters and a blur operation, allowing light to change color as it penetrates deeper into the volume. It is chosen when screen-space efficiency is prioritized over geometric accuracy. Link: [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md).

### [Hardware Blending Recurrence Evaluation](5317-hardware-blending-recurrence-evaluation.md)
Hardware Blending Recurrence Evaluation is a technique for evaluating the multiple forward scattering recurrence $I_k = g_{k-1} + T_{k-1} I_{k-1}$ by mapping the mathematical operation to GPU hardware blending and pixel read-back operations. It computes the incident light on each particle in a participating medium by using the frame buffer as an accumulator. It differs from [Incremental Lighting for In-Scattering](3401-incremental-lighting-for-in-scattering.md) by explicitly reading back pixel values to resolve the recurrence dependency. Link: [Hardware Blending Recurrence Evaluation](5317-hardware-blending-recurrence-evaluation.md).

### Volume Ray Casting with In-scattering
This variant uses the standard Riemann sum discretization for the eye ray integral, where the emitted color $g_k$ includes the in-scattering term $I_s(s_k)$ modulated by a bias $I_{bias}$ [S1]. The formula is $I_{eye} = \sum_{k=0}^n g_k \prod_{p=0}^{k-1} (1 - \alpha(s_p))$, with $g_k = (I_{bias} + (1 - I_{bias}) \cdot I_s(s_k)) \cdot c(s_k) \cdot \alpha(s_k)$ [S1]. It is used for single-pass volume rendering with approximate in-scattering. The bias parameter $I_{bias} \in [0, 1]$ adjusts the minimum light level, assuming a white light source by default [S1].

### Incremental Integration for Epipolar Rendering
This variant applies incremental integration to epipolar plane images (EPIs) for volumetric shadowing [S2]. It sweeps over view rays $\beta$ from the light away, maintaining an incremental function $b[\gamma]$ storing $\Gamma(\gamma)V(\gamma, \beta)$ [S2]. Initially $\beta=0$ and $b[\gamma]=\Gamma(\gamma)$. When stepping to a new $\beta$, $b[\gamma]$ is set to 0 for all light rays $\gamma$ where $S[\gamma]=\beta$ (light rays terminating at the current view ray) [S2]. The integral is $B(\beta) \sum_{\gamma<D[\beta]} b[\gamma]$ [S2]. The formula for intensity is $I[\beta] = \sum_{S[\gamma] > \beta} I(\gamma, \beta)$, where $D[\beta]$ is the view ray length, $S[\gamma]$ is the blocker distance, and $V(\gamma, \beta)$ is visibility [S2]. A pitfall is that naive incremental summation still requires iterating over all $\gamma$ for each $\beta$, resulting in $O(srd)$ complexity, same as brute force ray marching [S2].

## Operating characteristics
*   **Time Complexity:** The base method scales linearly with the number of scattering directions $J$ and the number of samples per ray. For $J$ directions, the complexity is $O(J \cdot N)$, where $N$ is the number of samples [S1]. The epipolar variant has $O(srd)$ complexity without hierarchical acceleration [S2].
*   **Memory Complexity:** Requires storage for $I_{stored}$ and intermediate buffers. For a volume of size $V$, it requires $O(V)$ memory for the accumulated intensity and potentially additional volumes for intermediate computations ($\alpha l_p$, $\alpha g$, $I_{final}$) [S1].
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Performance degrades linearly with $J$. Memory usage is constant with respect to $J$ if buffers are reused, but increases with volume resolution [S1].
*   **Strengths:** Allows for interactive updates as more scattering directions are added. Simple to implement.
*   **Limitations:** Approximate. Requires multiple passes or directions for accuracy. Memory intensive for large volumes [S1].
*   **When to use:** When approximate in-scattering is needed for interactive volume rendering.
*   **When not to use:** When high-fidelity global illumination is required, or when memory is severely constrained.

## Background
The method originates from the work on Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration [S1]. It was developed to address the computational cost of computing in-scattering in direct volume rendering. The approach uses a Riemann sum to discretize the ray casting integral and blends results from multiple scattering directions incrementally [S1]. The epipolar variant was developed for hierarchical volumetric shadow algorithms [S2].

## Sources
[S1] Interactive Global Light Propagation in Direct Volume Rendering using Local Piecewise Integration, https://doi.org/10.2312/vg/vg-pbg08/105-112, fetched 2026-10-26
[S2] A hierarchical volumetric shadow algorithm for single scattering, https://doi.org/10.1145/1866158.1866200, fetched 2026-11-03
