# DaReNeRF

Also called: DaReNeRF architecture, Direction-Aware Representation, Direction-aware Representation for Dynamic Scenes

## Overview
DaReNeRF is a method for novel view synthesis of dynamic scenes that employs a direction-aware representation to capture spatiotemporal dynamics from six distinct orientations [S1]. It addresses the shift variance and direction ambiguity (checkerboard artifacts) inherent in traditional 2D Discrete Wavelet Transforms (DWT) by utilizing the Dual-Tree Complex Wavelet Transform (DTCWT) [S1]. Unlike standard plane-based decompositions, DaReNeRF models features from six different directions, eliminating ghosting artifacts around moving objects while maintaining memory efficiency through low-rank plane-based decomposition [S1].

## How it works
**Inputs:** A set of posed 2D images with timestamps.
**Outputs:** Rendered RGB images for arbitrary camera poses and time steps.

**Data Structures:**
1.  **4D Volume Representation ($D$):** Decomposed into three terms, each consisting of the Hadamard product of two 2D plane-based representations and a learned vector [S1].
2.  **Plane-based Representation ($M^{AB}_r$):** A 2D tensor of size $m \times n$ (where $m, n$ are spatial/temporal resolutions) reconstructed via Inverse Dual-Tree Complex Wavelet Transform (IDTCWT) [S1].
3.  **Wavelet Coefficients:** For each plane $M^{AB}_r$, the representation is defined by:
    *   One approximation coefficient: $W^{AB}_{a,r} \in \mathbb{R}^{m/2 \times n/2}$ [S1].
    *   Six real direction-aware wavelet coefficients: $\{R\{W^{AB}_{i,r}\}\}_{i=1}^6 \in \mathbb{R}^{m/2 \times n/2}$ [S1].
    *   Six imaginary direction-aware wavelet coefficients: $\{I\{W^{AB}_{i,r}\}\}_{i=1}^6 \in \mathbb{R}^{m/2 \times n/2}$ [S1].
    *   *Note:* The coefficients are masked to enforce sparsity (see Pitfalls).
4.  **Learned Vectors ($v_{ri}$):** Vectors $v_{ri} \in \mathbb{R}^F$ where $F=3$ (RGB feature size) [S1].
5.  **Tensor $V_{RF}$:** A 2D tensor stacking all $v_{ri}$ [S1].
6.  **Emptiness Voxel:** A small 3D voxel grid indicating empty regions by aggregating maximum opacities across time steps [S1].

**Parameters:**
*   $R_1, R_2, R_3$: Number of low-rank components for each decomposition term. Total $R = R_1 + R_2 + R_3 \ll N$ (spatial resolution) [S1].
*   Spatial grid size: 512 [S1].
*   Temporal grid size: 300 [S1].
*   Batch size: 4,096 [S1].
*   Coordinate system: Normalized Device Coordinates (NDC) [S1].

**Formulas:**
The 4D volume $D$ is represented as the sum of three components [S1]:
$$ D = \sum_{r=1}^{R_1} M_{rXY} \circ M_{rZT} \circ v_{r1} + \sum_{r=1}^{R_2} M_{rXZ} \circ M_{rYT} \circ v_{r2} + \sum_{r=1}^{R_3} M_{rYZ} \circ M_{rXT} \circ v_{r3} $$
where $M_{rAB} \in \mathbb{R}^{A \times B}$ are learned 2D plane-based representations with $(A, B) \in \{X, Y, Z, T\}^2 | A \neq B$, and $v_{ri} \in \mathbb{R}^F$ are learned vectors [S1].

The plane-based representation is reconstructed using IDTCWT [S1]:
$$ M^{AB}_r = \text{IDTCWT}([W^{AB}_{a,r}, \{R\{W^{AB}_{i,r}\}\}_{i=1}^6, \{I\{W^{AB}_{i,r}\}\}_{i=1}^6]) $$
where IDTCWT is the inverse dual-tree complex wavelet transform [S1].

Masked wavelet coefficients are computed to encourage sparsity [S1]:
$$ \tilde{W}^{AB} = \text{sg}(H(M^{AB}) - \text{sigmoid}(M^{AB})) \odot W^{AB} $$
where $\text{sg}$ is the stop-gradient operator, $H$ is the Heaviside step function, and $M^{AB}$ are learned masks [S1].

The total loss function is [S1]:
$$ L = \frac{1}{|R|} \sum_{r \in R} ||C(r) - \hat{C}(r)||^2 + \lambda_{reg} L_{reg} + \lambda_m L_m $$
where $C(r)$ is rendered color, $\hat{C}(r)$ is ground-truth, $L_{reg}$ is total variation loss, and $L_m$ is mask sparsity loss [S1].

**Steps:**
1.  **Decomposition:** Decompose the 4D spatiotemporal volume $D$ into the sum of three terms defined by Equation 1 above [S1].
2.  **Feature Extraction:** For a given space-time point, extract feature vectors from paired planes (e.g., XY and ZT) [S1].
3.  **Fusion:** Multiply the feature vectors from the paired planes element-wise [S1].
4.  **Concatenation:** Concatenate the multiplied results into a single vector [S1].
5.  **Density/Appearance Computation:** Multiply the concatenated vector by $V_{RF}$ (the tensor stacking all $v_{ri}$) to compute density and appearance features [S1].
6.  **Opacity Query:** Query point opacities directly from the computed density features [S1].
7.  **Color Regression:** Regress RGB color values using a compact Multi-Layer Perceptron (MLP). Inputs to the MLP are the appearance features and view directions [S1].
8.  **Rendering:** Synthesize final images via differentiable volumetric rendering along rays [S1].
9.  **Optimization:** Minimize the total loss $L$ using a coarse-to-fine training strategy, progressively increasing grid resolution [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(point_coords, view_direction, time_step):
    # point_coords: (x, y, z) in NDC
    # time_step: t
    
    # 1. Identify relevant planes based on coordinate pairs
    # Pairs: (XY, ZT), (XZ, YT), (YZ, XT)
    
    features = []
    
    for r in range(R1):
        # Extract features from XY and ZT planes
        feat_xy = extract_plane_features(M_XY[r], point_coords.x, point_coords.y)
        feat_zt = extract_plane_features(M_ZT[r], point_coords.z, time_step)
        
        # Element-wise multiplication
        fused = feat_xy * feat_zt
        features.append(fused)
        
    for r in range(R2):
        # Extract features from XZ and YT planes
        feat_xz = extract_plane_features(M_XZ[r], point_coords.x, point_coords.z)
        feat_yt = extract_plane_features(M_YT[r], point_coords.y, time_step)
        
        fused = feat_xz * feat_yt
        features.append(fused)
        
    for r in range(R3):
        # Extract features from YZ and XT planes
        feat_yz = extract_plane_features(M_YZ[r], point_coords.y, point_coords.z)
        feat_xt = extract_plane_features(M_XT[r], point_coords.x, time_step)
        
        fused = feat_yz * feat_xt
        features.append(fused)
        
    # Concatenate all fused features
    concatenated_features = concatenate(features)
    
    # Multiply by V_RF tensor
    density_appearance = matrix_multiply(concatenated_features, V_RF)
    
    # Query opacity from density features
    opacity = query_opacity(density_appearance)
    
    # Regress color using MLP
    color = color_mlp(appearance_features=density_appearance, view_dir=view_direction)
    
    return opacity, color

def extract_plane_features(plane_tensor, coord1, coord2):
    # Bilinear interpolation or similar sampling from the 2D plane tensor
    # plane_tensor is reconstructed from wavelet coefficients via IDTCWT
    return sample(plane_tensor, coord1, coord2)
```

**Edge Cases and Pitfalls:**
*   **Shift Variance:** Traditional 2D DWT suffers from shift variance, where small shifts in input signal disrupt wavelet oscillation patterns, causing poor results in dynamic regions with motion or camera movement [S1]. DaReNeRF uses DTCWT to ensure shift invariance [S1].
*   **Direction Ambiguity:** Traditional 2D DWT has poor direction selectivity, producing a checkerboard pattern that blends representations from $\pm 45^\circ$, leading to ghosting artifacts around moving objects [S1]. DaReNeRF uses DTCWT with 6 real and 6 imaginary coefficients aligned to specific directions to eliminate this [S1].
*   **Empty Regions:** Large empty regions in dynamic scenes waste computation. DaReNeRF uses a small 3D voxel representation that indicates emptiness by aggregating maximum opacities across time steps to skip these points [S1].
*   **Sparsity:** Learned masks are used to zero out unnecessary wavelet coefficients, controlled by the mask loss $L_m$ [S1].

## Operating characteristics
*   **Memory Complexity:** Reduced from $O(N^3 T F)$ for direct 4D volume modeling to $O(R N^2 T F)$ using the plane-based decomposition, where $R \ll N$ [S1].
*   **Performance:** Achieves state-of-the-art performance on Plenoptic Video and D-NeRF datasets [S1].
*   **Hardware:** Trained on a single A100 GPU [S1].
*   **Strengths:** Robust handling of complex dynamic scenes, elimination of checkerboard artifacts, shift invariance, and memory efficiency [S1].
*   **Limitations:** The direction-aware representation introduces a 2D redundancy compared to DWT, resulting in lower storage efficiency than pure DWT-based methods, though still better than direct 4D modeling [S1].
*   **When to use:** Novel view synthesis of dynamic scenes where high fidelity and motion robustness are required [S1].
*   **When not to use:** Static scenes may not benefit from the temporal complexity, though it is applicable [S1].

## Background
DaReNeRF was introduced to address the limitations of traditional 2D Discrete Wavelet Transforms (DWT) in modeling dynamic scenes, specifically shift variance and direction ambiguity [S1]. It builds upon plane-based 4D volume decomposition [S1] and draws inspiration from the Dual-Tree Complex Wavelet Transform (DTCWT) [S1]. The method was developed to improve upon prior implicit representations and DWT-based approaches that failed to adequately model dynamic scenes, resulting in ghosting artifacts [S1]. It is used in DaReNeRF and its sparse variant, DaReNeRF-S, for novel view synthesis [S1].

## Sources
[S1] DaReNeRF: Direction-aware Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52733.2024.00481, fetched 2026-10-27
