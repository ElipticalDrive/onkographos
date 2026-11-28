# Local Cloud Flow with FlowMap

Also called: FlowMap Cloud Movement, Noise Texture Flow Offset

## Overview
Local Cloud Flow with FlowMap simulates localized, non-uniform cloud movement (such as swirls or eddies) by perturbing noise texture sampling coordinates using a pre-computed FlowMap texture [S1, unofficial]. Unlike Global Cloud Flow, which applies uniform vector offsets to simulate large-scale wind, this method allows for complex, spatially varying flow directions and speeds defined per-pixel in the texture [S1, unofficial]. It addresses the visual artifact of noise texture stretching caused by divergent flow fields by employing a temporal interpolation technique between two phase-shifted noise samples [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** World space position $\mathbf{P}_{world}$, time $t$, Shape Noise texture $N_{shape}$, optional Detail Noise texture $N_{detail}$, and a FlowMap texture $F$.
*   **Output:** Perturbed noise values for volume density calculation.

**Data Structures:**
*   **FlowMap ($F$):** A 2D texture storing 3D flow direction vectors. Typically stored in RGB channels, where each component represents the X, Y, and Z components of the flow vector at that UV coordinate. The vector should be normalized or treated as a directional offset.

**Steps:**
1.  **UV Coordinate Calculation:** Compute 2D UV coordinates from the world space horizontal components of the query point.
    $$ u = P_{world}.x $$
    $$ v = P_{world}.z $$
    [G] In practice, these coordinates may be scaled or offset to match the FlowMap's tiling, but the source specifies direct XZ mapping [S1, unofficial].

2.  **Flow Direction Sampling:** Sample the FlowMap at $(u, v)$ to retrieve the local flow direction vector $\mathbf{d}$.
    $$ \mathbf{d} = F(u, v) $$
    [G] $\mathbf{d}$ is a 3D vector $(d_x, d_y, d_z)$.

3.  **Temporal Phase Calculation:** To prevent severe stretching of noise features over time due to varying flow speeds, calculate two time phases using the fractional part of time.
    $$ t_1 = \text{fract}(t) $$
    $$ t_2 = \text{fract}(t + 0.5) $$
    [G] $\text{fract}(x) = x - \lfloor x \rfloor$. The offset of 0.5 represents a "half-beat" phase shift [S1, unofficial].

4.  **Noise Coordinate Perturbation:** Calculate two perturbed sampling coordinates for the noise texture. The flow vector $\mathbf{d}$ is scaled by the time phases to create the offset.
    $$ \mathbf{P}_1 = \mathbf{P}_{world} + t_1 \cdot \mathbf{d} $$
    $$ \mathbf{P}_2 = \mathbf{P}_{world} + t_2 \cdot \mathbf{d} $$
    [G] The source implies the flow vector acts as a velocity or displacement rate. The Y component of $\mathbf{d}$ controls vertical rolling, affecting the overall distribution of clouds over time [S1, unofficial].

5.  **Noise Sampling:** Sample the Shape Noise texture at both perturbed coordinates.
    $$ n_1 = N_{shape}(\mathbf{P}_1) $$
    $$ n_2 = N_{shape}(\mathbf{P}_2) $$
    [G] Only the Shape Noise texture is typically perturbed this way to save performance; other noise textures (e.g., detail noise) remain unperturbed [S1, unofficial].

6.  **Interpolation:** Interpolate between the two noise samples to produce the final flowed noise value.
    $$ n_{final} = \text{lerp}(n_1, n_2, 0.5) $$
    [G] The source specifies interpolating between the two samples. A standard linear interpolation with a weight of 0.5 is implied by the "smooth flow effect" description, though the exact blend factor is not explicitly defined beyond the half-beat offset [S1, unofficial].

**Pitfalls:**
*   **Stretching:** Without the `fract(time)` technique, noise textures will stretch severely over time because flow directions and speeds vary across the texture [S1, unofficial].
*   **Performance:** Applying FlowMap sampling and double noise evaluation to all noise textures is expensive. It is recommended to apply this only to the Shape Noise texture [S1, unofficial].

## Subtopics

### [Global Cloud Flow](2270-global-cloud-flow.md)
Simulates global movement using uniform wind vectors. Chose when large-scale, coherent wind patterns are needed without local turbulence.

### [Noise Texture Advection](10551-noise-texture-advection.md)
GPU-based advection of tensor fields. Chose when visualizing complex vector/tensor fields with edge detection, rather than simple procedural cloud flow.

## Operating characteristics
*   **Time Complexity:** $O(1)$ per sample, but effectively doubles the cost of Shape Noise evaluation due to the two-phase sampling and interpolation [S1, unofficial].
*   **Memory Complexity:** Requires storage for the FlowMap texture.
*   **Strengths:** Allows for complex, localized flow patterns (swirls) that global vectors cannot achieve. Simple to implement.
*   **Limitations:** Computationally expensive if applied to all noise layers. Requires careful tuning of the time fractionalization to avoid visual popping or excessive stretching.
*   **When to use:** When local turbulence or swirling cloud structures are desired.
*   **When not to use:** When performance is critical and global wind is sufficient, or when detail noise also requires flow (which is usually skipped for performance).

## Background
This method is described in the context of volumetric cloud rendering in games, specifically referencing implementations seen in titles like *Red Dead Redemption 2* (though RDR2 is cited for a different technique regarding cloud distribution, the FlowMap concept is discussed in the same article) and general industry practices for volume clouds [S1, unofficial]. It is a technique for enhancing the visual fidelity of procedural noise by introducing controlled, local motion.

## Sources
[S1] 体积云效果的实现，游戏世界的云合雾集, https://leihuo.163.com/institution/news/20230719/38423_1099592.html, fetched 2026-10-30 (unofficial)
