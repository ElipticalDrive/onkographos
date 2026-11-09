# MIMO-NeRF

Also called: MIMO-NeRF-distill, MIMO-NeRF-naive

## Overview
MIMO-NeRF is a neural rendering method that accelerates inference by replacing the standard Single-Input Single-Output (SISO) Multi-Layer Perceptron (MLP) with a Multi-Input Multi-Output (MIMO) MLP [S1]. It solves the computational bottleneck of evaluating the radiance field for every sample point individually by grouping neighboring samples on a ray and processing them in parallel [S1]. Unlike [AdaNeRF](6195-adanerf.md) which adapts to view-dependent appearance, or [Sparse and Efficient Representations](9326-sparse-and-efficient-representations.md) which change the data structure, MIMO-NeRF retains the canonical MLP architecture but modifies its input/output dimensions to reduce the number of forward passes (# Run) per pixel [S1]. It distinguishes itself from naive parallelization by employing a self-supervised learning framework with 3D consistency losses to resolve the ambiguity in density and color that arises when multiple points are predicted simultaneously [S1].

## How it works
**Inputs and Outputs:**
The method takes a set of $N$ samples along a ray as input during training and inference. The MIMO MLP $f_{\text{MIMO}}$ accepts $N_p$ grouped positions $\mathbf{x}_i, \dots, \mathbf{x}_j$ and the viewing direction $\mathbf{d}$, and outputs $N_p$ colors $\mathbf{c}_i, \dots, \mathbf{c}_j$ and densities $\sigma_i, \dots, \sigma_j$ [S1].

**Data Structures and Layout:**
The network architecture is identical to the baseline NeRF [S1] except for the input and output dimensions [S1].
*   **Positional Encoding:** Raw coordinates $\mathbf{x} \in \mathbb{R}^3$ and direction $\mathbf{d} \in \mathbb{R}^3$ are encoded using $\gamma(\mathbf{x})$ and $\gamma(\mathbf{d})$ [S1].
*   **Density Network ($F_\sigma$):** Takes the concatenated encoded positions of the group. For $N_p$ samples, the input dimension for position encoding is $N_p \times 63$ (assuming $L=10$ for position) [S1]. It consists of 8 hidden layers with ReLU activations, 256 hidden units, and skip connections from the input to the 5th layer [S1].
*   **Color Network ($F_c$):** Takes the intermediate feature vector from the density network (output of the last hidden layer before the density head) and the concatenated encoded directions of the group ($N_p \times 27$) [S1]. It uses a 1-layer MLP (ReLU, 128 units) followed by a linear layer to output RGB [S1].
*   **Scaling Factors ($R_m$):** To handle the increased input dimensionality, the network is reformulated with scaling factors $R_m$ for each input stream $m \in \{1, \dots, N_p\}$ [S1]. For $N_p=2$, $R_1=R_2=1$. For $N_p=4$, $R_1=1, R_2=2$. For $N_p=8$, $R_1=1, R_2=2, R_3=4$ [S1].

**Numbered Steps:**
1.  **Grouping:** Divide the $N$ samples on a ray into $N/N_p$ groups of $N_p$ samples each, starting from the sample nearest to the camera [S1].
2.  **Forward Pass:** For each group $i$ (where $j = i + N_p - 1$), compute:
    $$ (\mathbf{c}_i, \dots, \mathbf{c}_j, \sigma_i, \dots, \sigma_j) = f_{\text{MIMO}}(\mathbf{x}_i, \dots, \mathbf{x}_j, \mathbf{d}) $$
    [S1]
3.  **Rendering:** Compute the pixel color $C(\mathbf{r})$ using standard volumetric rendering integration with the predicted $\sigma$ and $\mathbf{c}$ values [S1].
4.  **Training Objective:** Minimize $\mathcal{L}_{\text{MIMO}} = \mathcal{L}_{\text{pixel}}^{\text{MIMO}} + \lambda \mathcal{L}_{\text{3D}}$ [S1].
    *   $\mathcal{L}_{\text{pixel}}^{\text{MIMO}}$ is the standard pixel-wise MSE loss between rendered and ground truth colors [S1].
    *   $\mathcal{L}_{\text{3D}}$ is the 3D consistency loss, comprising color and alpha consistency losses [S1].
5.  **3D Consistency Loss:** Calculate consistency between different reformulated MLPs (or different views/groups) using asymmetric weighting [S1].
    $$ \mathcal{L}_{\text{color 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\mathbf{c}_i^{m_1} - \text{sg}(\mathbf{c}_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\mathbf{c}_i^{m_1}) - \mathbf{c}_i^{m_2}\|_2^2 \right] $$
    $$ \mathcal{L}_{\text{alpha 3D}} = \sum_{m_1=1}^{M-1} \sum_{m_2=m_1+1}^M \frac{1}{N} \sum_{i=1}^N \left[ \mu_{m_1}^{m_2} \|\alpha_i^{m_1} - \text{sg}(\alpha_i^{m_2})\|_2^2 + \mu_{m_2}^{m_1} \|\text{sg}(\alpha_i^{m_1}) - \alpha_i^{m_2}\|_2^2 \right] $$
    [S1]
    where $\text{sg}$ denotes the stop-gradient operation [S1].
6.  **Asymmetric Weighting:** Compute weights $\mu_{mj}^{mi}$ to reduce the loss effect for more reliable predictions [S1]:
    $$ \mu_{mj}^{mi} = \frac{\sqrt{m_j} \sqrt{R_{\max}}}{\sqrt{R_{m_i}}} $$
    where $R_{\max} = \max_{m \in \{1, \dots, M\}} R_m$ [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_ray_mimo(ray_origin, ray_direction, near, far, Nc, Nf, Np, model, lambda_3d=1.0):
    """
    Renders a single ray using MIMO-NeRF.
    """
    # 1. Sample points
    # Coarse sampling
    t_coarse = stratified_sample(near, far, Nc)
    x_coarse = ray_origin + ray_direction * t_coarse
    
    # Fine sampling (hierarchical)
    # ... (standard NeRF hierarchical sampling logic) ...
    t_fine = hierarchical_sample(x_coarse, model, near, far, Nf)
    x_fine = ray_origin + ray_direction * t_fine
    
    # 2. Group samples
    # Combine coarse and fine samples if using hierarchical sampling, 
    # or just group fine samples. The paper defines #Run based on Nc and Nf.
    # Assuming we group the final set of samples used for color prediction.
    all_samples = concatenate(t_coarse, t_fine) # Or just t_fine depending on implementation detail
    # The paper calculates #Run = Nc/Np + (Nc+Nf)/Np. 
    # This implies grouping is applied to both coarse and fine passes.
    
    # Group Coarse
    groups_coarse = group_samples(all_samples[:Nc], Np)
    # Group Fine
    groups_fine = group_samples(all_samples[Nc:], Np)
    
    # 3. Forward Pass MIMO
    # For each group, predict Np densities and colors
    densities = []
    colors = []
    
    for group in groups_coarse:
        # group is list of Np positions
        # Input: positions (Np, 3), direction (3)
        # Output: colors (Np, 3), densities (Np, 1)
        c, s = model.forward_mimo(group, ray_direction)
        densities.append(s)
        colors.append(c)
        
    for group in groups_fine:
        c, s = model.forward_mimo(group, ray_direction)
        densities.append(s)
        colors.append(c)
        
    # 4. Flatten and Render
    # Flatten lists of arrays into single arrays
    sigma = flatten(densities)
    c_rgb = flatten(colors)
    
    # Compute alpha and transmittance
    # delta is distance between samples
    deltas = compute_deltas(all_samples)
    alpha = 1.0 - exp(-sigma * deltas)
    T = cumprod(1.0 - alpha)
    
    # Volume rendering
    C = sum(T * alpha * c_rgb)
    
    return C

def compute_3d_loss(model, rays, lambda_3d):
    """
    Computes the self-supervised 3D consistency loss.
    """
    # The paper uses multiple reformulated MLPs (M) or views to compute consistency.
    # For MIMO-NeRF-self, it jointly optimizes teacher and student.
    # Here we assume M represents the different reformulations or views.
    
    L_color_3d = 0
    L_alpha_3d = 0
    
    # Iterate over pairs of reformulations/views m1, m2
    for m1 in range(M):
        for m2 in range(m1 + 1, M):
            # Get predictions from both reformulations
            c_m1, s_m1 = model.forward_reformulated(m1, rays)
            c_m2, s_m2 = model.forward_reformulated(m2, rays)
            
            # Compute asymmetric weights
            R_max = max(R)
            mu_m1_m2 = sqrt(m2) * sqrt(R_max) / sqrt(R_m1)
            mu_m2_m1 = sqrt(m1) * sqrt(R_max) / sqrt(R_m2)
            
            # Stop gradient on one side
            c_m2_sg = stop_gradient(c_m2)
            c_m1_sg = stop_gradient(c_m1)
            s_m2_sg = stop_gradient(s_m2)
            s_m1_sg = stop_gradient(s_m1)
            
            # MSE Loss
            L_color_3d += mu_m1_m2 * mean((c_m1 - c_m2_sg)**2) + mu_m2_m1 * mean((c_m2 - c_m1_sg)**2)
            L_alpha_3d += mu_m1_m2 * mean((s_m1 - s_m2_sg)**2) + mu_m2_m1 * mean((s_m2 - s_m1_sg)**2)
            
    L_3d = L_color_3d + L_alpha_3d
    return L_3d
```

**Parameters:**
*   $N_p$: Number of grouped samples. Typical values: 2, 4, 8 [S1].
*   $N_c, N_f$: Number of coarse and fine samples. Blender: $N_c=64, N_f=128$. LLFF: $N_c=64, N_f=64$ [S1].
*   $\lambda$: Weight for 3D consistency loss. Blender: 1. LLFF: 0.4 [S1].
*   $R_m$: Scaling factors for reformulated MLPs. $N_p=2 \rightarrow R_1=R_2=1$; $N_p=4 \rightarrow R_1=1, R_2=2$; $N_p=8 \rightarrow R_1=1, R_2=2, R_3=4$ [S1].

**Edge Cases and Pitfalls:**
*   **Ambiguity:** Pixel-wise loss alone does not regularize 3D representations, leading to ambiguity in color and density for individual points [S1]. This is mitigated by the 3D consistency loss [S1].
*   **Asymmetric Weights:** When $N_p=2$, $R_1=R_2=1$, making the asymmetric weight ratio unity. Consequently, asymmetric weights are disabled and not ablated for this configuration [S1].
*   **Parameter Increase:** The number of parameters increases with $N_p$ because the input dimension of $\gamma(\mathbf{x})$ scales by $N_p$. For $N_p=2$, parameters increase by 1.06x; $N_p=4$, 1.17x; $N_p=8$, 1.39x [S1].

## Subtopics

### MIMO-TensoRF
Application of MIMO-NeRF to the TensoRF architecture [S1]. It improves inference and training speed without deteriorating image quality by selecting appropriate $N_p$ [S1]. For the Blender dataset, $N_p \le 2$ retains image quality; for LLFF, $N_p \le 4$ achieves comparable quality [S1]. It uses tensor rank parameters $R_\sigma, R_c$ (e.g., $R_\sigma=16, R_c=48$) [S1].

## Operating characteristics
**Time and Memory Complexity:**
The number of MLP runs per pixel is reduced to $\# \text{Run} = \frac{N_c}{N_p} + \frac{N_c + N_f}{N_p}$ [S1]. In baseline NeRF, $N_p=1$.
*   **Blender ($N_c=64, N_f=128$):**
    *   $N_p=1$: 192 runs.
    *   $N_p=2$: 96 runs.
    *   $N_p=4$: 48 runs.
    *   $N_p=8$: 24 runs [S1].
*   **FLOPs:** MIMO-NeRF reduces FLOPs significantly. For Blender, $N_p=8$ reduces FLOPs from 303.82 (NeRF) to 52.72 [S1].

**Measured Performance:**
*   **Blender ($N_p=2$):** PSNR 31.26, SSIM 0.953, LPIPS 0.054. Inference time: 5.15s (RTX 3080 Ti). Training time: 5.36h (A100) [S1].
*   **LLFF ($N_p=2$):** PSNR 27.70, SSIM 0.870, LPIPS 0.155. Inference time: 4.55s. Training time: 3.97h [S1].
*   **Comparison:** MIMO-NeRF-self outperforms MIMO-NeRF-naive and MIMO-NeRF-distill in PSNR, SSIM, and LPIPS [S1]. It achieves a superior speed-quality trade-off compared to NeRF-few and NeRF-small when adjusted for comparable FLOPs [S1].

**Strengths and Limitations:**
*   **Strengths:** High compatibility with existing NeRF pipelines; reduces inference time significantly while maintaining quality; self-supervised training avoids the need for a separate teacher model [S1].
*   **Limitations:** Parameter count increases with $N_p$; requires careful selection of $N_p$ to balance speed and quality; ambiguity in 3D representation requires additional loss terms [S1].

## Background
MIMO-NeRF was introduced in the paper "MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields" [S1]. It addresses the high computational cost of standard NeRF by parallelizing sample evaluation. The method builds on the canonical NeRF representation [S1] and extends it to multi-input multi-output configurations. It is evaluated on the Blender and LLFF datasets [S1].

## Sources
[S1] MIMO-NeRF: Fast Neural Rendering with Multi-input Multi-output Neural Radiance Fields, https://arxiv.org/abs/2310.01821v1, fetched 2026-10-26
