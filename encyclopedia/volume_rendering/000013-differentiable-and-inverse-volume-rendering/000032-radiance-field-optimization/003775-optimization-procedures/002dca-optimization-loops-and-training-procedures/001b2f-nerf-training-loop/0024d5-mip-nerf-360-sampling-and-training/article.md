# Mip-NeRF 360 Sampling and Training

Also called: Mip-NeRF 360 Training, Mip-NeRF 360 Proposal Sampling

## Overview
Mip-NeRF 360 Sampling and Training extends the Mip-NeRF framework to unbounded, 360-degree scenes by introducing a disparity-based ray parameterization, a contraction function for coordinate space, and a coarse-to-fine training loop with a proposal network. It solves the artifacts of "floaters" and "background collapse" common in unbounded scene reconstruction by regularizing the density distribution and ensuring efficient sampling through online distillation. It differs from standard NeRF by using linear-in-disparity sampling and from basic Mip-NeRF by handling unbounded scenes via coordinate contraction and disparity parameterization.

## How it works
**Inputs:**
- Set of training rays $\mathcal{R}$ with ground truth colors $\mathbf{C}^*(\mathbf{r})$.
- Near and far clip distances $t_n, t_f$ for each ray.

**Outputs:**
- Rendered image colors $\mathbf{C}(\mathbf{r})$.
- Trained parameters for the Proposal MLP $\Theta_{prop}$ and the NeRF MLP $\Theta_{NeRF}$.

**Data Structures:**
- **Proposal MLP ($\Theta_{prop}$):** A small MLP predicting volumetric density $\hat{\sigma}$, converted to proposal weights $\hat{\mathbf{w}}$.
- **NeRF MLP ($\Theta_{NeRF}$):** A larger MLP predicting weights $\mathbf{w}$ and color $\mathbf{c}$.
- **Ray Samples:** Sorted arrays of normalized distances $\mathbf{s}$ and weights $\mathbf{w}$.

**Coordinate Transformation and Contraction:**
To handle unbounded scenes, input coordinates $\mathbf{x}$ are mapped to a ball of radius 2 using the contraction function [S1]:
$$
\text{contract}(\mathbf{x}) = \begin{cases} 
\mathbf{x} & \text{if } \|\mathbf{x}\| \le 1 \\
\left(2 - \frac{1}{\|\mathbf{x}\|}\right) \frac{\mathbf{x}}{\|\mathbf{x}\|} & \text{if } \|\mathbf{x}\| > 1 
\end{cases}
$$
Gaussian intervals $(\boldsymbol{\mu}, \boldsymbol{\Sigma})$ are transformed using the linear approximation of the contraction function $f$ (where $f$ is `contract`) [S1]:
$$
f(\boldsymbol{\mu}, \boldsymbol{\Sigma}) = \left(f(\boldsymbol{\mu}), \mathbf{J}_f(\boldsymbol{\mu}) \boldsymbol{\Sigma} \mathbf{J}_f(\boldsymbol{\mu})^T\right)
$$
where $\mathbf{J}_f(\boldsymbol{\mu})$ is the Jacobian of $f$ at $\boldsymbol{\mu}$ [S1].

**Ray Parameterization:**
Ray distances are parameterized by an invertible mapping between Euclidean distance $t$ and normalized distance $s \in [0, 1]$ [S1]:
$$
s \triangleq \frac{g(t) - g(t_n)}{g(t_f) - g(t_n)}, \quad t \triangleq g^{-1}(s \cdot g(t_f) + (1-s) \cdot g(t_n))
$$
The function $g(x)$ is set to $1/x$ to produce ray samples distributed linearly in disparity (inverse depth) [S1]. Setting $g(x) = \log(x)$ yields logarithmic spacing [S1].

**Training Loop Steps:**
1. **Initialization:** Initialize $\Theta_{prop}$ and $\Theta_{NeRF}$ randomly [S1].
2. **Coarse Sampling:** For each ray $\mathbf{r} \in \mathcal{R}$, sample $N_c$ points uniformly in normalized disparity space $s \in [0, 1]$. Convert to $t$ using the inverse mapping. Evaluate $\Theta_{prop}$ to get proposal weights $\hat{\mathbf{w}}^c$ [S1].
3. **Fine Sampling:** Resample $N_f$ points from the proposal distribution $\hat{\mathbf{w}}^c$ to get fine samples $\mathbf{t}^f$. Evaluate $\Theta_{NeRF}$ to get fine weights $\mathbf{w}^f$ and colors $\mathbf{c}^f$ [S1].
4. **Coarse NeRF Evaluation:** Evaluate $\Theta_{NeRF}$ on the coarse samples $\mathbf{t}^c$ to get coarse weights $\mathbf{w}^c$ and colors $\mathbf{c}^c$ [S1].
5. **Rendering:** Compute rendered colors $\mathbf{C}(\mathbf{r}, \mathbf{t}^c)$ and $\mathbf{C}(\mathbf{r}, \mathbf{t}^f)$ using standard volume rendering integration [S1].
6. **Reconstruction Loss:** Compute the weighted sum of reconstruction losses for coarse and fine rays [S1]:
   $$
   \mathcal{L}_{\text{recon}} = \sum_{\mathbf{r} \in \mathcal{R}} \frac{1}{10} \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{r}, \mathbf{t}^c), \mathbf{C}^*(\mathbf{r})) + \mathcal{L}_{\text{recon}}(\mathbf{C}(\mathbf{r}, \mathbf{t}^f), \mathbf{C}^*(\mathbf{r}))
   $$
   where $\mathcal{L}_{\text{recon}}$ is mean squared error [S1].
7. **Proposal Loss:** Compute the proposal loss $\mathcal{L}_{\text{prop}}$ to ensure proposal weights bound NeRF weights [S1].
   - Define the bound function:
     $$
     \text{bound}(\hat{\mathbf{t}}, \hat{\mathbf{w}}, T) = \sum_{j: T \cap \hat{T}_j \neq \emptyset} \hat{w}_j
     $$
   - Compute the loss:
     $$
     \mathcal{L}_{\text{prop}}(\mathbf{t}, \mathbf{w}, \hat{\mathbf{t}}, \hat{\mathbf{w}}) = \sum_i \frac{1}{w_i} \max(0, w_i - \text{bound}(\hat{\mathbf{t}}, \hat{\mathbf{w}}, T_i))^2
     $$
   - Apply a stop-gradient on the NeRF MLP's outputs $\mathbf{t}$ and $\mathbf{w}$ when computing $\mathcal{L}_{\text{prop}}$ [S1].
   - Compute $\mathcal{L}_{\text{prop}}$ efficiently using summed-area tables because $\hat{\mathbf{t}}$ and $\mathbf{t}$ are sorted [S1].
8. **Total Loss:** Minimize $\mathcal{L}_{\text{recon}} + \lambda \mathcal{L}_{\text{prop}}$ (where $\lambda$ is a weighting factor, typically 1) using gradient descent [S1].

**Pitfalls:**
- The proposal loss is asymmetric; it only penalizes proposal weights for underestimating the NeRF distribution. Overestimates are expected as proposal weights form an upper envelope [S1].
- Floaters (small disconnected dense regions) and background collapse (distant surfaces modeled as close semi-transparent clouds) are mitigated by the regularization and parameterization [S1].

## Operating characteristics
- **Time Complexity:** The use of summed-area tables for the proposal loss allows efficient computation, scaling linearly with the number of samples [S1].
- **Memory Complexity:** Requires storing weights and distances for both coarse and fine samples, plus the MLP parameters [S1].
- **Strengths:** Handles unbounded scenes effectively; reduces floaters and background collapse; efficient sampling via disparity-based parameterization and proposal network [S1].
- **Limitations:** Requires careful tuning of the contraction function and disparity spacing; the proposal network adds complexity to the training loop [S1].

## Background
Mip-NeRF 360 was introduced by Barron et al. in 2022 [S1]. It builds on the Mip-NeRF framework [S1] to address the challenges of unbounded, 360-degree scenes. The method introduces coordinate contraction and disparity-based sampling to improve reconstruction quality and efficiency [S1].

## Sources
[S1] Mip-NeRF 360: Unbounded Anti-Aliased Neural Radiance Fields, https://doi.org/10.1109/cvpr52688.2022.00539, fetched 2026-10-26
