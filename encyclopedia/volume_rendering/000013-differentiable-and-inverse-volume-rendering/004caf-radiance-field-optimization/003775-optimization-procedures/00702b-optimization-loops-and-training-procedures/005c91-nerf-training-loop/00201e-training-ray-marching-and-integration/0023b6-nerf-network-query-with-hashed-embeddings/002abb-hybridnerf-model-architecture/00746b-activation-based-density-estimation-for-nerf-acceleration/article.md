# Activation-based Density Estimation for NeRF Acceleration

Also called: Activation-derived density estimate, Weight estimate from activations

## Overview
This method accelerates inference in pre-trained Neural Radiance Fields (NeRFs) by deriving density estimates from intermediate layer activations of the coarse MLP, allowing for reduced sampling or early termination without modifying the network architecture or training protocol [S1]. Unlike coarse-to-fine sampling which uses two separate NeRFs, this method uses intermediate activations from a single coarse NeRF to derive weights, reducing computational requirements by up to 50% with only a slight hit in rendering quality [S1]. Unlike sampling networks that predict density with reduced capacity, this method leverages the existing capacity of the trained coarse NeRF's intermediate features [S1].

## How it works
**Inputs and Outputs:**
- **Input:** A pre-trained coarse NeRF network $\Theta_{\text{coarse}}$, a camera ray $\vec{r}$ defined by origin $\vec{o}$ and direction $\vec{d}$, near/far bounds $t_n, t_f$, and a layer index $\ell$.
- **Output:** An estimated color $\hat{\mathcal{C}}(\vec{r})$ and optionally a boolean flag indicating whether the fine pass was executed.

**Data Structures:**
- **Ray Samples:** A set of positions $t_i$ for $i=1 \dots N_c$ uniformly distributed in $[t_n, t_f]$.
- **Positional Encoding:** $\gamma(\cdot)$ maps 3D coordinates to a higher-dimensional vector.
- **Activation Feature Vector:** $f_\ell \in \mathbb{R}^{N_s}$, where $N_s$ is the number of samples. This is derived from the raw activation matrix $A^{(\ell)} \in \mathbb{R}^{N_s \times N_h}$ by computing the mean over the hidden units dimension $N_h$:
  $$ f_{\ell, i} = \frac{1}{N_h} \sum_{k=1}^{N_h} A^{(\ell)}_{i,k} $$
- **Density Estimate:** $\hat{d}_i$ derived from $f_{\ell, i}$.
- **Weight Estimate:** $\hat{w}_i$ derived from $\hat{d}_i$.

**Numbered Steps:**

1.  **Ray Casting and Encoding:** For each pixel, generate a ray $\vec{r}(t) = \vec{o} + t\vec{d}$. Place $N_c$ samples uniformly at positions $t_i \in [t_n, t_f]$. Apply positional encoding $\gamma(\cdot)$ to each sample position $t_i$ [S1].

2.  **Coarse Network Forward Pass:** Pass the encoded samples through the coarse NeRF MLP $\Theta_{\text{coarse}}$. Retrieve the raw activation matrix $A^{(\ell)}$ for a specific intermediate layer $\ell$ [S1].

3.  **Feature Extraction:** Compute the activation feature vector $f_\ell \in \mathbb{R}^{N_c}$ by averaging the activations across the hidden dimension $N_h$ for each sample [S1]:
    $$ f_{\ell, i} = \frac{1}{N_h} \sum_{k=1}^{N_h} A^{(\ell)}_{i,k} $$

4.  **Density Estimation:** Derive a density estimate $\hat{d}_i$ from the activation feature $f_{\ell, i}$. The method relies on the observation that trained NeRFs map samples with high volumetric density $\sigma$ to local minima in the activation feature space along a ray [S1].
    *Note: The specific mathematical function mapping $f_{\ell, i}$ to $\hat{d}_i$ is not explicitly defined in the provided sources, other than the qualitative relationship that minima in $f_\ell$ correspond to high $\sigma$ [S1].* [G: In implementation, this typically involves normalizing the activations or applying a monotonic transformation that inverts the "minima = high density" relationship, such as $\hat{d}_i = \max(f_\ell) - f_{\ell, i}$ or similar scaling, but the exact formula is not given in the sources.]

5.  **Weight Estimation:** Convert the density estimates $\hat{d}_i$ into weights $\hat{w}_i$ that approximate the standard NeRF weights $w_i = T_i(1 - \exp(-\sigma_i \delta_i))$. These weights are normalized such that $\sum \hat{w}_i = 1$ to form a piecewise-constant probability density function (PDF) along the ray [S1].

6.  **Fine Pass Sampling (Optional):** Use inverse transform sampling with the derived weight estimate $\hat{w}$ to generate a new set of $N_f$ samples for the fine pass [S1].

7.  **Early Termination (Optional):** To further reduce inference time, perform the fine pass only if a condition based on the activation value $v_\ell$ and a threshold $\mu(V_\ell)$ is met. Specifically, if $v_\ell < \mu(V_\ell)$, the fine pass is skipped [S1]. Here, $v_\ell$ likely refers to a scalar summary of the activations (e.g., mean or min) and $\mu(V_\ell)$ is a learned or statistical threshold [G: The exact definition of $v_\ell$ and $\mu(V_\ell)$ is not provided in the sources, but it serves as a gate for computational cost].

8.  **Rendering:**
    - If the fine pass is executed, render using all $N_c + N_f$ samples with the fine network $\Theta_{\text{fine}}$.
    - If the fine pass is skipped, render using only the $N_c$ coarse samples with $\Theta_{\text{coarse}}$.
    - The color $\hat{\mathcal{C}}(\vec{r})$ is estimated using the standard volume rendering equation:
      $$ \hat{\mathcal{C}}(\vec{r}) = \sum_{i} T_i (1 - \exp(-\sigma_i \delta_i)) \vec{c}_i $$
      where $\delta_i = t_{i+1} - t_i$ and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ [S1].

**Parameters:**
- $N_s$: The number of samples used to estimate the ray color in the standard NeRF formulation [S1].
- $N_c$: The number of samples placed uniformly by the coarse NeRF [S1].
- $N_f$: The number of samples generated for the fine pass via inverse transform sampling [S1].

**Edge Cases and Pitfalls:**
- **Unbounded Scenes:** For unbounded 3D scenes, coordinates must be mapped using the spatial contraction function $\text{contract}(\cdot)$ from Mip-NeRF 360 to map $\mathbb{R}^3$ to a ball of radius 2:
  $$ \text{contract}(\vec{x}) = \begin{cases} \vec{x} & \|\vec{x}\| \leq 1 \\ \left(2 - \frac{1}{\|\vec{x}\|}\right)\left(\frac{\vec{x}}{\|\vec{x}\|}\right) & \|\vec{x}\| > 1 \end{cases} $$
  [S1]
- **Precision:** The method relies on the stability of intermediate activations. Degenerate inputs (e.g., rays with no intersection) may produce undefined activation patterns. [G: Standard NeRF handling of empty space should be applied.]

## Operating characteristics
- **Time Complexity:** Reduces computational requirements of trained NeRFs by up to 50% [S1]. This is achieved by avoiding the full fine-pass network evaluation for a significant portion of pixels/rays.
- **Memory Complexity:** No additional memory overhead compared to standard coarse-to-fine NeRFs, as it reuses the existing coarse network activations [S1].
- **Strengths:** Does not require retraining or architectural changes; leverages existing network capacity; simple to implement as a post-processing step on activations.
- **Limitations:** Slight hit in rendering quality compared to standard coarse-to-fine sampling [S1]. Performance depends on the correlation between activation minima and density, which may vary across different NeRF architectures or datasets.
- **When to use:** When inference speed is critical and a small quality trade-off is acceptable.
- **When not to use:** When maximum rendering quality is required or when the activation-density correlation is weak for the specific scene/network.

## Background
- **Origin:** Proposed in "Analyzing the Internals of Neural Radiance Fields" [S1].
- **History:** The method builds on the observation that NeRF-MLPs naturally partition themselves into structural and color layers [S1]. It contrasts with sampling networks that predict density with reduced capacity, arguing that the coarse NeRF's deep features already contain valuable density information [S1].
- **Usage:** Applied to accelerate inference in synthetic and real-world NeRF datasets [S1].

## Sources
[S1] Analyzing the Internals of Neural Radiance Fields, https://doi.org/10.1109/cvprw63382.2024.00288, fetched 2026-10-26
