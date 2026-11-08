# Method-Specific Training Configurations


## Overview
This family defines the hyperparameters, optimization routines, and architectural constraints required to train specific neural radiance field variants. It addresses the problem of stabilizing differentiable volume rendering training, which is sensitive to initialization, sampling density, and activation function choices. It sets itself apart from general [Training Configuration](2332-training-configuration.md) by providing implementation-specific details (e.g., activation functions, learning rate schedules, and loss weights) that are critical for convergence in methods like mip-NeRF, DONeRF, and Neuralangelo.

## Shared principles
The training configurations for these methods share the following foundational mechanisms:
1.  **Differentiable Volume Rendering**: All methods minimize the Mean Squared Error (MSE) between rendered images and ground truth by optimizing an MLP that maps 3D coordinates and view directions to density ($\sigma$) and color ($c$) [S1][S3, S4, unofficial]. The rendering equation is:
    $$ C(\mathbf{r}) = \sum_{i=1}^{N} T_i \alpha_i c_i $$
    where $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$ and $T_i = \prod_{j=1}^{i-1} (1 - \alpha_j)$ [S3].
2.  **Adam Optimizer**: All cited methods use the Adam optimizer [S1][S2][S3, S4, unofficial].
3.  **Ray Sampling**: Training involves sampling rays from camera poses and querying the network at discrete points along these rays [S1][S2][S3, S4, unofficial].
4.  **Chunked Processing**: To manage GPU memory, forward passes are often performed in chunks [S4, unofficial].

## Subtopics

### [NeRF Hyperparameters](3065-nerf-hyperparameters.md)
The standard NeRF configuration establishes baseline settings for network architecture, sampling, and optimization. It employs a two-stage hierarchical sampling process with a proposal network and a fine NeRF MLP, optimized for synthetic scenes like the Blender dataset. It uses standard positional encoding and ReLU activations.

### [Mip-NeRF Optimization and Activation Details](3409-mip-nerf-optimization-and-activation-details.md)
This configuration specifies implementation details for mip-NeRF and JaxNeRF, focusing on activation functions, input encoding, and learning rate scheduling. It removes identity concatenation from the MLP input and employs shifted softplus and widened sigmoid activations to handle multiscale data and prevent optimization stagnation. Unlike standard NeRF, it uses Integrated Positional Encoding (IPE) and a specific warmup schedule.

### [DONeRF](4461-donerf.md)
DONeRF replaces standard raymarching with a two-network architecture: a **depth oracle network** and a **shading network**. It solves computational cost by using the oracle to predict optimal sample locations, allowing rendering with only 2–4 samples per ray. It decouples geometry estimation from color rendering and trains the oracle as a classification task on discretized depth values. Unlike acceleration methods using explicit structures, DONeRF remains a compact MLP-based representation.

### NeUDF Training Configuration
NeUDF uses the ADAM optimizer with a learning rate that warms up to $2 \times 10^{-4}$ in the first 5,000 iterations and decreases to $1 \times 10^{-5}$ by the end [S1]. The batch size consists of 512 random rays sampled from 8 randomly selected camera poses per iteration [S1]. Training lasts for 400,000 iterations, taking 9 hours with mask and 11 hours without mask on a single Nvidia 3090 GPU [S1].

### NeRF-SDP Training Configuration
NeRF-SDP uses the Adam optimizer with a learning rate of $1 \times 10^{-3}$ for the Res-UNet and $5 \times 10^{-4}$ for other parts [S2]. The batch size is 500, trained on an Nvidia GeForce RTX 3090 for 250,000 steps [S2]. The first 170,000 steps optimize auxiliary and rendering losses jointly, while the last 80,000 steps optimize only the rendering loss [S2]. Feature dimensions are set to 24 for 2D features and 8 for 3D features [S2].

### Neuralangelo Training Configuration
Neuralangelo samples pixels randomly without tracking statistics to ensure sufficient detail sampling, requiring long training iterations [S3]. The Region of Interest (ROI) is assumed to be inside a unit sphere [S3]. Training runs for 500,000 iterations [S3]. Inactive hash resolutions have their feature vectors set to zero [S3]. The learning rate is $1 \times 10^{-3}$ with a linear warmup of 5,000 iterations, decaying by a factor of 10 at 300k and 400k iterations [S3]. It uses the AdamW optimizer with weight decay of $10^{-2}$ and $w_{iek} = 0.1$ [S3]. The SDF MLP has one layer, and the color MLP has four layers [S3]. Batch size is 1 for the DTU benchmark and 16 for the Tanks and Temples dataset [S3].

## Background
The standard NeRF hyperparameters were introduced in the original NeRF paper [S1][S3, S4, unofficial]. Mip-NeRF modifications were introduced to improve training stability and robustness against catastrophic failure modes [S1]. DONeRF was developed to address the prohibitive computational cost of dense sampling by decoupling geometry and shading [S2]. NeUDF, NeRF-SDP, and Neuralangelo are subsequent variants that adapt these configurations for specific tasks such as unsigned distance fields, generalizable radiance fields, and high-fidelity surface reconstruction [S1][S2][S3].

## Sources
[S1] NeUDF: Leaning Neural Unsigned Distance Fields with Volume Rendering, https://doi.org/10.1109/cvpr52729.2023.00031, fetched 2026-10-26
[S2] NeRF-SDP: Efficient Generalizable Neural Radiance Field with Scene Depth Perception, https://doi.org/10.1145/3595916.3626380, fetched 2026-10-30
[S3] Neuralangelo: High-Fidelity Neural Surface Reconstruction, https://doi.org/10.1109/cvpr52729.2023.00817, fetched 2026-10-26
