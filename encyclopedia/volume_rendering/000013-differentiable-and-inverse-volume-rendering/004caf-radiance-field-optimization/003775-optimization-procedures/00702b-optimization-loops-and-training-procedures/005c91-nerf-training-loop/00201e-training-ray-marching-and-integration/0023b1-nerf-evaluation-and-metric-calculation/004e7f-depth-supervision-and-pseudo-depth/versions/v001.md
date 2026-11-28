# Depth Supervision and Pseudo-Depth

Also called: Depth-Aware Optimization, Pseudo-Surface Sampling

## Overview
This family of techniques incorporates geometric constraints into the radiance field optimization loop to improve reconstruction quality, particularly in low-data regimes or when explicit geometry is required for downstream tasks. While standard NeRF optimization relies solely on photometric RGB losses, these methods introduce depth-related penalties or sampling strategies. Depth Supervision explicitly minimizes the difference between rendered depth and ground-truth depth (derived from SfM keypoints or stereo), whereas Pseudo-Depth approximates surface geometry via weighted averaging of sampled points to guide feature distillation or segmentation without requiring explicit depth labels.

## Shared principles
All methods in this family operate within the differentiable volume rendering framework. The core mechanism involves computing a depth estimate $\hat{D}(r)$ for a ray $r$ during the forward pass of the renderer and incorporating this value into the loss function or sampling strategy.

The predicted depth $\hat{D}(r)$ is typically computed as the expected depth along the ray, weighted by the transmittance and density at each sample point $t_i$:

$$
\hat{D}(r) = \sum_{i=1}^{N} w_i t_i
$$

where $w_i$ is the weight assigned to sample $i$, defined as:

$$
w_i = T_i (1 - \exp(-\sigma_i \delta_i))
$$

Here, $\sigma_i$ is the volume density, $\delta_i = t_{i+1} - t_i$ is the interval between adjacent samples, and $T_i = \exp\left(-\sum_{j=1}^{i-1} \sigma_j \delta_j\right)$ is the accumulated transmittance up to sample $i$. This formulation ensures that depth contributions are proportional to the opacity of the surface encountered at that depth.

## Subtopics

### Depth Supervision for TensoRF
This variant applies explicit depth supervision to the TensoRF architecture to enhance performance in low-data scenarios. It differs from standard NeRF depth supervision by leveraging TensoRF's tensor decomposition for efficiency, while using COLMAP keypoints to generate sparse ground-truth depth labels.

**Definition**
TensoRF is augmented with a depth supervision term derived from COLMAP keypoints to improve low-data performance [S1, unofficial].

**Procedure**
1.  Extract 3D point locations and their corresponding 2D pixel coordinates from the COLMAP keypoints stored in the dataset [S1, unofficial].
2.  Using the coordinate information and the camera matrix for the ground truth image, calculate the ray $r(t) = o + td$ and the ground-truth depth $D$ from the projection origin [S1, unofficial].
3.  Sample the TensoRF model along these rays to calculate the model-estimated depths $\hat{D}$ [S1, unofficial].

**Loss Function**
The depth loss is calculated using Mean Squared Error (MSE):

$$
L_{\hat{D}} = \mathbb{E}_{r \in R(P)} ||\hat{D}(r) - D(r)||_2^2
$$

where $\hat{D}(r)$ is the predicted depth and $D(r)$ is the ground-truth depth [S1, unofficial]. The total loss combines this with the color loss $L_C$ and a weighting hyperparameter $\lambda_i$:

$$
L_{tot} = L_C + \lambda_i L_{\hat{D}}
$$

**Parameters and Context**
*   **Dataset:** LLFF dataset, consisting of 20-30 images of household/office environments captured with a smartphone in a grid pattern [S1, unofficial].
*   **Rationale:** The LLFF dataset was chosen due to the small variance of angles, making it more suitable for this approach than 360-degree view datasets [S1, unofficial].
*   **Loss Choice:** MSE loss was implemented as a baseline. Although the DS-NeRF paper reported slightly better results with KL loss, the improvements were of similar magnitude [S1, unofficial].
*   **Architecture:** TensoRF was selected over standard NeRF due to faster training times and lower space complexity, fitting limited GPU and RAM resources [S1, unofficial].

### Pseudo Depth Computation
This technique approximates surface geometry without explicit depth labels. It is primarily used in feature field distillation (e.g., for CLIPNeRF) to enable efficient sampling around the "surface" for feature extraction.

**Procedure**
1.  Compute the pseudo depth of the surface as a weighted average of the depth of each sampled point during volume rendering [S2].
2.  Use this pseudo-surface point to compute features by sampling a small neighborhood (e.g., five points) around it with small perturbations during training and inference [S2].

**Context**
This method is employed for computational efficiency in feature rendering, avoiding the complete process of volume rendering for features. It allows gradients to be backpropagated only to rays selected by specific semantic masks (e.g., "apple") during optimization [S2].

### Panoramic Depth Extraction via Sphere Projection
This method generates ground-truth depth for novel panoramic views by projecting 2D pixel coordinates onto a uniform sphere centered at the camera position. It enables the extraction of depth priors for 360-degree scenes.

**Definition**
A method to extract depth ground truths for novel panoramic views by projecting 2D pixel coordinates to a uniform sphere centered at the camera position [S3].

**Procedure**
1.  For a pixel $(x, y)$ on a panorama of height $H$ and width $W$, calculate the vertical viewing angle $\theta = \pi y / H$ and horizontal viewing angle $\phi = 2\pi x / W$ [S3].
2.  Define the ray origin as the camera position (coordinate center) and the ray direction as the unit vector from the center to the sphere point defined by $\theta$ and $\phi$ [S3].
3.  Determine visibility for a novel view by moving the camera to a new position and checking which parts of the ground truth sphere are visible to the emitted rays from the new origin [S3].

**Optimization**
The network parameters $\theta$ are optimized using RGB-D frames. The geometric loss is the absolute difference between predicted and true depths, normalized by the depth variance [S3].

## Background
Depth supervision in NeRF-like models originated from the observation that photometric loss alone often leads to ambiguous geometry, especially in textureless regions or with sparse inputs. The DS-NeRF paper introduced depth supervision using KL divergence to align predicted depth distributions with ground truth [S1, unofficial]. TensoRF-DS adapted this to tensor-decomposed fields for efficiency [S1, unofficial]. Pseudo-depth techniques emerged in the context of feature distillation, where explicit depth is unavailable but surface proximity is needed for feature sampling [S2]. Panoramic depth extraction extends these concepts to 360-degree scenes using spherical projections [S3].

## Sources
[S1] Low-Cost Neural Radiance Fields, https://doi.org/10.48550/arxiv.2605.09312, fetched 2026-10-26 (unofficial)
[S2] Decomposing NeRF for Editing Via Feature Field Distillation, https://doi.org/10.52202/068431-1694, fetched 2026-10-26
[S3] 360FusionNeRF: Panoramic Neural Radiance Fields with Joint Guidance, https://doi.org/10.1109/iros55552.2023.10341346, fetched 2026-10-26
