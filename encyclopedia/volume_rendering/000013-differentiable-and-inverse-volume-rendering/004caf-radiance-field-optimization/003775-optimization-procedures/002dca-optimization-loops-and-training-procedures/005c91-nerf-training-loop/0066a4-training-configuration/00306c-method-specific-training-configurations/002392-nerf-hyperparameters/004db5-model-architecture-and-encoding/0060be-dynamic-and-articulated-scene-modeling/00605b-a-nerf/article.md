# A-NeRF
Also called: Articulated Neural Radiance Fields

## Overview
A-NeRF learns a user-specific neural 3D body model and underlying skeleton pose from unlabelled monocular videos by extending Neural Radiance Fields (NeRFs) with a skeleton to apply to time-varying and articulated motion [S1, unofficial]. Its core technical novelty is defining spatial latent variables relative to the pose of body parts to overcome ill-posed inverse operations with an overparameterization, embedding domain knowledge of how humans move [S1, unofficial]. Unlike [MoCo-NeRF](16197-moco-nerf.md) which uses residual-based conditioning, A-NeRF uses skeleton-relative embeddings; unlike [SelfNeRF](18160-selfnerf.md) which uses surface-based SDFs, A-NeRF uses volumetric density conditioned on bone-relative coordinates [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A sequence of $N$ images $I_k \in \mathbb{R}^{H \times W \times 3}$ of the same person. Initial pose estimates $\hat{\theta}_k$ from an off-the-shelf estimator (e.g., SPIN [26]).
*   **Outputs:** Refined skeleton poses $\theta_k$, neural network parameters $\phi$ (defining shape and appearance), and rendered images $C_\phi(\theta_k)$.

**Data Structures:**
1.  **Skeleton Representation:**
    *   Rest pose: 3D joint locations $\tilde{a}_m$ for $m=1..24$, root at 0, connected via $B$ bones.
    *   Pose $\theta_k$: Vector containing relative rotations of 24 joints $\tilde{\theta}_{k,m}$ to parents and global position $\tilde{\theta}_{k,0}$.
2.  **Neural Network Architecture ($F_\phi$):**
    *   Follows original NeRF [42] structure.
    *   **Inputs:** Positional Encoded (PE) query point $\Gamma(q)$, PE view direction $\Gamma(d)$, appearance code, and Relative Ray encoding.
    *   **Hidden Layers:** 10 fully-connected layers with sizes $[256, 256, 256, 256, 256, 256, 256, 256, 256, 256, 128]$.
    *   **Activations:** ReLU for all layers except the last two. Sigmoid for the final output layer.
    *   **Specific Injection:** Appearance code and Relative Ray encoding are concatenated into the network at the second-to-last fully-connected layer (before the final 128-unit layer) to prevent them from affecting body geometry [S1, unofficial].
    *   **Outputs:** Density $\sigma \in \mathbb{R}$ and color $c \in \mathbb{R}^3$.

**Numbered Steps:**
1.  **Initialization:** Initialize camera intrinsics, bone lengths $a_m$, and pose $\theta_k$ using SPIN [26] for every frame $k$.
2.  **Encoding:** Map input scene point $q$ and view direction $d$ to higher dimensional space using Positional Encoding $\Gamma$ [S1, unofficial].
3.  **Skeleton-Relative Encoding (Rel. Ray):**
    *   For a query point $q$ and frame $k$, compute the vector from $q$ to each joint $m$: $\tilde{q}_{k,m} = q - \tilde{a}_m(\theta_k)$.
    *   Compute the angle between the ray direction $\tilde{d}_k$ and the vector to joint $m$: $d'_{k,m} = \arccos(\tilde{d}_k \cdot \tilde{q}_{k,m})$.
    *   Concatenate these angles for all 24 joints to form the Relative Ray encoding [S1, unofficial].
4.  **Network Forward Pass:** Feed $\Gamma(q)$, $\Gamma(d)$, appearance code, and Rel. Ray encoding into $F_\phi$ to obtain $\sigma$ and $c$ [S1, unofficial].
5.  **Volume Rendering:** Render the image $C_\phi(\theta_k)$ via classical ray-marching using coarse-to-fine importance sampling. Restrict samples to be within a coarse human silhouette estimated with [10] unless the background is monochrome [S1, unofficial].
6.  **Loss Calculation:** Compute the total loss $L_{SV}$ [S1, unofficial]:
    $$ L_{SV}(\theta, \phi) = \sum_k \| C_\phi(\theta_k) - I_k \|_1 + \lambda_\theta \sum_k d(\theta_k - \hat{\theta}_k) + \lambda_t \sum_k \left\| \frac{\partial^2 \theta_k}{\partial t^2} \right\|^2 $$
    Where:
    *   Data term: L1 distance between generated and input images.
    *   Pose regularizer: $d(x) = \min(\|x\|_2 - \epsilon, 0)$ with $\epsilon = 0.01$. Encourages solution to stay close to initial pose $\hat{\theta}_k$.
    *   Smoothness prior: Penalizes acceleration $\frac{\partial^2 \theta_k}{\partial t^2}$ between consecutive frames.
7.  **Optimization Loop:**
    *   Form a training batch by randomly sampling $N_{batch} = 3072$ rays from all available images.
    *   Accumulate gradient update $\nabla \theta$ for $G \times (30 \times N_{batch}) / N$ iterations so that each $\theta_k$ is expected to be sampled 30 times before update.
    *   Optimize poses $\theta_k$ and neural parameters $\phi$ for 500k iterations on objective Eq. 2.
    *   Stop optimizing pose and continue training $\phi$ with only the data term for additional 200k iterations [S1, unofficial].

**Parameters:**
*   $\lambda_t$: Hyperparameter balancing the smoothness prior term.
*   $\lambda_\theta$: Hyperparameter balancing the pose regularizer term.
*   $\epsilon$: Tolerance for small shifts in the pose regularizer, value 0.01.
*   Cutoff distance $t$: 500mm, with exponential decay over 250k iterations.
*   Cutoff temperature $\tau$: Starts at -20, decays exponentially to -200 over 500k iterations.
*   Coarse samples: 64.
*   Fine samples: 16.
*   Positional encoding frequencies for input ($L_{input}$): 7.
*   Positional encoding frequencies for view ($L_{view}$): 4.
*   Learning rate for $\phi$: Starts at $5 \times 10^{-4}$, decays exponentially over 200k iterations.
*   Learning rate for $\phi$ (finetune): $2 \times 10^{-4}$ decaying to $5 \times 10^{-5}$.
*   Gradient accumulation steps $G$: 20 for Mixamo, 50 for Human 3.6M, 10 for MonoPerfCap.

**Pitfalls:**
*   Using World Ray direction causes overfitting in static/estimated camera settings, leading to flickering results for novel view rendering (e.g., bullet-time effect) [S1, unofficial].
*   Encoding world coordinates directly fails on motions and has large memory consumption. Encoding positions relative to skeleton has high dimensionality [S1, unofficial].
*   Straightforward baselines conditioning directly on pose (World Position + Joint Position + $\theta$) perform worst, achieving 0.7 SSIM by simply rendering background colors [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Training takes approximately 60 hours on 2 Nvidia Tesla V100 32GB GPUs [S1, unofficial].
*   **Memory Complexity:** High due to per-bone embeddings and large MLP. Encoding positions relative to the skeleton has high dimensionality compared to world coordinates [S1, unofficial].
*   **Measured Performance:**
    *   On Human 3.6M S9 Protocol II: PSNR 26.64, SSIM 0.9192, PA-MPJPE 41.00 with per-image code and Rel. Ray [S1, unofficial].
    *   On SURREAL: PSNR 29.10, SSIM 0.9655, compared to NeuralBody PSNR 23.86, SSIM 0.9304 [S1, unofficial].
*   **Rendering Time:** 1-4 seconds per 512x512x3 image on a single Nvidia Tesla V100 GPU [S1, unofficial].
*   **Strengths:** Works better than NeuralBody for longer sequences and retargeting to diverse poses. Can be trained with only a single view; adding more views does not help if poses are not diverse enough. One long video with diverse poses is comparable to multiple shorter multi-view recordings [S1, unofficial].
*   **Limitations:** Requires rough 3D pose estimates for initialization. Does not require surface supervision or initialization, but relies on off-the-shelf predictors which may be inaccurate [S1, unofficial].

## Background
A-NeRF builds on NeRF [42] and uses appearance code design from [36, 49]. It was proposed to learn a generative neural body model from unlabelled monocular videos by extending Neural Radiance Fields (NeRFs) with a skeleton to apply to time-varying and articulated motion [S1, unofficial]. The method addresses the ill-posed inverse problem of mapping 3D world coordinates to a canonical space for articulated bodies by overparameterizing with one embedding per bone [S1, unofficial].

## Sources
[S1] A-NeRF: Articulated Neural Radiance Fields for Learning Human Shape, Appearance, and Pose, https://doi.org/10.48550/arxiv.2102.06199, fetched 2026-10-30 (unofficial)
