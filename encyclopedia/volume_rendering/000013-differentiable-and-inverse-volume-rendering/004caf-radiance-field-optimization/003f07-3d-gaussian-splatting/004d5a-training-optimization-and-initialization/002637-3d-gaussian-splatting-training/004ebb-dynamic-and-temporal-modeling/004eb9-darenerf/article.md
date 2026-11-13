# DaReNeRF

Also called: Direction-aware Representation for Dynamic Scenes

## Overview
DaReNeRF is a method for novel view synthesis of dynamic scenes that addresses the shift variance and direction ambiguity inherent in traditional Discrete Wavelet Transforms (DWT) by employing a direction-aware representation [S1]. It differs from standard plane-based decompositions by capturing scene dynamics from six distinct orientations using an inverse dual-tree complex wavelet transformation (DTCWT), thereby eliminating the checkerboard artifacts and ghosting associated with DWT [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of posed 2D images of a dynamic scene, each associated with a timestamp $t$.
*   **Output:** A continuous 4D radiance field representation capable of rendering novel views at arbitrary poses and timestamps.

**Data Structures:**
The core data structure is the direction-aware representation of the 4D spatiotemporal volume $D$. The volume is decomposed into a sum of three terms, each consisting of the Hadamard product of two 2D plane-based representations and a learned vector [S1].

The mathematical layout is defined as:
$$ D = \sum_{r=1}^{R_1} M_{rXY} \circ M_{rZT} \circ v_{r1} + \sum_{r=1}^{R_2} M_{rXZ} \circ M_{rYT} \circ v_{r2} + \sum_{r=1}^{R_3} M_{rYZ} \circ M_{rXT} \circ v_{r3} $$

Where:
*   $M_{rAB} \in \mathbb{R}^{A \times B}$ are learned 2D plane-based representations. The indices $(A, B)$ are pairs from $\{X, Y, Z, T\}^2$ such that $A \neq B$. Specifically, the planes are $(XY, ZT)$, $(XZ, YT)$, and $(YZ, XT)$ [S1].
*   $v_{ri} \in \mathbb{R}^F$ are learned vectors along the feature axes.
*   $R_1, R_2, R_3$ are the number of low-rank components for each term.
*   $R = R_1 + R_2 + R_3$ is the total number of low-rank components, where $R \ll N$ ($N$ being spatial resolution) [S1].
*   $F$ is the feature size, typically 3 for RGB colors [S1].

**Steps:**
1.  **Direction-Aware Decomposition:** Instead of using standard DWT, the representation is learned via an inverse dual-tree complex wavelet transformation (DTCWT) to capture features from six distinct orientations [S1]. This ensures shift invariance and avoids the checkered pattern blending representations from $\pm 45^\circ$ found in DWT [S1].
2.  **Feature Computation:** For a given space-time point, compute density and appearance features by:
    *   Extracting feature vectors from the paired planes (e.g., $XY$ and $ZT$).
    *   Multiplying these vectors (Hadamard product).
    *   Concatenating the multiplied results from all three terms into a single vector.
    *   Multiplying this concatenated vector by $V$, a 2D tensor stacking all $v_{ri}$ [S1].
3.  **Opacity Query:** Query point opacities directly from the computed density features [S1].
4.  **Color Regression:** Regress RGB color values using a compact Multi-Layer Perceptron (MLP). The inputs to the MLP are the appearance features and the view directions [S1].
5.  **Rendering:** Synthesize final images via differentiable volumetric rendering along camera rays [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_density_and_appearance_features(point_coords, planes, vectors):
    """
    Computes density and appearance features for a 4D point.
    
    Args:
        point_coords: Tuple (x, y, z, t) representing the 4D coordinate.
        planes: Dictionary containing 2D plane tensors.
                Keys: 'XY', 'ZT', 'XZ', 'YT', 'YZ', 'XT'.
                Values: 2D tensors of shape (Res_A, Res_B).
        vectors: List of learned vectors v_r1, v_r2, v_r3.
                 Each vector is of shape (F,).
    
    Returns:
        density_features: Tensor for opacity query.
        appearance_features: Tensor for color MLP input.
    """
    x, y, z, t = point_coords
    
    # Extract values from planes at specific coordinates
    # Note: In implementation, this involves sampling or indexing the 2D planes
    val_xy = planes['XY'][x, y]
    val_zt = planes['ZT'][z, t]
    
    val_xz = planes['XZ'][x, z]
    val_yt = planes['YT'][y, t]
    
    val_yz = planes['YZ'][y, z]
    val_xt = planes['XT'][x, t]
    
    # Term 1: XY and ZT
    term1 = val_xy * val_zt * vectors[0] # Hadamard product with vector v_r1
    
    # Term 2: XZ and YT
    term2 = val_xz * val_yt * vectors[1] # Hadamard product with vector v_r2
    
    # Term 3: YZ and XT
    term3 = val_yz * val_xt * vectors[2] # Hadamard product with vector v_r3
    
    # Concatenate results
    combined_features = concatenate([term1, term2, term3])
    
    # Multiply by V (stacked vectors)
    # V is a 2D tensor of shape (3*F, F) or similar, stacking v_r1, v_r2, v_r3
    # Assuming V is pre-computed from vectors
    features = matmul(combined_features, V)
    
    # Split into density and appearance if necessary, or use directly
    # The source states opacities are queried from density features
    # and color from appearance features. 
    # Assuming a split or specific indexing based on F.
    # For simplicity, we return the full feature vector.
    return features

def render_ray(ray_origin, ray_direction, time, planes, vectors, color_mlp):
    """
    Renders a single ray using volumetric rendering.
    """
    # Sample points along the ray
    points = sample_points_along_ray(ray_origin, ray_direction)
    
    densities = []
    colors = []
    
    for point in points:
        # Get 4D coordinates including time
        point_4d = (point.x, point.y, point.z, time)
        
        # Compute features
        features = compute_density_and_appearance_features(point_4d, planes, vectors)
        
        # Query opacity from density features
        # [G] Assuming a simple mapping or direct usage of a subset of features
        opacity = query_opacity(features)
        
        # Regress color using MLP
        # Input: appearance features and view direction
        color = color_mlp(features, ray_direction)
        
        densities.append(opacity)
        colors.append(color)
        
    # Perform volumetric rendering (e.g., standard alpha compositing)
    image_color = volumetric_render(densities, colors, ray_transmittance)
    
    return image_color
```

**Parameters:**
*   $F$: Feature size. Typical value is 3 (for RGB) [S1].
*   $R$: Total number of low-rank components ($R = R_1 + R_2 + R_3$). Must be much smaller than spatial resolution $N$ ($R \ll N$) [S1].
*   $N$: Spatial resolution [S1].
*   $T$: Temporal resolution [S1].

**Edge Cases and Pitfalls:**
*   **Shift Variance:** Traditional DWT suffers from shift variance, where small shifts in the input signal disrupt wavelet oscillation patterns. This causes poor results in dynamic regions with motion or camera movement. DaReNeRF mitigates this using DTCWT [S1].
*   **Direction Ambiguity:** Traditional 2D DWT has poor direction selectivity, producing a checkered pattern that blends representations from $\pm 45^\circ$. This leads to ghosting artifacts around moving objects. DaReNeRF captures features from six distinct orientations to avoid this [S1].

## Operating characteristics
*   **Memory Complexity:** The method reduces memory complexity from $O(N^3 T F)$ for direct 4D volume modeling to $O(R N^2 T F)$ using the plane-based decomposition [S1].
*   **Performance:** The method achieves state-of-the-art performance in novel view synthesis for dynamic scenes [S1].
*   **Strengths:** Robustness to motion and camera movement due to shift invariance; reduced ghosting artifacts due to direction-aware representation [S1].
*   **Limitations:** The direction-aware representation introduces a 2D redundancy (with $d=2$ for plane-based decomposition) compared to DWT representation, resulting in lower storage efficiency than ideal DWT, though still better than direct 4D modeling [S1].

## Background
DaReNeRF was introduced to address the limitations of implicit representations for dynamic scenes, specifically the shift variance and lack of direction selectivity in Discrete Wavelet Transforms (DWT) [S1]. It builds upon the baseline plane-based 4D volume decomposition [S1] and incorporates concepts from the dual-tree complex wavelet transform (DTCWT) [S1]. The method is designed for novel view synthesis and re-rendering of 3D scenes from 2D images, with implications for AR/VR applications [S1].

## Sources
[S1] DaReNeRF: Direction-aware Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52733.2024.00481, fetched 2026-10-27
