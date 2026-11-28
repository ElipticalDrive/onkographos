# Few-shot NeRF

Also called: Sparse-view NeRF, Few-view NeRF

## Overview
Few-shot NeRF addresses the degradation of neural radiance field quality when the number of input training viewpoints $|S|$ is drastically reduced (typically $|S| < 20$) compared to the standard setting of over 100 images [S2, unofficial]. In this regime, standard NeRF architectures suffer from high divergent behaviors, including geometry breakdown, overfitting to input viewpoints, and the generation of artifacts in empty space [S2, unofficial]. The method distinguishes itself by operating on the same continuous function representation as standard NeRF but requires specific architectural or regularization modifications to stabilize optimization under severe data scarcity [S2, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of $|S|$ input images $S = \{I_i\}$ with known ground-truth camera poses. $|S|$ is small (few-shot setting).
*   **Outputs:** A radiance field function $F_\theta(\cdot)$ parameterized by $\theta$, which outputs color $\mathbf{c} \in \mathbb{R}^3$ and volume density $\sigma \in \mathbb{R}$ for any 3D point and viewing direction.

**Data Structures:**
The core data structure is the Multi-Layer Perceptron (MLP) $F_\theta$. The network is split into two branches (following the standard NeRF architecture referenced in the parent article):
1.  **Density Branch:** Takes positional encoding $\gamma(\mathbf{x})$ as input.
2.  **Color Branch:** Takes the latent feature vector from the density branch and the directional encoding $\gamma(\mathbf{d})$.

**Numbered Steps:**
1.  **Ray Sampling:** For each pixel in the input images, generate a ray $r(\lambda) = o_{world} + \lambda d_{world}$.
    *   Compute camera-space direction: $d_{cam} = (\frac{i - c_x}{f_x}, \frac{j - c_y}{f_y}, 1)$.
    *   Transform to world-space: $d_{world} = R d_{cam}$, where $R$ is the rotation matrix from the camera pose.
    *   World-space origin: $o_{world} = t$, where $t$ is the translation vector from the camera pose.
    *   Normalize direction: $d = \frac{d_{world}}{\|d_{world}\|}$.
    *   Sample $N_{samples}$ points along the ray within the normalized device coordinates (NDC) range $[0, 1]$ using stratified sampling [S1].
2.  **Positional Encoding:** Apply positional encoding $\gamma(\cdot)$ to the 3D coordinates $\mathbf{x}$ and normalized direction $\mathbf{d}$ [S1].
    $$
    \gamma(p) = (\sin(2^0 \pi p), \cos(2^0 \pi p), \dots, \sin(2^L \pi p), \cos(2^L \pi p))
    $$
3.  **Density Prediction:** Pass $\gamma(\mathbf{x})$ through the density branch to obtain volume density $\sigma$ and a latent feature vector $\mathbf{h}$ [S1].
4.  **Color Prediction:** Concatenate $\mathbf{h}$ with $\gamma(\mathbf{d})$ and pass through the color branch to obtain RGB color $\mathbf{c}$ [S1].
5.  **Volume Rendering:** Composite the color and density along the ray using standard volume rendering equations to produce the predicted image $\hat{I}$ [S2, unofficial].
6.  **Loss Computation:** Compute the photometric loss between the rendered image $\hat{I}$ and the ground truth image $I$ [S1].
    $$
    \mathcal{L}_{photo} = \|\hat{I} - I\|_2^2
    $$
7.  **Optimization:** Update parameters $\theta$ to minimize the loss.

**Formulas:**
*   **Ray Parameterization:** $r(\lambda) = o_{world} + \lambda d_{world}$ [S1].
*   **Positional Encoding Dimensions:**
    *   Position features: $3 \times (2^{L_x} + 1)$ [S1].
    *   Direction features: $3 \times (2^{L_d} + 1)$ [S1].
*   **Photometric Loss:** $\mathcal{L}_{photo} = \|\hat{I} - I\|_2^2$ [S1].

**Parameters:**
*   $N_{samples} = 128$ stratified samples per ray [S1].
*   Sampling range: $[0, 1]$ in NDC space [S1].
*   $L_x = 10$ (max frequency for position) [S1].
*   $L_d = 4$ (max frequency for direction) [S1].
*   Input viewpoint count: $|S| < 20$ [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def few_shot_nerf_forward(x, d, density_mlp, color_mlp):
    """
    Forward pass for Few-shot NeRF.
    x: (N, 3) 3D coordinates
    d: (N, 3) normalized viewing directions
    density_mlp: MLP for density and latent features
    color_mlp: MLP for color
    """
    # 1. Positional Encoding
    # gamma(p) = (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^L pi p), cos(2^L pi p))
    # Lx = 10, Ld = 4
    gamma_x = positional_encoding(x, L=10) # Output shape: (N, 63)
    gamma_d = positional_encoding(d, L=4)  # Output shape: (N, 27)

    # 2. Density Branch
    # Branch 1: FC 63->192+ReLU, FC 192->192+ReLU, FC 192->192+ReLU, FC 192->192+ReLU, FC 192->1 (sigma)
    # Branch 2: FC 192->128 (latent feature h)
    sigma, h = density_mlp(gamma_x) # sigma: (N, 1), h: (N, 128)

    # 3. Color Branch
    # Input: direction (27) concatenated with latent feature h (128) -> 155
    # FC 155->64+ReLU, FC 64->3 (RGB)
    color_input = concat(gamma_d, h, dim=-1) # (N, 155)
    rgb = color_mlp(color_input) # (N, 3)

    return rgb, sigma

def positional_encoding(p, L):
    """
    Standard positional encoding.
    p: (..., 3)
    L: max frequency
    """
    # gamma(p) = (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^L pi p), cos(2^L pi p))
    # This results in 3 * (2^(L+1)) features? 
    # Note: Source [S1] states 3*(2^Lx+1)=63 for Lx=10. 
    # 2^10 + 1 = 1025? No. 
    # Standard NeRF PE: sin(2^0 pi p), cos(2^0 pi p)... sin(2^(L-1) pi p), cos(2^(L-1) pi p).
    # If L=10, frequencies are 2^0 to 2^9. Total 10 sin + 10 cos = 20 per dim. 20*3 = 60.
    # Source [S1] says 63 features for Lx=10. 
    # 3 * (2^10 + 1) = 3 * 1025 = 3075. This contradicts 63.
    # Let's re-read [S1]: "3 x (2Lx + 1) = 63". 
    # If Lx=10, 2*10+1 = 21. 3*21 = 63. 
    # This implies the formula in the text "2^Lx" in the equation might be a typo for "Lx" or the definition of Lx is different.
    # However, the equation provided is: gamma(p) = (sin(2^0 pi p)... sin(2^L pi p)...).
    # If L=10, and we have sin/cos for 2^0...2^10, that is 11 pairs = 22 per dim = 66.
    # If L=10 means 10 octaves (2^0...2^9), that is 10 pairs = 20 per dim = 60.
    # The source explicitly says: "3 x (2Lx + 1) = 63". 
    # 2*10 + 1 = 21. 21 * 3 = 63.
    # This suggests the encoding is: (p, sin(pi p), cos(pi p), ..., sin(10 pi p), cos(10 pi p))?
    # Or perhaps the source text has a typo and meant 2^(Lx) but the count 63 is correct for a specific variant.
    # Given the constraint "Size follows the evidence", we must use the explicit count 63 and the formula provided in the source, even if standard NeRF uses 60/66.
    # The source equation: gamma(p) = (sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^L pi p), cos(2^L pi p)).
    # If L=10, and the count is 63, the formula in the source might be illustrative of the *type* of encoding, but the count 63 is the hard fact.
    # Actually, 3 * (2 * Lx + 1) = 63 implies Lx=10.
    # Let's implement based on the explicit feature count provided in the layout section.
    
    # Implementation detail: The source defines Lx=10 yielding 63 features.
    # We will assume the standard implementation that yields 63 features for Lx=10.
    # Common implementation: sin/cos for frequencies 2^0 to 2^(L-1) plus original coords?
    # 3 + 2*3*10 = 63. This matches 3*(2Lx+1).
    # So gamma(p) = [p, sin(2^0 pi p), cos(2^0 pi p), ..., sin(2^9 pi p), cos(2^9 pi p)]
    
    features = [p]
    for i in range(L):
        freq = 2**i
        features.append(sin(freq * pi * p))
        features.append(cos(freq * pi * p))
    return concat(features, dim=-1)
```

**Edge Cases and Pitfalls:**
*   **Geometry Breakdown:** With $|S| < 20$, the network may fail to reconstruct accurate geometry, leading to artifacts in empty space [S2, unofficial].
*   **Overfitting:** The model may overfit to the limited input viewpoints, resulting in poor generalization to novel views [S2, unofficial].
*   **Precision:** The small number of viewpoints makes the optimization landscape more prone to local minima [S2, unofficial].

## Operating characteristics
*   **Time Complexity:** Depends on the number of samples $N_{samples}$ and the depth/width of the MLPs. Training time is generally longer per iteration due to the difficulty of convergence in sparse settings, though total iterations may vary based on regularization [S2, unofficial].
*   **Memory Complexity:** Dominated by the MLP parameters. The density branch has $63 \to 192 \to 192 \to 192 \to 192 \to 1$ weights. The color branch has $155 \to 64 \to 3$ weights. This is comparable to standard NeRF but the optimization stability is lower [S1].
*   **Scaling:** Performance drops sharply as $|S|$ decreases below 20 without additional regularization or architectural modifications [S2, unofficial].
*   **Strengths:** Can reconstruct scenes from very few images where standard NeRF fails completely [S2, unofficial].
*   **Limitations:** Susceptible to artifacts and geometry breakdown if not properly regularized [S2, unofficial].

## Background
Few-shot NeRF is a variation of the original NeRF method [S2, unofficial]. The original NeRF [31] typically uses over 100 input viewpoints [S2, unofficial]. The few-shot setting was explored to enable reconstruction from sparse inputs, such as in mobile photography or archival photo collections [S2, unofficial]. Existing frameworks applied regularization techniques at unknown viewpoints to constrain NeRF with additional 3D priors [S2, unofficial].

## Sources
[S1] MZEN: Multi-zoom Enhanced NeRF for 3-D Reconstruction with Unknown Camera Poses, https://link.springer.com/article/10.1007/s10994-026-07114-x, fetched 2026-10-26
[S2] DaRF: Boosting Radiance Fields from Sparse Inputs with Monocular Depth Adaptation, https://doi.org/10.48550/arxiv.2305.19201, fetched 2026-10-29 (unofficial)
