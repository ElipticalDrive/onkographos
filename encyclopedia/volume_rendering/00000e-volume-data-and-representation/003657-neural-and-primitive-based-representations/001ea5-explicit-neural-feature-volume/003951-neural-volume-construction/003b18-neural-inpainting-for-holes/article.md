# Neural Inpainting for Holes

Also called: Inpaint Network for Holes

## Overview
Neural Inpainting for Holes is a method to resolve the "one-to-many" geometric ambiguity that arises during forward warping of voxel grids in dynamic volume rendering [S1]. When a scene expands, multiple source voxels map to a single target voxel, leaving empty "holes" in the target grid if warping is performed naively [S1]. This method employs a neural network to predict and fill these missing voxel values, enabling the reconstruction of a complete, high-resolution voxel grid from a lower-resolution warped input [S1]. It is distinct from sibling methods like Sparse Domain Decomposition and TensorBase by specifically addressing temporal warping artifacts through a dedicated 3D convolutional architecture rather than static representation or spatial partitioning [S1].

## How it works
**Inputs and Outputs**
*   **Input:** A warped voxel grid $V^{Rt}$ containing holes (empty or undefined positions) resulting from forward warping of a source grid at time $t-1$ to time $t$ [S1].
*   **Output:** An inpainted and up-sampled voxel grid $V^{Rt}_{Inp}$ with filled values and potentially higher resolution [S1].

**Data Structures**
The input is a 3D voxel grid. The internal representation within the network is a tensor of shape corresponding to the grid dimensions. The output is a 3D voxel grid of the same or higher resolution as the input, with all previously empty positions populated [S1].

**Algorithm**
The method consists of two main stages: warping to a low resolution (to save compute) and neural inpainting/up-sampling [S1].

1.  **Forward Warping (Pre-processing):** The source voxel grid is warped to the target coordinate system using forward warping. Due to the one-to-many mapping issue (where one source position maps to multiple target positions), this step leaves holes in the target grid [S1].
2.  **Network Input:** The warped voxel grid $V^{Rt}$ is fed into the inpaint network. The network does not receive temporal information (e.g., previous frames or flow vectors) [S1].
3.  **Neural Inpainting:** The network processes $V^{Rt}$ to predict values for the empty voxels based on local spatial context [S1].
4.  **Up-sampling:** Simultaneously with inpainting, the network up-samples the grid to the final rendering resolution [S1].

**Network Architecture**
The inpaint network $F_{\theta3}$ consists of a 3D U-Net structure followed by an up-sampling layer [S1].
*   **3D U-Net:** Uses encoding layers to learn information from local neighboring voxels and decoding layers to recover resolution [S1].
*   **Up-sampling Layer:** Consists of an interpolation layer, convolution layers, and activation layers [S1].

**Formula**
The output is computed as:
$$ V^{Rt}_{Inp} = F_{\theta3} (V^{Rt}) $$
Where:
*   $V^{Rt}_{Inp}$ is the inpainted and up-sampled voxel grid [S1].
*   $V^{Rt}$ is the input warped voxel grid with holes [S1].
*   $F_{\theta3}$ is the neural network function with parameters $\theta3$ [S1].

**Parameters**
*   **Temporal Information:** The network has no access to temporal information [S1].
*   **Resolution:** The warping is performed at a low resolution to save time and memory, while the output is at a higher resolution for better image quality [S1]. Specific resolution factors are not given in the sources.

**Pitfalls**
*   **One-to-many issue:** If a circle expands, multiple positions in the target correspond to the same position in the source. Naive warping leaves holes in the target grid [S1].
*   **Lack of Temporal Context:** Because the network only sees the warped grid at time $t$, it cannot learn to warp; it is limited strictly to inpainting missing values based on spatial context [S1].
*   **Computational Cost:** Warping operations and the inpaint network are relatively expensive to compute [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def neural_inpainting_for_holes(warped_grid: Tensor3D) -> Tensor3D:
    """
    Fills holes in a warped voxel grid using a 3D U-Net and up-samples it.
    
    Args:
        warped_grid: A 3D tensor representing the voxel grid at time t,
                     containing holes (e.g., NaN or 0) where forward warping
                     failed due to one-to-many mapping.
                     
    Returns:
        inpainted_grid: A 3D tensor with holes filled and resolution up-sampled.
    """
    # The inpaint network F_theta3 consists of a 3D U-Net followed by an up-sampling layer.
    # The up-sampling layer includes interpolation, convolution, and activation layers.
    # The network has no access to temporal information.
    
    # 1. Pass the warped grid through the 3D U-Net encoder to extract local features.
    # 2. Decode features to recover spatial resolution and fill missing values.
    # 3. Apply the up-sampling layer (interpolation + conv + activation) to reach target resolution.
    
    # Note: The specific architecture details (kernel sizes, number of layers) are not provided.
    # This is a conceptual representation of the function F_theta3.
    
    inpainted_grid = apply_3d_unet_and_upsample(warped_grid)
    
    return inpainted_grid

def apply_3d_unet_and_upsample(grid: Tensor3D) -> Tensor3D:
    """
    Placeholder for the specific implementation of F_theta3.
    In the source, this is a 3D U-Net structure followed by an up-sampling layer.
    """
    # Implementation details of the 3D U-Net and up-sampling layer are not provided in the sources.
    # [G] Standard 3D U-Net implementation would involve:
    # - Encoder path: Conv3D -> BatchNorm -> ReLU -> MaxPool3D
    # - Decoder path: ConvTranspose3D -> Concat -> Conv3D -> BatchNorm -> ReLU
    # - Up-sampling layer: Interpolation (e.g., Trilinear) -> Conv3D -> Activation
    raise NotImplementedError("Specific architecture parameters not provided in sources.")
```

## Subtopics

### Hole Filling
A process to fill holes (pixels with zero values) in projected depth images that correspond to missing information on the surface of a foreground object, while maintaining background areas [S2]. This method is used in the context of depth estimation for NeRFs, distinct from the 3D voxel inpainting in the main method [S2]. It distinguishes itself by using a statistical threshold on neighboring pixel values to decide whether a hole is surface information or background [S2].

**Definition:** Fill holes in projected depth images by distinguishing between background areas and missing surface information [S2].

**Steps:**
1.  For a hole pixel $p$, calculate the mean $\mu$ and standard deviation $\sigma$ from an $M \times M$ neighborhood of pixels centered at $p$ [S2].
2.  Check if the condition $\frac{p - \mu}{\sigma} > \kappa$ is satisfied [S2].
3.  If satisfied, the hole is determined to be missing surface information. Fill it by applying a moving average filter with kernel size $M \times M$ [S2].

**Formula:**
$$ \frac{p - \mu}{\sigma} > \kappa $$
Where:
*   $p$ is the pixel value (hole pixel) [S2].
*   $\mu$ is the average of the $M \times M$ neighboring pixels [S2].
*   $\sigma$ is the standard deviation of the $M \times M$ neighboring pixels [S2].
*   $\kappa$ is a specified threshold [S2].

**Parameters:**
*   $\kappa$: Threshold for hole filling. Empirically found best value: 2 [S2].
*   $M$: Kernel size for the neighborhood and moving average filter. Not given in the sources.

**Pitfalls:**
*   If $\kappa$ is too large, holes with missing surface information may remain unfilled [S2].
*   If $\kappa$ is too small, holes in background areas may be filled, blurring the depth image [S2].

**Performance:**
On the DONeRF Lego dataset ($N=8, \alpha=1/16$), rendering PSNR results for different $\kappa$ values are:
*   $\kappa=0.25$: 30.16 dB
*   $\kappa=0.5$: 30.24 dB
*   $\kappa=1$: 30.25 dB
*   $\kappa=2$: 30.25 dB (Best)
*   $\kappa=3$: 30.20 dB
*   $\kappa=4$: 30.16 dB
*   Without hole filling: 27.29 dB [S2].

## Operating characteristics
**Time and Memory Complexity:**
The warping operations and the inpaint network are relatively expensive to compute [S1]. To mitigate this, the method proposes warping grids at a low resolution and up-sampling afterwards, which saves time and memory compared to high-resolution warping [S1]. Specific complexity formulas are not given in the sources.

**Measured Performance:**
No specific performance metrics (FPS, memory usage) for the Neural Inpainting for Holes method itself are provided in the sources [S1]. The Hole Filling variant shows PSNR improvements from 27.29 dB to 30.25 dB on the DONeRF Lego dataset [S2].

**Scaling:**
The method scales by allowing low-resolution warping followed by neural up-sampling, which reduces the computational load of the warping step while maintaining high-resolution output quality [S1].

**Strengths and Limitations:**
*   **Strengths:** Effectively resolves the one-to-many warping issue that leaves holes in target grids [S1]. Allows for efficient computation by operating on low-resolution grids [S1].
*   **Limitations:** The network lacks temporal information, limiting its ability to only inpaint rather than predict motion [S1]. The computational cost of the network and warping is high [S1].

**When to Use:**
Use when forward warping of voxel grids results in holes due to expansion (one-to-many mapping) and a complete, high-resolution voxel grid is required for volume rendering [S1].

**When Not to Use:**
Do not use if temporal consistency is critical and the network cannot access temporal information [S1]. Do not use if computational resources are extremely limited, as the method is expensive [S1].

## Background
**Origin:**
The method was proposed in the paper "Forward Flow for Novel View Synthesis of Dynamic Scenes" [S1].

**History and Development:**
The method addresses the limitations of naive forward warping in dynamic scene synthesis, specifically the "one-to-many" issue where expanding objects leave holes in the target grid [S1]. It builds on the concept of average splatting for the "many-to-one" issue [S1].

**Where it is used:**
Used in novel view synthesis of dynamic scenes, specifically within the volume rendering pipeline for dynamic NeRFs [S1].

## Sources
[S1] Forward Flow for Novel View Synthesis of Dynamic Scenes, https://doi.org/10.1109/iccv51070.2023.01468, fetched 2026-10-26
[S2] Improving Neural Radiance Fields Using Near-Surface Sampling with Point Cloud Generation, https://link.springer.com/article/10.1007/s11063-024-11654-5, fetched 2026-10-30
