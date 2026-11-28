# Synthetic Motion Blur Simulation

Also called: Stochastic Motion Blur for Training

## Overview
Synthetic Motion Blur Simulation generates blurry training images from sharp RGB-D sequences by stochastically applying a motion blur simulator with controlled size and intensity parameters [S1]. It serves as a data augmentation strategy to prevent hybrid neural rendering models from overfitting to sharp rendered images when trained against blurry real-world reference images [S1]. Unlike [Motion Blur via Sub-frame Optimization](11034-motion-blur-via-sub-frame-optimization.md), which optimizes sub-frame representations, this method applies explicit convolutional blur kernels to rendered patches during the training phase only [S1].

## How it works
The method operates in two distinct modes: stochastic augmentation for general training data and kernel-based optimization for specific patch losses.

### Inputs and Outputs
*   **Inputs:** Sharp RGB image (or rendered image patch) $C$, reference image patch $C_{gt}$.
*   **Outputs:** Blurry training image (or optimized rendered patch).

### Data Structures
*   **Blur Kernel $B_i$:** A 2D convolution kernel representing motion in a specific direction and distance.
*   **Parameters:**
    *   $k$: Size parameter (moving distance).
    *   $\phi$: Intensity parameter (shake level).
    *   $N_v$: Number of blur directions.
    *   $N_d$: Number of moving distances.

### Steps

#### Mode 1: Stochastic Augmentation
This mode is used to create blurry training images from sharp sequences [S1].

1.  For each sharp RGB training image, sample a random value.
2.  If the random value falls within the 0.75 probability threshold, proceed to step 3; otherwise, skip to step 5 [S1].
3.  Sample the size parameter $k$ uniformly from the range $(3, 16)$ [S1].
4.  Sample the intensity parameter $\phi$ uniformly from the range $(0, 0.1)$ [S1].
5.  Apply the motion blur simulator to the sharp RGB image using the sampled $k$ and $\phi$ to produce a blurry training image [S1].

#### Mode 2: Kernel-Based Patch Optimization
This mode is used to preserve sharpness during training by comparing blurred rendered patches with reference patches [S1].

1.  Create a set of blur kernels $B_i$ for $i = 0$ to $N_v \times N_d$, considering $N_v$ directions and $N_d$ distances [S1].
    *   $i=0$ represents no blur simulation [S1].
2.  Sample $8 \times 8$ patches with dilations from the rendered result $C$ [S1].
3.  Apply all blur kernels to the rendered result to obtain blurred image patches $\hat{C}_i$:
    $$ \hat{C}_i = \text{Conv}(C, B_i) $$
    where $\text{Conv}$ denotes the convolution operation [S1].
4.  Calculate the photo-metric loss $L_i$ for each kernel:
    $$ L_i = \|\hat{C}_i - C_{gt}\|_2 $$
    where $C_{gt}$ is the reference image patch [S1].
5.  Select the kernel $i$ that yields the minimum loss:
    $$ L_{color} = \min \{ L_i \mid i = 0, \dots, N_v \times N_d \} $$
    The rendered patch is optimized based on this minimum loss [S1].

### Parameters
*   **Blur Probability:** 0.75 per image [S1].
*   **Size Parameter $k$:** Controls moving distance, uniformly sampled from $k \in (3, 16)$ [S1].
*   **Intensity Parameter $\phi$:** Determines shake level in the moving direction, uniformly sampled from $\phi \in (0, 0.1)$ [S1].
*   **Number of Directions $N_v$:** Typical value is 12 (4 symmetrical: 'left-right', 'up-down', 'top left-bottom down', 'bottom left-top right'; and 8 asymmetrical) [S1].
*   **Number of Distances $N_d$:** Typical value is 3 (distances 1, 2, 4) [S1].
*   **Patch Size:** $8 \times 8$ patches with dilations [S1].

### Edge Cases and Pitfalls
*   **Inference:** The blur simulation process is removed during inference to produce sharp images $C$ [S1].
*   **Content Awareness:** The method assumes camera moves in one direction at a given moment while capturing high frame rate videos [S1].
*   **Performance:** This method runs faster and is more memory efficient than methods requiring extra ray rendering or exhaustive per-view embedding optimization (e.g., Deblur-NeRF) [S1].

## Subtopics

### Synthetic Motion Blur Simulation for Training
A specific application of the method where blur kernels are explicitly constructed and applied to rendered image patches to preserve sharpness during training by comparing blurred rendered patches with reference patches [S1]. It differs from the stochastic augmentation mode by using a discrete set of predefined kernels ($N_v \times N_d$) and selecting the best fit via minimum photo-metric loss [S1]. It is chosen when training on datasets with blurry reference images (e.g., ScanNet) to prevent the neural renderer from learning blur artifacts [S1]. Typical parameters are $N_v=12$ and $N_d=3$ [S1].

## Operating characteristics
*   **Time Complexity:** Faster than methods requiring extra ray rendering or exhaustive per-view embedding optimization [S1].
*   **Memory Complexity:** More memory efficient than Deblur-NeRF [S1].
*   **Scalability:** Suitable for large-scale scenes with dense views [S1].
*   **Strengths:** Simple to implement, computationally efficient, effective at handling blurry reference data [S1].
*   **Limitations:** Requires careful selection of kernel directions and distances; stochastic mode relies on random sampling which may not cover all blur types [S1].
*   **When to Use:** When training neural rendering models on datasets with blurry reference images (e.g., ScanNet) and sharp synthetic data [S1].
*   **When Not to Use:** During inference, as the blur simulation is removed to produce sharp images [S1].

## Background
The method was proposed in "Hybrid Neural Rendering for Large-Scale Scenes with Motion Blur" [S1]. It addresses the negative influence of blurry reference images on optimizing hybrid neural rendering models [S1]. The approach is complementary to content-aware blur detection, which down-weights blurry images based on blurriness scores [S1]. The method was tested on ScanNet [6] and synthetic data generated from Habitat-sim [24] [S1].

## Sources
[S1] Hybrid Neural Rendering for Large-Scale Scenes with Motion Blur, http://arxiv.org/abs/2304.12652v1, fetched 2026-10-27
