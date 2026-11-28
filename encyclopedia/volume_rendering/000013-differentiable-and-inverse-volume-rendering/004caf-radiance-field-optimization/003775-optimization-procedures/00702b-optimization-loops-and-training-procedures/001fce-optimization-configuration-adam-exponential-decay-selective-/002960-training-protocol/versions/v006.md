# Training Protocol

Also called: DCCVT Optimization Setup, DCCVT Training Configuration

## Overview
The Training Protocol defines the standard optimization procedure for neural networks in differentiable volume rendering, encompassing optimizer settings, learning rate scheduling, and data augmentation strategies [S2]. It addresses the instability of training coordinate-based neural fields by providing a robust configuration for convergence, particularly when dealing with noisy real-world scan data or complex geometric priors [S2]. It sets itself apart from generic optimization loops by explicitly integrating domain-specific augmentations (such as Gaussian noise for point clouds) and specific regularization techniques (like CVT for discretization stability) into the core training loop [S1, S2].

## How it works
The protocol establishes a baseline configuration for training neural implicit representations, with specific variations for different architectural needs (e.g., pilot networks, variable views) [S2, S3, S5].

### Inputs and Outputs
*   **Inputs:** Input data (point clouds, voxel grids, or radiographic projections), target values (SDF, density, color), and hyperparameters (optimizer settings, batch size, epochs) [S2, S5][S6, unofficial].
*   **Outputs:** Trained neural network weights and biases [S2].

### Data Structures
*   **Optimizer State:** Maintains first and second moment estimates for Adam/AdamW [G].
*   **Learning Rate Scheduler:** Tracks current learning rate based on epoch/step count [S2, S5][S6, unofficial].
*   **Pilot Network (Optional):** A small MLP used for early-stage coarse geometry estimation [S3].

### Numbered Steps
1.  **Initialization:**
    *   Initialize the main neural network parameters [G].
    *   For methods like DCCVT, initialize both the main model and auxiliary structures (like MTet sites) with the same initial sites [S1].
    *   If using a Pilot Network, initialize it with two hidden layers, each containing 64 neurons [S3].
    *   Normalize input coordinates and target values to the range $[-1, 1]$ to match the output range of activation functions like $\sin(\cdot)$ [S5].
    *   Initialize weights using a standard scheme (e.g., Kaiming uniform or the protocol by Sitzmann et al. for SIREN-like networks) [S5].
2.  **Early Training (Pilot Phase - Optional):**
    *   If a Pilot Network is employed, use it for the first $N_{pilot}$ steps (e.g., 300 steps) [S3].
    *   The pilot network functions as a standard volume rendering model to aid in finding coarse geometry, preventing convergence failure caused by feature integration of all samples in the early stage [S3].
    *   After $N_{pilot}$ steps, discard the pilot network and switch to the main framework (e.g., VFR) for the remaining training steps [S3].
3.  **Main Training Loop:**
    *   For each epoch or iteration:
        a.  **Data Augmentation:** Apply random scaling and translation to input data [S2]. For point cloud inputs, add Gaussian noise with standard deviation $\sigma = 0.5$ (assuming unit cube cells) to simulate real scan noise [S2]. For variable-view methods (e.g., X-GRM), randomly select the number of views from a set (e.g., $\{6, 8, 10\}$) evenly sampled from available projections [S5].
        b.  **Forward Pass:** Compute the rendered output (e.g., SDF, density, color) using the current network weights [G]. If using structured latent codes, compute their contribution with a higher learning rate multiplier (e.g., $10\times$) [S4, unofficial].
        c.  **Loss Computation:** Compute the loss using the same functions as the baseline method (e.g., UNDC) [S2]. For DCCVT, incorporate CVT regularization to maintain discretization stability, distinct from pure Chamfer distance optimization [S1].
        d.  **Backward Pass:** Compute gradients with respect to network parameters [G].
        e.  **Optimization Step:** Update parameters using the Adam or AdamW optimizer [S2, S5][S6, unofficial]. Apply learning rate decay as specified (e.g., halve every 100 epochs, cosine annealing, or multistep) [S2, S5][S6, unofficial].
4.  **Post-Training:**
    *   Save the model weights. For MoE-INR, use 16-bit floating point for storage while training/inference uses 32-bit [S5].

### Formulas
*   **Adam Optimizer Update:**
    $$ \theta_{t+1} = \theta_t - \frac{\eta \cdot \hat{m}_t}{\sqrt{\hat{v}_t} + \epsilon} $$
    Where $\eta$ is the learning rate, $\hat{m}_t$ is the bias-corrected first moment, and $\hat{v}_t$ is the bias-corrected second moment [S2, S5].
*   **Gaussian Noise Augmentation:**
    $$ p_{noisy} = p_{clean} + \mathcal{N}(0, \sigma^2) $$
    Where $\sigma = 0.5$ for point cloud inputs [S2].

### Parameters
*   **Optimizer:** Adam or AdamW [S2, S5][S6, unofficial].
*   **Learning Rate ($\eta$):**
    *   Standard: $1 \times 10^{-4}$ [S5] or $5 \times 10^{-4}$ [S1] or $0.0001$ [S2].
    *   MoE-INR: $2 \times 10^{-5}$ [S5].
    *   UVA: Decay from $5 \times 10^{-4}$ to $5 \times 10^{-6}$ [S4, unofficial].
*   **Beta Coefficients ($\beta_1, \beta_2$):**
    *   Common: $(0.9, 0.999)$ [S2, S5][S6, unofficial].
    *   DCCVT: $(0.8, 0.99)$ [S1].
*   **Batch Size:**
    *   Standard: 1 (per shape) [S2].
    *   UVA: 1024 [S4, unofficial].
    *   MoE-INR: 16,000 [S5].
*   **Iterations/Epochs:**
    *   DCCVT: 1000 iterations [S1].
    *   Neural Dual Contouring: 400 epochs (SDF/voxel) or 250 epochs (point cloud) [S2].
    *   UVA: 300K iterations [S4, unofficial].
    *   MoE-INR: 30 pre-training epochs + 270 training epochs [S5].
*   **Precision:**
    *   Standard: 32-bit float [S5].
    *   X-GRM: bFloat16 [S5].

### Edge Cases and Pitfalls
*   **Convergence Failure:** In early training, feature integration of all samples can cause convergence failure. Using a pilot network for the first 300 steps mitigates this [S3].
*   **Noisy Data:** Real scan data requires heavy augmentation, including Gaussian noise ($\sigma=0.5$) and random transformations, to prevent overfitting to artifacts [S2].
*   **Discretization Stability:** For methods like DCCVT, omitting CVT regularization can lead to instability, especially with non-uniform site distributions [S1].

## Subtopics

### [Learned Initializations for Neural Fields](6509-learned-initializations-for-neural-fields.md)
This method addresses the inefficiency of optimizing coordinate-based neural networks from random weights for each new signal by applying meta-learning to derive optimal initial weight parameters. It sets itself apart from standard training protocols by decoupling the initialization phase from the specific instance optimization, allowing the network to start closer to the solution manifold of a target signal class (e.g., faces, 3D shapes). This results in faster convergence and improved generalization when only partial observations are available.

### [Training Protocol with Augmented Backgrounds and Density Prior](7043-training-protocol-with-augmented-backgrounds-and-density-pri.md)
This protocol defines a specific training schedule for differentiable volume rendering that combines background augmentation with density regularization to suppress floating artifacts and stabilize convergence. It addresses the instability of immediate background augmentation and the emergence of scene-matching floaters by enforcing a strict temporal sequence: constant-color initialization, followed by augmented backgrounds, and finally density priors. It differs from standard training by explicitly decoupling the learning of general scene structure from the refinement of density bias and object compactness.

### [Network Training Settings](7193-network-training-settings.md)
This article specifies the activation functions, dropout regularization, and latent space dimensionality validation for neural networks in radiance-field optimization. It addresses stabilizing training and preventing overfitting in differentiable rendering pipelines by defining architectural hyperparameters. It builds on [Training Protocol](4528-training-protocol.md).

### Pilot Network for Early Training
A small neural network used during the early training stage to aid the model in finding coarse geometry, preventing convergence failure caused by feature integration of all samples [S3]. It uses two hidden layers, each with 64 neurons, and is only used in the first 300 training steps. After 300 steps, the pilot network is discarded, and the main framework (e.g., VFR) takes over for the remaining training steps (e.g., total 20K steps) [S3].

### UVA Training Configuration
Optimizes the UVA model using Adam with learning rate decay from $5 \times 10^{-4}$ to $5 \times 10^{-6}$ [S4, unofficial]. Structured latent codes have a $10\times$ learning rate relative to the base rate [S4, unofficial]. The motion field and canonical field are jointly trained for 300K iterations with a batch size of 1024 [S4, unofficial]. Training takes about 10 hours using two NVIDIA Tesla V100 GPUs [S4, unofficial].

### [X-GRM Training Configuration](13049-x-grm-training-configuration.md)
Optimizes the X-GRM model using AdamW with a batch size of 8 for 100 epochs. The initial learning rate is $2 \times 10^{-5}$, with $\beta_1 = 0.9$ and $\beta_2 = 0.999$. It uses bFloat16 precision for model storage, while training uses 32-bit float. It employs variable-view learning, randomly selecting views from available projections.

### MoE-INR Training Configuration
Pre-trains the model for 30 epochs, then trains for 270 epochs [S5]. Coordinates and target values are normalized to $[-1, 1]$ [S5]. Uses Adam with initial learning rate $2 \times 10^{-5}$, $\beta_1 = 0.9$, $\beta_2 = 0.999$, and a multistep scheduler for decay [S5]. Uses 32-bit float for training/inference and 16-bit for storage [S5]. Batch size is 16,000 [S5]. Parameter initialization follows the protocol by Sitzmann et al. [S5].

## Operating characteristics
*   **Time Complexity:** Depends on the number of iterations and batch size. For example, UVA training takes ~10 hours on two V100s for 300K iterations [S4, unofficial].
*   **Memory Complexity:** Large batch sizes (MoE-INR: 16,000) require significant memory [S5].
*   **Strengths:** Robust to noisy data with appropriate augmentation [S2]. Pilot networks prevent early-stage convergence failure [S3]. CVT regularization ensures discretization stability [S1].
*   **Limitations:** Requires careful tuning of learning rate schedules and augmentation parameters. Pilot networks add complexity to the early training phase [S3].
*   **When to Use:** Use for training neural implicit representations in volume rendering, especially when dealing with noisy point clouds or requiring stable discretization [S1, S2]. Use pilot networks for difficult initial geometry estimation [S3].

## Background
The training protocol for neural volume rendering has evolved from standard optimization techniques (Adam, learning rate decay) to more specialized configurations addressing the unique challenges of implicit representations [S2]. Early work focused on basic convergence [S2], while later methods introduced specific regularizations (CVT) [S1], pilot networks for stability [S3], and large-scale configurations for efficiency [S5][S6, unofficial]. The protocol is widely used in differentiable rendering pipelines for tasks like surface reconstruction, view synthesis, and 3D shape editing [S2][S4, unofficial].

## Sources
[S1] DCCVT: Differentiable Clipped Centroidal Voronoi Tessellation, https://arxiv.org/abs/2601.13603v1, fetched 2026-10-26
[S2] Neural Dual Contouring, https://arxiv.org/abs/2202.01999v3, fetched 2026-10-26
[S3] Volume Feature Rendering for Fast Neural Radiance Field Reconstruction, https://doi.org/10.52202/075280-2853, fetched 2026-10-26
[S4] UVA: Towards Unified Volumetric Avatar for View Synthesis, Pose rendering, Geometry and Texture Editing, https://doi.org/10.48550/arXiv.2304.06969, fetched 2026-10-29 (unofficial)
[S5] PDF MoE-INR: Implicit Neural Representation with Mixture-of-Experts for ..., https://academicweb.nd.edu/~cwang11/papers/vis25-moeinr.pdf, fetched 2026-10-29
[S6] Limitations of NERF with pre-trained Vision Features for Few-Shot 3D Reconstruction, https://arxiv.org/abs/2506.18208v1, fetched 2026-10-26 (unofficial)
