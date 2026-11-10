# SuRFNet

Also called: Sparse Radiance Fields Network

## Overview
SuRFNet is a 3D pipeline that distills multiple input image views into partial 3D Sparse Radiance Fields (SRFs) and learns within the 3D sparse voxel space to generate full SRFs from these partial inputs [S1]. It addresses few-view novel view synthesis by ensuring multi-view consistency, particularly for out-of-distribution views, unlike 2D prior-based methods [S1]. It operates directly on irregular volumetric sparse voxel structures using 3D sparse convolutions, rather than projecting 2D features or relying on dense volumetric grids [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** Partial SRFs $X$, derived from input images [S1].
*   **Output:** Whole SRFs $\hat{X}$, representing the complete scene radiance field [S1].

**Data Structures:**
The method operates on SRFs, which are irregular volumetric representations stored as sparse voxel grids [S1]. Sparse convolutional strides and pruning alter the sparse voxel coordinates during processing [S1].

**Algorithm:**
The network $F$ maps partial SRFs to whole SRFs: $\hat{X} = F(X)$ [S1]. The implementation uses MinkowskiNet as the backbone 3D sparse convolution network [S1].

1.  **Input Processing:** Partial SRFs $X$ are fed into the MinkowskiNet-based architecture [S1].
2.  **Sparse Convolution:** The network processes the sparse voxel coordinates and features through 3D sparse convolutional layers [S1].
3.  **Topology Handling:** Due to sparse convolutional strides and pruning, the output topology (voxel coordinates) may differ from the input topology [S1].
4.  **Loss Calculation:** To handle the topological mismatch and guide training, a sampling function $S$ is used to define the loss [S1]. The loss is calculated using Quantized Gaussian sampling (Q-Gaussian) [S1]. Q-Gaussian is a 3D normal distribution quantized to integer coordinates, centered at the middle of the voxel grid [S1]. This provides a prior for where the output is expected and where the loss is defined [S1].
5.  **Radiance Color Loss:** A specific loss term ensures the output SRFs follow ground truth densities and colors [S1].

**Pitfalls and Edge Cases:**
*   **Memory Constraints:** Training cannot involve densifying voxels to the original resolution due to prohibitive memory requirements [S1].
*   **Topological Mismatch:** The input and output topologies are not necessarily the same because strides and pruning alter sparse voxel coordinates, making the sampling function $S$ critical for correct loss definition [S1].
*   **Irregularity:** SRFs are irregular volumetric representations that do not necessarily reflect the underlying 3D shape/scene, as they result from optimizing posed images into volume [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import random

def surfnet_forward(partial_srf_x, network_f):
    """Converts partial SRFs to whole SRFs using the learned network F."""
    whole_srf_x_hat = network_f(partial_srf_x)
    return whole_srf_x_hat

def quantized_gaussian_sample(mean, std, num_samples):
    """Generates random 3D coordinates from a normal distribution and quantizes to integers."""
    sampled_coords = []
    for _ in range(num_samples):
        x = random.gauss(mean[0], std)
        y = random.gauss(mean[1], std)
        z = random.gauss(mean[2], std)
        # Quantize to integer coordinates
        sampled_coords.append([round(x), round(y), round(z)])
    return sampled_coords

def extract_features(srf, coords):
    """Retrieves feature vectors at specified voxel coordinates from the SRF."""
    features = []
    for coord in coords:
        # Lookup feature at coord in the sparse voxel grid
        features.append(srf.lookup(coord))
    return features

def mean_squared_error(pred, target):
    """Computes the mean squared error between two lists of feature vectors."""
    total_error = 0.0
    count = 0
    for p, t in zip(pred, target):
        for val_p, val_t in zip(p, t):
            total_error += (val_p - val_t) ** 2
            count += 1
    return total_error / count

def calculate_loss(whole_srf_x_hat, ground_truth_srf, voxel_grid_center):
    """Calculates the loss using Quantized Gaussian sampling."""
    # Sample coordinates for loss calculation
    # [G] Standard deviation not specified in sources; 1.0 is a typical initialization
    sampled_coords = quantized_gaussian_sample(voxel_grid_center, std=1.0, num_samples=1000)

    # Extract features at sampled coordinates
    sampled_features = extract_features(whole_srf_x_hat, sampled_coords)
    gt_features = extract_features(ground_truth_srf, sampled_coords)

    # Compute Radiance Color Loss
    loss = mean_squared_error(sampled_features, gt_features)
    return loss
```

## Subtopics

### SparseCNN Network Architecture
A generalizable efficient 3D CNN ($\psi^{3D}$) providing geometric priors for foreground contents, implemented using `torchsparse` [S3]. It takes a global point cloud $P \in \mathbb{R}^{N_p \times 3}$, quantizes it with voxel size $v_{\text{size}} = 0.1\text{m}$, and feeds the resulting sparse tensors into $\psi^{3D}$ to predict a latent feature volume $F$ [S3]. The architecture is U-Net-like with skip connections, consisting of sparse convolution, batch normalization, and ReLU layers [S3]. It is used to provide geometric priors, distinct from the SRF generation focus of the main SuRFNet pipeline [S3].

**Layer Configuration:**
*   **Convi=0:** Kernel $3\times3\times3$, Stride 1, In/Out Ch. 3/16 [S3]
*   **Convi=1:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 16/16 [S3]
*   **Convi=2:** Kernel $3\times3\times3$, Stride 1, In/Out Ch. 16/16 [S3]
*   **Convi=3:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 16/32 [S3]
*   **Convi=4:** Kernel $3\times3\times3$, Stride 1, In/Out Ch. 32/32 [S3]
*   **Convi=5:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 32/64 [S3]
*   **Convi=6:** Kernel $3\times3\times3$, Stride 1, In/Out Ch. 64/64 [S3]
*   **DeConvi=7:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 64/32 [S3]
*   **DeConvi=8:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 32/16 [S3]
*   **DeConvi=9:** Kernel $3\times3\times3$, Stride 2, In/Out Ch. 16/16 [S3]

## Operating characteristics
*   **Memory:** Training avoids densifying voxels to original resolution due to prohibitive memory requirements [S1].
*   **Consistency:** Ensures multi-view consistency, especially for out-of-distribution views [S1].
*   **Complexity:** Not given in the sources.
*   **Strengths:** 3D learning setup ensures multi-view consistency; handles sparse irregular volumetric data efficiently [S1].
*   **Limitations:** Input/output topologies may differ, requiring careful sampling for loss calculation; SRFs are irregular and may not reflect true 3D shape [S1].

## Background
SuRFNet was proposed in the paper "SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images" [S1]. It extends previous few-view novel view synthesis methods (like PixelNeRF and VisionNeRF) which learned 2D priors, by instead learning in 3D sparse voxel space [S1]. It leverages MinkowskiNet for 3D sparse convolutions [S1].

## Sources
[S1] SPARF: Large-Scale Learning of 3D Sparse Radiance Fields from Few Input Images, https://doi.org/10.1109/iccvw60793.2023.00315, fetched 2026-10-26
[S2] PDF Deep Learning Methods and Neural Representations for Scientific Volume ..., https://www.cs.uoregon.edu/Reports/AREA-202603-Mathai.pdf, fetched 2026-10-27 (unofficial)
[S3] EVolSplat: Efficient Volume-based Gaussian Splatting for Urban View Synthesis, https://doi.org/10.1109/CVPR52734.2025.01054, fetched 2026-11-03
