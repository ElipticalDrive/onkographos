# RehearsalNeRF

Also called: Rehearsal prior-based Neural Radiance Fields

## Overview
RehearsalNeRF is a method for learning disentangled neural fields in scenes with severe, dynamic illumination changes by leveraging a "rehearsal stage" (captured under stable lighting) to enforce geometric consistency [S1]. It solves the "illumination-radiance ambiguity" where dynamic lighting effects are entangled with object radiance, a problem not addressed by methods assuming stable light sources [S1]. Unlike [D2NeRF](11973-urban-radiance-fields.md) which handles dynamic objects but assumes stable lighting, RehearsalNeRF explicitly decouples dynamic illuminations, static objects, and dynamic objects into five separate neural fields, enabling applications like controlling lighting while stopping motion [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:**
    *   Rehearsal stage video $\hat{C}_{reh}$ (stable lighting, no dynamic objects).
    *   Main stage video $\hat{C}_{main}$ (dynamic lighting, dynamic objects present).
    *   Camera parameters (from COLMAP) [S1].
    *   Binary masks $M_{reh}$ and $M_{main}$ for dynamic objects, obtained via off-the-shelf interactive segmentation [S1].
    *   Optical flow $G_{reh \to main}$ from rehearsal to main stage [S1].
*   **Outputs:**
    *   Rendered main stage color $\hat{C}_{main}(r, \tau)$.
    *   Disentangled components: Static object radiance/density, Dynamic object radiance/density, Dynamic illumination radiance.

**Data Structures and Neural Fields:**
The architecture consists of five neural fields (MLPs), each mapping inputs to specific properties [S1]:
1.  **Static Objects ($\psi_S$):**
    *   Input: Location $x \in \mathbb{R}^3$, Viewing direction $d$.
    *   Output: Density $\sigma_S$, Radiance $c_S$.
    *   Note: Does not take time $\tau$ as input [S1].
2.  **Dynamic Objects ($\psi_D$):**
    *   Input: Location $x$, Viewing direction $d$, Time-stamp $\tau \in \mathbb{R}$.
    *   Output: Density $\sigma_D$, Radiance $c_D$.
3.  **Lighting Effects ($\psi_L$):**
    *   Input: Location $x$, Viewing direction $d$, Time-stamp $\tau$.
    *   Output: Saturation $S$, Value $V$, Probability vector $P_H \in \mathbb{R}^k$, Density $\sigma_L$.
    *   $k$ is the number of hue candidates (typically 5) [S1].
4.  **Intensity ($\psi_I$):**
    *   Input: Location $x$, Viewing direction $d$, Time-stamp $\tau$.
    *   Output: Intensity $i \in \mathbb{R}$.
5.  **Appearance ($\psi_A$):**
    *   Input: Radiance components $c_x, c_L$.
    *   Output: Brightness/Alpha $\alpha_x$ (used to modulate illumination on objects) [S1].

**Learnable Illumination Vector:**
A learnable vector $v_h \in \mathbb{R}^{k \times 3}$ stores hue candidates for dynamic illuminations. It is initialized by taking the centroids of $k$-means clustering of hue values that differ between the rehearsal and main stage scenes [S1].

**Rendering Formulas:**
The dynamic illumination color $c_L$ is computed from HSV components and intensity [S1]:
$$ H = \sum_{j=1}^{k} P_{H,j} v_{h,j}(\tau) $$
$$ c_L = f_{rgb}(H, S, V) \times i $$
where $f_{rgb}$ converts HSV to RGB, $P_{H,j}$ is the probability of the $j$-th hue candidate, and $v_{h,j}(\tau)$ is the hue candidate at time $\tau$ [S1].

The total main stage color $C_{main}(r, \tau)$ is the sum of static and dynamic components [S1]:
$$ C_{main}(r, \tau) = C_S^{main}(r, \tau) + C_D^{main}(r, \tau) $$
where $C_S^{main}$ and $C_D^{main}$ are rendered using volume rendering with densities $\sigma_S, \sigma_D$ and modulated radiance. The illumination affects both static and dynamic objects via the appearance field $\psi_A$ [S1].

The rendered color of static objects without dynamic illumination is $\tilde{C}_S(r, \tau)$, and for dynamic objects is $\tilde{C}_D(r, \tau)$ [S1].

**Optimization Steps:**
1.  **Initialize:** Initialize $v_h$ using $k$-means centroids of hue differences between $\hat{C}_{reh}$ and $\hat{C}_{main}$ [S1].
2.  **Compute Optical Flow:** Estimate $G_{reh \to main}$ between rehearsal and main stage videos [S1].
3.  **Forward Pass:** For each ray $r$ and time $\tau$:
    *   Query $\psi_S(x, d) \to (\sigma_S, c_S)$.
    *   Query $\psi_D(x, d, \tau) \to (\sigma_D, c_D)$.
    *   Query $\psi_L(x, d, \tau) \to (S, V, P_H, \sigma_L)$.
    *   Query $\psi_I(x, d, \tau) \to i$.
    *   Compute $c_L$ using $H, S, V, i$ and $v_h$.
    *   Compute $\alpha$ using $\psi_A$ to modulate illumination [S1].
    *   Render $C_S^{main}, C_D^{main}$ via volume rendering [S1].
    *   Render $\tilde{C}_S, \tilde{C}_D$ (without dynamic illumination) [S1].
4.  **Compute Losses:**
    *   **Reconstruction Loss ($L_{recon}$):**
        $$ L_{recon}(r, \tau) = \| \hat{C}_{main}(r, \tau) - C_{main}(r, \tau) \|_2^2 $$
    *   **Dynamic Object Regularization ($L_{dyn}$):** Uses optical flow to link rehearsal prior to dynamic objects [S1]:
        $$ L_{dyn}(r, \tau) = M_{main} \| E_{reh \to main} \{ G_{reh \to main} (M_{reh} \odot \hat{C}_{reh}) - \tilde{C}_D \} \|_2^2 + \sum_{i=-2}^{2} \| E_{\tau+i \to \tau}^{dynamic} \{ G_{\tau+i \to \tau} (\tilde{C}_D(r, \tau+i)) - \tilde{C}_D(r, \tau) \} \|_2^2 $$
        where $E$ is the optical flow confidence [S1].
    *   **Rehearsal Prior Loss ($L_{reh}$):** Decouples static objects from illumination [S1]:
        $$ L_{reh}(r, \tau) = (1 - M_{reh})\|\hat{C}_{reh}(r, \tau) - \tilde{C}_S(r, \tau)\|_2^2 + (1 - M_{main})\|\hat{C}_{main}(r, \tau) - C_S^{main}(r, \tau)\|_2^2 + (1 - M_{main})\|\tilde{C}_S(r, \tau) - (\tilde{C}_S(r, \tau) + \tilde{C}_D(r, \tau))\|_2^2 $$
        The third term provides self-supervision for regions where masks do not overlap [S1].
    *   **Static/Dynamic Separation Loss ($L_p$):**
        $$ L_p(r, \tau) = M_{main} \frac{\sigma_S(r, \tau)}{\sigma_D(r, \tau)} + (1 - M_{main}) \frac{\sigma_D(r, \tau)}{\sigma_S(r, \tau)} $$
        This penalizes regions where both static and dynamic densities are high [S1].
5.  **Total Loss:**
    $$ L(r, \tau) = L_{recon}(r, \tau) + \lambda_{dyn} L_{dyn}(r, \tau) + \lambda_{reh} L_{reh}(r, \tau) + \lambda_p L_p(r, \tau) $$
6.  **Update:** Backpropagate to update all MLPs and $v_h$ [S1].

**Parameters:**
*   $k$: Number of hue candidates (k-means centroids). Set to 5 based on the number of light sources [S1].
*   $\lambda_{dyn}$: 0.01 [S1].
*   $\lambda_{reh}$: 0.5 [S1].
*   $\lambda_p$: 0.01 [S1].

**Pitfalls:**
*   Neural fields may fail to learn in regions where rehearsal and main stage masks do not overlap. This is mitigated by the third term in $L_{reh}$ which provides self-supervision [S1].
*   Optical flow estimation can be erroneous due to illumination differences between rehearsal and main stages. This is addressed by the temporal consistency term in $L_{dyn}$ [S1].

## Operating characteristics
*   **Time Complexity:** Training requires approximately 120K iterations. With a batch size of 3,072, it takes about one day on a single RTX 3090 GPU [S1].
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Explicitly handles varying illuminations and the illumination-radiance ambiguity. Decouples dynamic lighting from static/dynamic objects, enabling video editing applications like controlling lighting while stopping motion [S1].
*   **Limitations:** Requires a rehearsal stage video (stable lighting) captured before the dynamic illumination occurs. Relies on off-the-shelf segmentation and optical flow, which may introduce errors [S1].
*   **When to use:** Scenes with severe, dynamic illumination changes where disentangling lighting from object appearance is required.
*   **When not to use:** Scenes with stable lighting (use standard NeRF or [D2NeRF](11973-urban-radiance-fields.md)) or where rehearsal data is unavailable [S1].

## Background
RehestralNeRF was introduced to address the lack of methods explicitly handling varying illuminations in Neural Radiance Fields (NeRFs) [S1]. Previous works assumed stable light sources or only handled subject motion, failing to resolve the "illumination-radiance ambiguity" where dynamic lighting is entangled with object radiance [S1]. The method builds on [D2NeRF](11973-urban-radiance-fields.md) as a baseline for dynamic objects but extends it with a learnable illumination vector and rehearsal priors [S1]. It was developed to enable video editing tasks such as freezing motions under varying lights and vice versa [S1].

## Sources
[S1] RehearsalNeRF: Decoupling Intrinsic Neural Fields of Dynamic Illuminations for Scene Editing, https://link.springer.com/article/10.1007/s11263-026-02816-0, fetched 2026-10-27
