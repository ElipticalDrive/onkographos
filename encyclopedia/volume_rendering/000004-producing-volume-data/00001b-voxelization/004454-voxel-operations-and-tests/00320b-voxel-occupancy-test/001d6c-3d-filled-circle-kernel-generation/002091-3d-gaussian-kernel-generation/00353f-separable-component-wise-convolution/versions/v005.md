# Separable Component-Wise Convolution

Also called: Decomposed Low-Rank Tensor Convolution, Separable component-wise convolution on decomposed low-rank tensor

## Overview
Separable Component-Wise Convolution is a method for applying 3D Gaussian smoothing to decomposed low-rank tensor volumes without reconstructing the full 3D grid, thereby preserving space compression [S1][S2, unofficial]. It solves the problem of high computational and memory costs associated with naive 3D convolution on dense volumes by leveraging the algebraic structure of the tensor decomposition [S1][S2, unofficial]. Unlike traditional separated kernel convolution which only separates the kernel and requires sequential 1D convolutions on the full 3D volume, this method convolves individual vector and matrix components separately, reducing time complexity from $O(I \cdot J \cdot K \cdot L_N^3)$ to $O(I \cdot J \cdot L_N + J \cdot K \cdot L_N + K \cdot I \cdot L_N)$ [S1][S2, unofficial].

## How it works
**Inputs:**
- Decomposed low-rank tensor volume $T$ with rank $R$ [S2, unofficial].
- Tensor dimensions $I, J, K$ along axes $X, Y, Z$ [S1].
- Gaussian standard deviation $\sigma$.
- Kernel size parameter $L_N$.

**Outputs:**
- Convoluted tensor components $\tilde{v}_{\sigma,r}$ and $\tilde{M}_{\sigma,r}$ [S1][S2, unofficial].
- Density field $\tilde{F}_\sigma(x)$ and spectral restricted color field $\vec{F}_c(x, \vec{d})$ [S1][S2, unofficial].

**Data Structures:**
The decomposed tensor consists of:
- Vector components: $v_r^X \in \mathbb{R}^I$, $v_r^Y \in \mathbb{R}^J$, $v_r^Z \in \mathbb{R}^K$ for $r=1 \dots R$ [S1].
- Matrix components: $M_r^{Y,Z} \in \mathbb{R}^{J \times K}$, $M_r^{X,Z} \in \mathbb{R}^{I \times K}$, $M_r^{X,Y} \in \mathbb{R}^{I \times J}$ for $r=1 \dots R$ [S1].
- Color basis vectors: $b_r \in \mathbb{R}^G$ for $r=1 \dots R$ (used for color field reconstruction) [S1].

**Steps:**

1. **Generate 1D Gaussian Kernel:**
   Create a 1D kernel $N_{1d}(\sigma)$ of size $L_N$ [S1]. Sample from the continuous Gaussian distribution and clamp values to a maximum of 1.0 [S1].
   $$
   N_{1d}(\sigma)_x = \begin{cases} 
   \min\left(1, \sqrt{2\pi}\sigma \cdot e^{-\frac{x^2}{2\sigma^2}}\right) & \text{if } \sigma > 0.0001 \\
   \delta[x] & \text{if } \sigma < 0.001 
   \end{cases}
   $$
   where $x$ ranges from $-L_N/2$ to $L_N/2$, and $\delta[x]$ is the discrete impulse function (1 at center, 0 elsewhere) [S1].

2. **Generate 2D Gaussian Kernel:**
   Compute the 2D kernel $N_{2d}(\sigma)$ as the outer product of the 1D kernel with itself:
   $$
   N_{2d}(\sigma) = N_{1d}(\sigma) \otimes N_{1d}(\sigma)
   $$
   where $\otimes$ denotes the outer product [S1].

3. **Convolve Vector Components:**
   Apply 1D Gaussian convolution ($*_{1d}$) using $N_{1d}(\sigma)$ to each vector component:
   $$
   \tilde{v}_{\sigma,r} = N_{1d}(\sigma) *_{1d} v_r
   $$
   This produces convoluted vectors $\tilde{v}_{\sigma,r}^X, \tilde{v}_{\sigma,r}^Y, \tilde{v}_{\sigma,r}^Z$ [S1][S2, unofficial].

4. **Convolve Matrix Components:**
   Apply 2D Gaussian convolution ($*_{2d}$) using $N_{2d}(\sigma)$ to each matrix component:
   $$
   \tilde{M}_{\sigma,r} = N_{2d}(\sigma) *_{2d} M_r
   $$
   This produces convoluted matrices $\tilde{M}_{\sigma,r}^{Y,Z}, \tilde{M}_{\sigma,r}^{X,Z}, \tilde{M}_{\sigma,r}^{X,Y}$ [S1][S2, unofficial].

5. **Reconstruct Convolved Tensor:**
   Combine the individually convoluted components to form the convoluted tensor $\tilde{T}_\sigma$:
   $$
   \tilde{T}_\sigma = \sum_{r=1}^{R} \left( \tilde{v}_{\sigma,r}^X \otimes \tilde{M}_{\sigma,r}^{Y,Z} + \tilde{v}_{\sigma,r}^Y \otimes \tilde{M}_{\sigma,r}^{X,Z} + \tilde{v}_{\sigma,r}^Z \otimes \tilde{M}_{\sigma,r}^{X,Y} \right)
   $$
   where $\otimes$ denotes the outer product between the vector and matrix components along their respective axes [S1][S2, unofficial].

6. **Compute Fields:**
   - **Density Field:** Sample linearly from $\tilde{T}_\sigma$:
     $$
     \tilde{F}_\sigma(x) = \tilde{T}_\sigma(x)
     $$
   - **Color Field:** For color, combine convoluted components with basis vectors $b_r$ to form $\tilde{T}_c$, then apply spectral function $S$:
     $$
     \tilde{T}_c = \sum_{r=1}^{R} \left( \tilde{v}_{c,r}^X \otimes \tilde{M}_{c,r}^{Y,Z} \otimes b_r + \tilde{v}_{c,r}^Y \otimes \tilde{M}_{c,r}^{X,Z} \otimes b_r + \tilde{v}_{c,r}^Z \otimes \tilde{M}_{c,r}^{X,Y} \otimes b_r \right)
     $$
     $$
     \vec{F}_c(x, \vec{d}) = S(\tilde{T}_c(x), \vec{d})
     $$
     [S1][S2, unofficial].

**Parameters:**
- $L_N$: Size of the discrete Gaussian kernel in each dimension [S1].
- $\sigma$: Standard deviation of the Gaussian kernel. When $\sigma < 0.001$, use discrete impulse $\delta$ to avoid numerical instability [S1].
- Kernel Scale Factor: Randomly sampled from uniform distribution $[0, 1]$ to mitigate weak/noisy gradients [S1].

**Pitfalls:**
- Naive 3D convolution ($N_{3d} *_{3d} T_\sigma$) requires reconstructing the full 3D volume, destroying space compression and increasing time complexity to $O(I \cdot J \cdot K \cdot L_N^3)$ and space complexity to $O(I \cdot J \cdot K)$ [S1][S2, unofficial].
- Traditional separated kernel convolution separates the kernel but not the input signal, requiring three sequential 1D convolutions on the full 3D volume with time complexity $O(I \cdot J \cdot K \cdot L_N)$ and space complexity $O(I \cdot J \cdot K)$ [S1][S2, unofficial].

## Subtopics

### [Dr. TVAM Convolution](8425-dr-tvam-convolution.md)
A separable 3D convolution method using Gaussian diffusion kernels, implemented via Dr.Jit's `dr.convolve` function for tomographic volumetric additive manufacturing simulations. It builds on [Separable Component-Wise Convolution](7446-separable-component-wise-convolution.md) by applying specific Gaussian diffusion kernels rather than generic low-rank tensor decompositions.

## Operating characteristics
- **Time Complexity:** Computing convoluted components is $O(I \cdot J \cdot L_N + J \cdot K \cdot L_N + K \cdot I \cdot L_N)$, assuming 2D Gaussian convolution on matrix components is separated into 1D Gaussian convolutions [S1][S2, unofficial]. Each query sample is $O(R)$ [S1][S2, unofficial].
- **Space Complexity:** $O(R \cdot (IJ + JK + KI))$ for storing the decomposed components and their convolved versions, significantly less than the $O(I \cdot J \cdot K)$ required for full volume reconstruction [S1][S2, unofficial].
- **Performance:** Drastically reduces computation required for filtering 3D radiance fields compared to naive or traditional methods [S1][S2, unofficial].
- **Strengths:** Preserves space compression of decomposed low-rank tensors; computationally efficient for large volumes; enables coarse-to-fine training schedules [S1][S2, unofficial].
- **Limitations:** Requires the input to be in decomposed low-rank tensor format; relies on the validity of the component-wise convolution identity (Theorem 4) [S1][S2, unofficial].
- **When to use:** When applying Gaussian smoothing to decomposed low-rank tensor volumes (e.g., TensoRF) for spectrum limiting in radiance field optimization [S1][S2, unofficial].
- **When not to use:** When the data is not in decomposed low-rank tensor format, or when full 3D convolution is explicitly required for non-separable operations [S1][S2, unofficial].

## Background
The method was proposed in the context of improving robustness for joint optimization of camera pose and decomposed low-rank tensorial radiance fields, specifically addressing issues with inaccurate poses and suboptimal reconstruction quality [S1][S2, unofficial]. It was introduced in the paper "Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields" published at AAAI-24 [S1]. The method leverages the separability of the decomposed low-rank tensor structure to perform efficient 3D Gaussian convolution, avoiding the need to reconstruct the full 3D volume [S1][S2, unofficial].

## Sources
[S1] Improving Robustness for Joint Optimization of Camera Pose and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.1609/aaai.v38i2.27859, fetched 2026-10-26
[S2] Improving Robustness for Joint Optimization of Camera Poses and Decomposed Low-Rank Tensorial Radiance Fields, https://doi.org/10.48550/arxiv.2402.13252, fetched 2026-10-26 (unofficial)
[S3] Tensor-Train-based 4D Time-Dependent Volume Rendering, https://repositorio.ie.edu/entities/publication/348991b8-1dff-4bea-acbc-433ce708313d/full, fetched 2026-10-29 (unofficial)
