# See-NeRF

Also called: See-NeRF framework, Sensor-Physics Grounded Deblurring HDR NeRF

## Overview
See-NeRF is a unified sensor-physics grounded NeRF framework for sharp High Dynamic Range (HDR) novel view synthesis from single-exposure blurry Low Dynamic Range (LDR) images and corresponding event data [S1, unofficial]. It solves the problem of suboptimal HDR and deblurring results caused by sensor-physics mismatches, where traditional methods ignore the specific imaging processes of RGB and event sensors [S1, unofficial]. Unlike sibling methods like EvHDR-NeRF that apply tone mapping directly to 3D points, See-NeRF explicitly models the radiance-to-measurement process for both RGB and event sensors, aligning rendered outputs with physical sensor recordings [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:** Single-exposure blurry LDR images $\mathcal{I}_{LDR}$, corresponding event data $B$, camera poses $p(t)$, and intrinsics.
*   **Outputs:** Sharp HDR novel view images $\hat{\mathcal{I}}_{HDR}$ and LDR images $\hat{\mathcal{I}}_{LDR}$ for arbitrary camera poses.

**Data Structures**
*   **NeRF Network ($F_\theta$):** A Multi-Layer Perceptron (MLP) with parameters $\theta$. It takes positional encoding of 3D coordinates $\mathbf{o}$ and view direction $\mathbf{d}$ as input.
    *   Input: $\gamma_o(\mathbf{o}), \gamma_d(\mathbf{d})$.
    *   Output: Raw HDR radiance $\mathbf{e} \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$.
*   **RGB CRF Model:** Three separate MLPs ($f_{crf}$), one for each color channel (R, G, B).
    *   Input: Log-domain raw HDR pixel value.
    *   Output: Estimated LDR pixel value.
*   **Event CRF Model:**
    *   **Delay Coefficient MLP ($f_{ev}$):** Predicts delay coefficient $\epsilon_i$.
        *   Input: Event sensor pixel radiance $E_{ev}$.
        *   Output: Scalar $\epsilon_i$.
    *   **Delay Offset Function ($h$):** Estimates temporal delay offset from input events. (Implementation details omitted in source; treated as a black-box function $h(\cdot)$ [S1, unofficial]).

**Positional Encoding**
The input coordinates $\mathbf{x} \in \mathbb{R}^3$ (position) and $\mathbf{d} \in \mathbb{R}^3$ (view direction) are mapped to higher dimensions using hierarchical positional encoding $\gamma_M(\cdot)$ [S1, unofficial]:
$$ \gamma_M(x) = \{\sin(2^m\pi x), \cos(2^m\pi x)\}_{m=0}^{M} $$
This is applied to each component of $\mathbf{o}$ and $\mathbf{d}$ [S1, unofficial].

**Numbered Steps**

1.  **Scene Representation:** For a 3D point $\mathbf{o}$ and direction $\mathbf{d}$, compute raw HDR radiance $\mathbf{e}$ and density $\sigma$ using the NeRF network:
    $$ (\mathbf{e}, \sigma) = F_{\theta}(\gamma_o(\mathbf{o}), \gamma_d(\mathbf{d})) $$
    [S1, unofficial]

2.  **Ray Sampling and Volume Rendering:**
    *   Discretely sample $b+1$ time points $\{t_i\}_{i=0}^b$ within the exposure time $[t_{start}, t_{end}]$ [S1, unofficial].
    *   For each time point $t_i$, render the latent sharp HDR radiance $E_i(x,y,p(t_i))$ at pixel $(x,y)$ using volume rendering:
        $$ E(x,y,p(t)) = \sum_{i=1}^{N} T_i (1-\exp(-\sigma_i \delta_i)) \mathbf{e}_i $$
        where $T_i = \exp(-\sum_{j=1}^{i-1} \sigma_j \delta_j)$, $\delta_i$ is the distance between adjacent sampled 3D points, and $N$ is the number of samples along the ray [S1, unofficial].

3.  **Raw HDR Pixel Calculation:** Calculate the predicted raw HDR pixel value by summing weighted radiance values at sampled time points:
    $$ \hat{\mathcal{I}}_{HDR}(x,y) = \sum_{i=0}^{b} w_i E_i(x,y,p(t_i)) $$
    where $w_i$ are weights calculated from time points $\{t_i\}_{i=0}^b$ [S1, unofficial].

4.  **Physically Grounded RGB CRF Model:** Convert the raw HDR pixel value to LDR using the learned CRF MLPs in the log domain:
    $$ \hat{\mathcal{I}}_{LDR}(x,y) = f_{crf}(\ln(\hat{\mathcal{I}}_{HDR}(x,y) \Delta t_{exp})) $$
    where $\Delta t_{exp} = t_{end} - t_{start}$ is the exposure time [S1, unofficial].

5.  **Event CRF Model:**
    *   **Bayer Pattern Adaptation:** Input the $b+1$ scene radiance values $\{E(x,y,p(t_i))\}_{i=0}^b$ into an RGGB Bayer pattern to obtain event sensor pixel radiance values $E_{ev}(x,y,p(t_i))$ [S1, unofficial].
    *   **Delay Coefficient:** Calculate delay coefficient $\epsilon_i$ using the MLP $f_{ev}$:
        $$ \epsilon_i = f_{ev}(E_{ev}(x,y,p(t_i))) $$
        [S1, unofficial]
    *   **Latency Synthesis:** Synthesize event latency using a second-order low-pass filter:
        $$ L^{i}_{lp} = (1-\epsilon_i)L^{i-1}_{lp} + \epsilon_i E_{ev}(p(t_i)) $$
        with initialization $L^0_{lp} = E_{ev}(p(t_0))$ [S1, unofficial].
    *   **Ideal Event Count:** Calculate estimated event numbers $\hat{B}'(t_i, t_{i+1})$ using the ideal model on filtered radiance:
        $$ \hat{B}'(t_i, t_{i+1}) = \text{floor}\left(\frac{\ln(L^{i+1}_{lp}) - \ln(L^{i}_{lp})}{\Theta}\right) $$
        where $\Theta$ is the contrast threshold [S1, unofficial].
    *   **Delay Compensation:** Compensate for temporal delay by estimating an offset $h(B(t_i, t_{i+1}))$ from input events and subtracting it:
        $$ \hat{B}(t_i, t_{i+1}) = \hat{B}'(t_i, t_{i+1}) - h(B(t_i, t_{i+1})) $$
        [S1, unofficial]

6.  **Optimization:** Jointly optimize the NeRF network and CRF models by minimizing the combined loss function:
    $$ \mathcal{L} = \lambda \mathcal{L}_{evs} + \mathcal{L}_{ldr} $$
    where:
    $$ \mathcal{L}_{evs} = \sum_{(x,y)\in \mathcal{X}} \sum_{i=0}^{b-1} ||\hat{B}(t_i, t_{i+1}) - B(t_i, t_{i+1})||^2_2 $$
    $$ \mathcal{L}_{ldr} = \sum_{(x,y)\in \mathcal{X}} ||\hat{\mathcal{I}}_{LDR}(x,y) - \mathcal{I}_{LDR}(x,y)||^2_2 $$
    and $\mathcal{X}$ denotes all pixels of the camera sensor [S1, unofficial].

**Parameters**
*   $b$: Number of time intervals minus one (total $b+1$ points). Value: 4. Context: Trade-off between performance and training time [S1, unofficial].
*   $\lambda$: Weight for event loss. Value: 0.005 [S1, unofficial].
*   $\Theta$: Contrast threshold in event generation. Definition: Minimum detectable brightness change lower bound [S1, unofficial].
*   $\Delta t_{exp}$: Exposure time. Definition: $t_{end} - t_{start}$ [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Sensor-Physics Mismatch:** Traditional methods ignore sensor-physics mismatches (e.g., applying tone mapping directly to 3D points), leading to suboptimal HDR and deblurring results because rendered outputs do not align with supervision signals [S1, unofficial].
*   **Event Delay in Low-Light:** Actual event generation may deviate from the ideal model with significant delay in low-light regions, requiring the delay compensation term $h(\cdot)$ [S1, unofficial].
*   **Quantifiable Errors:** Due to the fixed contrast threshold $\Theta$, the minimum detectable brightness change is lower-bounded, leading to quantifiable errors in radiometric estimation [S1, unofficial].

## Operating characteristics
*   **Time and Memory Complexity:** Not given in the sources.
*   **Measured Performance:** Achieves state-of-the-art HDR and deblurring novel view synthesis results on synthetic and real datasets (RealWorld-Challenge) [S1, unofficial].
*   **Scaling:** Not given in the sources.
*   **Strengths:** Explicitly models the radiance-to-measurement process for both RGB and event sensors, ensuring physical consistency. Leverages spatial and temporal dynamic information in events to enhance sharp HDR 3D representation learning [S1, unofficial].
*   **Limitations:** Requires corresponding event data and blurry LDR images. Relies on event-guided COLMAP for pose estimation during exposure [S1, unofficial].
*   **When to use:** When reconstructing sharp HDR scenes from single-exposure blurry LDR images and corresponding events, especially in extreme lighting conditions [S1, unofficial].
*   **When not to use:** When event data is unavailable or when the input images are not blurry/single-exposure [S1, unofficial].

## Background
See-NeRF was proposed to address the limitations of existing methods like EvHDR-NeRF, which extend HDR-NeRF by applying tone mapping to 3D points without considering event generation characteristics [S1, unofficial]. The method is grounded in the physical imaging principles of RGB and event sensors, aiming to bridge the gap between physical scene dynamics and sensor output [S1, unofficial]. It builds on the principles of View-Dependent Appearance and Lighting [18221-view-dependent-appearance-and-lighting.md] by using positional encoding and MLPs, but diverges by removing the view direction condition from color prediction to prevent spurious correlations, and by introducing sensor-physics grounded CRF models [18221-view-dependent-appearance-and-lighting.md][S1, unofficial].

## Sources
[S1] Seeing through Light and Darkness: Sensor-Physics Grounded Deblurring HDR NeRF from Single-Exposure Images and Events, https://arxiv.org/abs/2601.15475v4, fetched 2026-10-26 (unofficial)
