# Accumulation-level intermixing

Also called: accumulation-level blending, step-wise intermixing

## Overview
Accumulation-level intermixing is a multi-modal volume rendering technique that resolves the depth-ordering artifacts inherent in image-level intermixing by combining ray samples from multiple co-registered volumes at every integration step along the viewing ray [S1, unofficial]. Unlike image-level methods which composite final images and lose spatial precedence, this method preserves the correct occlusion relationships between different data modalities (e.g., CT and MRI) by mixing their optical properties during the ray-marching accumulation process [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** Two or more co-registered volume datasets (e.g., $V_1, V_2$), corresponding transfer functions mapping density to color ($C$) and opacity ($\alpha$), and a viewing ray defined by origin $\mathbf{o}$ and direction $\mathbf{d}$.
*   **Outputs:** A single pixel color $C_{out}$ and opacity $\alpha_{out}$ representing the composite view.

**Data Structures:**
*   **Ray Sample:** At each step $i$ along the ray, the renderer retrieves density values $d_1, d_2$ from the respective volumes.
*   **Intermediate State:** The ray accumulator maintains the current accumulated color $C_{acc}$ and accumulated opacity $\alpha_{acc}$.

**Steps:**
1.  **Sample:** For each step along the ray, sample the density values $d_1$ and $d_2$ from the respective volumes at the current position.
2.  **Transfer:** Map densities to intermediate color and opacity values using the respective transfer functions:
    $$ c_1 = T_1(d_1), \quad \alpha_1 = T_{\alpha,1}(d_1) $$
    $$ c_2 = T_2(d_2), \quad \alpha_2 = T_{\alpha,2}(d_2) $$
    [G] If a volume is empty or density is below threshold, set $\alpha=0$ and $c=0$.
3.  **Mix:** Combine the intermediate color and opacity values $(c_1, \alpha_1)$ and $(c_2, \alpha_2)$ using a mixing function $M$. The specific mixing function depends on the desired visual effect (e.g., inclusive opacity).
4.  **Accumulate:** Update the ray accumulator using the standard front-to-back compositing equation with the mixed values:
    $$ \alpha_{acc}^{new} = \alpha_{acc}^{old} + (1 - \alpha_{acc}^{old}) \cdot \alpha_{mixed} $$
    $$ C_{acc}^{new} = C_{acc}^{old} + (1 - \alpha_{acc}^{old}) \cdot \alpha_{mixed} \cdot c_{mixed} $$
5.  **Terminate:** If $\alpha_{acc} \geq 1$ (or a threshold close to 1), terminate the ray. Otherwise, advance to the next step.

**Formulas:**
The most common mixing function cited is the **inclusive opacity mode**, which assumes the volumes are transparent overlays where the combined opacity is the probability that at least one volume is opaque at that point:
$$ \alpha_{mixed} = 1 - (1 - \alpha_1)(1 - \alpha_2) $$
$$ c_{mixed} = \frac{\alpha_1 c_1 + \alpha_2 c_2 (1 - \alpha_1)}{\alpha_{mixed}} \quad \text{(if } \alpha_{mixed} > 0 \text{, else } 0) $$
[Note: The source explicitly provides the opacity formula $\alpha = 1 - (1 - \alpha_{CT})(1 - \alpha_{MRI})$ [S1, unofficial]. The color combination formula is derived from standard alpha compositing logic for mixed channels, though the source text primarily highlights the opacity calculation. If the source implies simple weighted addition for color, it is not explicitly defined in the provided text.]

**Parameters:**
*   **Mixing Function:** The choice of mixing function (e.g., inclusive, exclusive, weighted sum) is the primary parameter. The inclusive mode is standard for preserving depth order of semi-transparent structures [S1, unofficial].
*   **Step Size:** Determines the trade-off between performance and aliasing.

**Edge Cases and Pitfalls:**
*   **Zero Denominator:** If $\alpha_{mixed} = 0$, the color contribution is undefined; handle by setting $c_{mixed} = 0$.
*   **Precision:** Accumulating many small opacity steps can lead to floating-point precision errors in $\alpha_{acc}$. Use double precision for accumulation if necessary.
*   **Degenerate Input:** If one volume has zero opacity everywhere, the method degrades to single-volume rendering.

## Subtopics

### Illumination model level intermixing
A variant where ray samples from different volumes are combined *before* colors and opacities are computed, typically by summing densities [S1, unofficial]. This allows one volume to act as an emission source (e.g., SPECT metabolic activity) and another as an attenuation medium (e.g., CT anatomy) [S1, unofficial]. It is chosen when the physical interaction between modalities is additive (density-based) rather than optical (opacity-based). Unlike accumulation-level, it does not preserve depth ordering of the final rendered surfaces as naturally, as the transfer functions are applied to the combined density [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$, where $N$ is the number of steps per ray and $M$ is the number of volumes. Each step requires sampling from all $M$ volumes and performing the mixing operation.
*   **Memory Complexity:** $O(V)$, where $V$ is the total size of all volume datasets. No additional large buffers are required compared to single-volume rendering.
*   **Performance:** Slower than image-level intermixing because the compositing loop runs for every pixel with full mixing logic, but faster than re-rendering separate images and compositing them in post if the number of volumes is large.
*   **Strengths:** Preserves correct depth ordering between modalities; allows for physically plausible blending of different data types (e.g., emissive + attenuating) [S1, unofficial].
*   **Limitations:** Requires co-registered volumes; mixing function choice is non-trivial and affects visual interpretation; computationally more intensive per pixel than simple alpha blending.
*   **When to Use:** When visualizing multiple modalities (e.g., CT/MRI, SPECT/CT) where the spatial relationship between structures from different sources is critical and must be preserved [S1, unofficial].
*   **When Not to Use:** When volumes are not co-registered, or when performance is the primary constraint and depth ordering between modalities is not important.

## Background
Accumulation-level intermixing was introduced as a solution to the limitations of image-level intermixing in multi-modal volume rendering [S1, unofficial]. It is discussed in the context of rendering multi-channel data, such as the Visible Man’s CT and MRI datasets, where preserving the depth relationship between anatomical structures from different imaging modalities is essential [S1, unofficial]. The technique is particularly relevant in medical visualization, such as radiation therapy dose planning, where visualizing an emissive radiation beam within an attenuating CT volume is required [S1, unofficial].

## Sources
[S1] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
