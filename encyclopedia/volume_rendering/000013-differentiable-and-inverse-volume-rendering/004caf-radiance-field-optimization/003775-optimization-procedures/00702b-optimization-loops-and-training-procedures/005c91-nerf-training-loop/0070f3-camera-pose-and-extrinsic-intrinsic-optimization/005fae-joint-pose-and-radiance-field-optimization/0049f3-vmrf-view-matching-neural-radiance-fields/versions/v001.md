# VMRF (View Matching Neural Radiance Fields)
Also called: View Matching NeRF

## Overview
VMRF is a method for novel view synthesis that learns Neural Radiance Field (NeRF) representations without prior knowledge of camera poses or hand-crafted camera pose distributions [S1]. It distinguishes itself from pose-free alternatives like GNeRF by estimating relative pose transformations via view matching using unbalanced optimal transport, rather than estimating absolute poses directly from images [S1]. This approach makes VMRF robust to out-of-dataset views and eliminates the need for accurate, hand-crafted camera pose distributions required by inversion networks [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A set of $n$ real RGB images $I = \{I_1, I_2, ..., I_n\}$ of a static scene. No camera poses are provided.
- **Outputs:** Optimized NeRF parameters $\Theta$ and estimated camera poses $\hat{\Phi} = \{\hat{\phi}_i, i \in [1, n]\]$.

**Data Structures:**
- **NeRF Representation:** A 5D function $F_\Theta: \mathbb{R}^3 \times S^2 \to \mathbb{R}^3 \times \mathbb{R}$ parameterized by an MLP with parameters $\Theta$, mapping a 3D point $x$ and viewing direction $d$ to color $c$ and volume density $\sigma$ [S1].
- **Camera Pose:** Represented as a $4 \times 4$ transformation matrix. Rotation is encoded as a continuous 6D embedding [S1]. Position is a 3D Euclidean embedding [S1].
- **Feature Sets:** Source feature set $f_s \in \mathbb{R}^{l \times d}$ and target feature set $f_t \in \mathbb{R}^{l \times d}$, where $l=64$ is the number of feature vectors and $d=256$ is the dimension [S1].
- **Transport Plan:** $T \in \mathbb{R}^{l \times l}$, where $T_{ij}$ represents the mass moved from source feature $i$ to target feature $j$ [S1].
- **Relative Transformation:** Output vector $[t_x, t_y, t_z, \theta_x, \theta_y, \theta_z]$ representing relative translation and rotation [S1].

**Numbered Steps:**

1. **Initialization:** Initialize a coarse NeRF with parameters $\Theta$ and randomly initialized camera poses $\Phi' = \{\phi'_i, i \in [1, n]\}$ [S1].
2. **Adversarial Pre-training:** Train the coarse NeRF using an adversarial loss to ensure generated images have a similar distribution to real images [S1].
   $$ L_{adv}(F, D) = \mathbb{E}_{I \sim P_{data}} [\log(D(I))] + \mathbb{E}_{F(\Phi') \sim P_g} [\log(1 - D(F(\Phi')))] $$
   where $F$ is the NeRF, $D$ is a discriminator, $P_{data}$ is the real image distribution, and $P_g$ is the generated image distribution [S1].
3. **Feature Extraction:** For each real image $I_i$ and its corresponding rendered image $\hat{I}_i$ (rendered using the coarse NeRF and random pose $\phi'_i$):
   - Extract source features $f_s$ from $\hat{I}_i$ using a pre-trained VGG-19 network [S1].
   - Extract target features $f_t$ from $I_i$ using a pre-trained VGG-19 network [S1].
4. **Cost Matrix Computation:** Compute the cost matrix $M \in \mathbb{R}^{l \times l}$ based on feature similarities:
   $$ M = 1 - \frac{f_s \cdot f_t^T}{\|f_s\| \|f_t\|} $$
   where $M_{ij}$ denotes the moving cost from source feature $i$ to target feature $j$ [S1].
5. **Unbalanced Optimal Transport (UOT):** Compute the optimal transport plan $T^*$ by solving the entropic regularized UOT problem:
   $$ T^* = \arg \min_T \left[ \sum_{i,j=1}^{l} T_{ij} M_{ij} - \eta H(T) + \epsilon KL(T \mathbf{1}_l || \mu_s) + \epsilon KL(T^T \mathbf{1}_l || \mu_t) \right] $$
   where:
   - $\mu_s = \sum_{i=1}^l p_i^s \delta(f_i^s)$ and $\mu_t = \sum_{j=1}^l p_j^t \delta(f_j^t)$ are discrete source and target distributions with probability masses $p_i^s, p_j^t$ [S1].
   - $H(T) = -\sum_{i,j=1}^l T_{ij} \log(T_{ij} - 1)$ is the entropic regularization term [S1].
   - $\eta > 0$ is the entropic regularization parameter (default 0.005) [S1].
   - $\epsilon > 0$ is the KL divergence regularization parameter [S1].
   - $KL$ denotes Kullback-Leibler divergence [S1].
   - The solution is computed using the Sinkhorn algorithm on the Fenchel-Legendre dual form [S1].
6. **Relative Pose Prediction:** Predict the relative transformation $[t_x, t_y, t_z, \theta_x, \theta_y, \theta_z]$ from the transport plan $T^*$ using a Relative Transformation Predictor $RTP(\cdot)$ (implemented as a Vision Transformer with modified last layer) [S1]:
   $$ [t_x, t_y, t_z, \theta_x, \theta_y, \theta_z] = RTP(T^*) $$
7. **Pose Calibration:** Construct the relative rotation matrix $\Delta R$ and translation vector $\Delta t$:
   $$ \Delta R = R_x(\theta_x) R_y(\theta_y) R_z(\theta_z) $$
   $$ \Delta t = [t_x, t_y, t_z]^T $$
   where $R_x, R_y, R_z$ are standard rotation matrices:
   $$ R_x(\theta_x) = \begin{bmatrix} 1 & 0 & 0 \\ 0 & \cos\theta_x & \sin\theta_x \\ 0 & -\sin\theta_x & \cos\theta_x \end{bmatrix}, \quad R_y(\theta_y) = \begin{bmatrix} \cos\theta_y & 0 & -\sin\theta_y \\ 0 & 1 & 0 \\ \sin\theta_y & 0 & \cos\theta_y \end{bmatrix}, \quad R_z(\theta_z) = \begin{bmatrix} \cos\theta_z & \sin\theta_z & 0 \\ -\sin\theta_z & \cos\theta_z & 0 \\ 0 & 0 & 1 \end{bmatrix} $$
   Estimate the calibrated camera pose $\hat{\phi}_i$ by applying the relative transformation to the random pose $\phi'_i$:
   $$ \hat{\phi}_i = \begin{bmatrix} \Delta R & \Delta t \\ 0 & 1 \end{bmatrix} \phi'_i $$
   [S1].
8. **NeRF Optimization:** Optimize the NeRF parameters $\Theta$ using the calibrated poses $\hat{\Phi}$ and the photometric loss:
   $$ L_P(F, \hat{\Phi}) = \frac{1}{n} \sum_{i=1}^n \| I_i - F(\hat{\phi}_i) \|^2 $$
   [S1].
9. **Iterative Refinement:** Repeat steps 3-8 in an end-to-end hybrid iterative optimization scheme [S1].

**Volume Rendering Equation:**
To render an image, samples are taken along camera rays. For a ray $r$, the color $c_r$ is:
$$ c_r = \sum_{i=1}^N T_{ri} \alpha_{ri} c_{ri} $$
where:
- $N$ is the number of random samples along the ray [S1].
- $c_{ri}$ and $\sigma_{ri}$ are the color and volume density of the $i$-th sample [S1].
- $\delta_{ri} = x_{ri+1} - x_{ri}$ is the distance between adjacent samples [S1].
- $\alpha_{ri} = 1 - \exp(\sigma_{ri} \delta_{ri})$ is the alpha value [S1].
- $T_{ri} = \prod_{j=1}^{i-1} (1 - \alpha_{rj})$ is the transmittance [S1].

**Parameters:**
- $l = 64$: Number of feature vectors in each feature set [S1].
- $d = 256$: Dimension of feature vectors [S1].
- $\eta = 0.005$: Entropic regularization parameter for UOT smoothness [S1].
- $\epsilon$: KL divergence regularization parameter (value not specified in sources) [S1].

**Edge Cases and Pitfalls:**
- **Low Texture/Repetitive Patterns:** Classical Structure-from-Motion (SfM) fails here due to few detectable keypoints. VMRF avoids this by not relying on SfM [S1].
- **Out-of-Dataset Views:** Inversion networks (e.g., in GNeRF) are disturbed by views beyond the training dataset if accurate pose distributions are unavailable. VMRF mitigates this by using relative pose estimation via view matching [S1].
- **Occlusion:** Causes inconsistent total mass between source and target feature distributions. Standard Optimal Transport fails by matching outliers. VMRF uses Unbalanced OT (UOT) with KL divergence penalties to handle this [S1].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** VMRF outperforms state-of-the-art methods (like GNeRF) qualitatively and quantitatively on Synthetic-NeRF and DTU datasets [S1].
- **Scaling:** Not given in the sources.
- **Strengths:** Does not require prior knowledge of camera poses or hand-crafted pose distributions; robust to out-of-dataset views; handles low-texture scenes where SfM fails; handles occlusions via UOT [S1].
- **Limitations:** Requires pre-trained VGG-19 for feature extraction; relies on adversarial training for coarse initialization; computational cost of UOT and Sinkhorn iterations [S1].
- **When to use:** When camera poses are unknown or unreliable, and in scenes with low texture or repetitive patterns [S1].
- **When not to use:** When accurate camera poses are available and SfM works reliably, standard NeRF training may be simpler [S1].

## Background
VMRF was proposed by Jiahui Zhang, Fangneng Zhan, Rongliang Wu, Yingchen Yu, Wenqing Zhang, Bai Song, Xiaoqin Zhang, and Shijian Lu from Nanyang Technological University, Max Planck Institute for Informatics, Wenzhou University, and ByteDance [S1]. It was published in 2022 [S1]. The method addresses limitations of pose-free NeRFs like GNeRF, which require accurate, hand-crafted camera pose distributions for training inversion networks and are susceptible to out-of-dataset views [S1]. VMRF introduces view matching via unbalanced optimal transport to estimate relative poses, removing the need for prior pose knowledge [S1].

## Sources
[S1] VMRF: View Matching Neural Radiance Fields, https://doi.org/10.1145/3503161.3548078, fetched 2026-10-29
