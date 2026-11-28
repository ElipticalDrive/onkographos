# Explicit Tone Mapping

Also called: Camera response modeling, Gamma Correction, Global Tone-Mapping Function, HDR to LDR Conversion, M(E, EV), Reinhard Tone Mapping, Reinhard Tone-Mapping with Exposure Scaling, Soft Clipping

## Overview
Explicit tone mapping is a post-processing step that compresses the high dynamic range (HDR) RGB image output from volume rendering into a perceptible low dynamic range (LDR) image suitable for display [S1, S2, S3]. It differs from implicit or learned tone mapping methods by using a fixed, parameterized mathematical function—typically combining exposure scaling, a global tone-mapping operator (such as Reinhard), and gamma correction—to simulate camera response and ensure the final image fits within the displayable range [S2, S3].

## How it works
**Inputs and Outputs**
*   **Input:** An HDR image $I_h$ (or pixel radiance $C_h(\mathbf{r})$) with values typically in linear RGB space, potentially exceeding the [0, 1] range [S3, S5].
*   **Parameters:** Exposure value $EV$ (or exposure time $\Delta t$), white balance coefficients $\theta_w$, and optionally a scale factor $\Phi$ or soft-clipping threshold $\tau$ [S2, S3][S4, unofficial][S5].
*   **Output:** An LDR image $I_l$ with values in the range [0, 1], typically in sRGB space [S1, S5].

**Data Structures**
*   **HDR Image:** A 2D array or tensor of floating-point values representing linear radiance per pixel [S3].
*   **White Balance Parameters:** A vector $\theta_w = [w_r, w_g, w_b]^T \in \mathbb{R}^3$ [S3].
*   **CRF Control Points (Optional):** If parameterizing the Camera Response Function (CRF) explicitly, a set of 256 control points defining a piece-wise linear function [S3].

**Steps**
The standard explicit tone mapping pipeline, as defined in HDR-NeRF and related works, proceeds as follows [S2, S3]:

1.  **Exposure Scaling:** Scale the HDR pixel value $E$ by $2^{EV}$ to simulate camera exposure. This is equivalent to multiplying by the exposure time $\Delta t = 2^{EV}$ [S2].
    $$ E' = 2^{EV} E $$
    *Note: Some implementations use a fixed scale factor $\Phi$ instead of or in addition to $EV$, e.g., $E' = \Phi \Delta t_2 E$ [S4, unofficial].*

2.  **White Balance (Optional):** Apply element-wise multiplication with white balance parameters $\theta_w$ to the HDR pixel color $C_h(\mathbf{r})$ [S3].
    $$ I_w = C_h(\mathbf{r}) \odot \theta_w $$
    The parameters $\theta_w$ are regularized to be positive [S3].

3.  **Tone Mapping Operator:** Apply a global tone-mapping function to compress the dynamic range. The most common operator is the Reinhard tone mapper [S1, S2][S4, unofficial]:
    $$ T(E') = \frac{E'}{E' + 1} $$
    *Alternative:* Some implementations use a soft clipping function $\phi(x)$ with threshold $\tau$ (e.g., $\tau=0.95$) for $x > \tau$ [S5]:
    $$ \phi(x) = \begin{cases} x & \text{if } x \leq \tau \\ \tau + \frac{x-\tau}{1-\tau} (1 - e^{-\frac{x-\tau}{1-\tau}}) & \text{if } x > \tau \end{cases} $$
    *Note: The Reinhard operator is often preferred for its simplicity and differentiability [S1, S2].*

4.  **Camera Response Function (CRF) (Optional):** If a learnable or explicit CRF $g$ is used, map the tone-mapped value $I_w$ to $I_l$ using piece-wise linear interpolation over 256 intervals [S3]. The CRF is constrained by boundary conditions $g(0)=0$ and $g(1)=1$ [S3]. For differentiability during training, leaky thresholding is applied to handle saturation:
    $$ g_{\text{leaky}}(x) = \begin{cases} \alpha x & \text{if } x < 0 \\ g(x) & \text{if } 0 \leq x \leq 1 \\ -\frac{\alpha}{\sqrt{x}} + \alpha + 1 & \text{if } x > 1 \end{cases} $$
    where $\alpha$ is a thresholding coefficient [S3].

5.  **Gamma Correction:** Apply gamma correction to convert the linear LDR value to sRGB space. The standard gamma is $\gamma = 2.2$ [S2][S4, unofficial][S5].
    $$ I_l = (T(E'))^{\frac{1}{2.2}} $$
    *Note: Some implementations omit this step if the tone mapper already accounts for it or if the target display does not require sRGB gamma [S1].*

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def explicit_tone_mapping(hdr_image, ev=0.0, white_balance=None, gamma=2.2, use_reinhard=True, tau=0.95):
    """
    Converts an HDR image to LDR using explicit tone mapping.
    
    Args:
        hdr_image: 2D array or tensor of shape (H, W, 3) with linear RGB values.
        ev: Exposure value (EV). Exposure time delta_t = 2^ev.
        white_balance: Optional vector [wr, wg, wb] for white balancing.
        gamma: Gamma value for correction (default 2.2).
        use_reinhard: If True, use Reinhard operator; else use soft clipping.
        tau: Threshold for soft clipping (default 0.95).
        
    Returns:
        ldr_image: 2D array or tensor of shape (H, W, 3) with sRGB values in [0, 1].
    """
    # Step 1: Exposure Scaling
    # delta_t = 2^ev
    delta_t = math.pow(2, ev)
    scaled_image = hdr_image * delta_t
    
    # Step 2: White Balance (Optional)
    if white_balance is not None:
        # Element-wise multiplication
        scaled_image = scaled_image * white_balance
        
    # Step 3: Tone Mapping Operator
    if use_reinhard:
        # Reinhard tone mapping: E / (E + 1)
        tone_mapped = scaled_image / (scaled_image + 1.0)
    else:
        # Soft clipping
        tone_mapped = scaled_image.copy()
        mask = scaled_image > tau
        if mask.any():
            x = scaled_image[mask]
            # phi(x) = tau + (x-tau)/(1-tau) * (1 - exp(-(x-tau)/(1-tau)))
            term = (x - tau) / (1 - tau)
            tone_mapped[mask] = tau + term * (1 - math.exp(-term))
            
    # Step 4: Gamma Correction
    # Raise to power 1/gamma
    ldr_image = tone_mapped ** (1.0 / gamma)
    
    # Clamp to [0, 1] to handle any numerical overflow
    ldr_image = np.clip(ldr_image, 0.0, 1.0)
    
    return ldr_image
```

**Edge Cases and Pitfalls**
*   **Precision:** HDR values can be very large. The Reinhard operator handles large values gracefully by asymptotically approaching 1, but floating-point overflow can occur before tone mapping if not scaled properly [S2].
*   **Saturation:** In training scenarios, saturated pixels (clipped to 0 or 1) provide no gradient information. Leaky thresholding or masking is required to propagate loss through these regions [S3].
*   **Gamma Mismatch:** Applying gamma correction twice (e.g., if the renderer already outputs sRGB) will result in incorrect colors [S1].
*   **White Balance Regularization:** White balance parameters must be constrained to positive values to maintain physical plausibility [S3].

## Subtopics

### [HDR-GS Tone Mapping via MLP](19698-hdr-gs-tone-mapping-via-mlp.md)
This method models the inverse tone-mapping function for High Dynamic Range Gaussian Splatting (HDR-GS) by transforming the input signal to a logarithmic domain, thereby enhancing Multi-Layer Perceptron (MLP) training stability and avoiding nonlinearity and discontinuity issues inherent in linear domain multiplication. It differs from traditional non-parametric calibration methods by parameterizing the Camera Response Function (CRF) as a learnable MLP, allowing for joint optimization with the scene representation.

## Operating characteristics
*   **Time Complexity:** $O(N)$ where $N$ is the number of pixels, as it involves per-pixel arithmetic operations [S2].
*   **Memory Complexity:** $O(N)$ for storing the output LDR image [S2].
*   **Performance:** Extremely fast, suitable for real-time rendering and interactive applications [S1].
*   **Strengths:** Simple to implement, differentiable (if using smooth operators like Reinhard or soft clipping), and physically interpretable via exposure and white balance parameters [S2, S3].
*   **Limitations:** Global tone mapping may not preserve local contrast as well as adaptive methods. The choice of gamma and exposure is critical and may require manual tuning or optimization [S2].
*   **When to use:** When a simple, differentiable, and fast conversion from HDR to LDR is needed, especially in neural rendering pipelines where the tone mapping parameters can be learned or optimized [S2, S3].
*   **When not to use:** When high-fidelity local contrast preservation is required, or when the dynamic range is extremely wide and requires adaptive tone mapping [S2].

## Background
Explicit tone mapping has its roots in photographic tone reproduction and computer graphics. The Reinhard tone mapping operator was introduced by Reinhard et al. in 2002 [S1, S2]. It has been widely adopted in neural radiance field (NeRF) and volume rendering pipelines to generate LDR ground truth images for training and evaluation [S2, S3]. The method is often combined with gamma correction to match the sRGB standard [S2, S5].

## Sources
[S1] Real-time interactive platform-agnostic volumetric path tracing in webGL 2.0, https://doi.org/10.1145/3208806.3208814, fetched 2026-10-28
[S2] HDR-NeRF: High Dynamic Range Neural Radiance Fields, https://xhuangcv.github.io/hdr-nerf/, fetched 2026-10-27
[S3] HDR-Plenoxels: Self-Calibrating High Dynamic Range Radiance Fields, https://doi.org/10.1007/978-3-031-19824-3_23, fetched 2026-10-26
[S4] Seeing through Light and Darkness: Sensor-Physics Grounded Deblurring HDR NeRF from Single-Exposure Images and Events, https://arxiv.org/abs/2601.15475v4, fetched 2026-10-26 (unofficial)
[S5] Neural Light Field Estimation for Street Scenes with Differentiable Virtual Object Insertion, https://doi.org/10.1007/978-3-031-20086-1_22, fetched 2026-10-30
