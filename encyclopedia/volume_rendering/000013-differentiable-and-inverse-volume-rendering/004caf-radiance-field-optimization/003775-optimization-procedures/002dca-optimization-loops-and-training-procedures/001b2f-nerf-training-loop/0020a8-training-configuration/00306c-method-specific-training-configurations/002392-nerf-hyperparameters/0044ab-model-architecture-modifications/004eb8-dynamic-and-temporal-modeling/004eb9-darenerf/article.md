# DaReNeRF

Also called: Direction-aware Representation for Dynamic Scenes

## Overview
DaReNeRF is a method for novel view synthesis of dynamic scenes that addresses the shift variance and direction ambiguity inherent in traditional Discrete Wavelet Transforms (DWT) by employing a direction-aware representation [S1]. It differs from standard plane-based NeRF variants by decomposing the 4D spatiotemporal volume using components derived from the Dual-Tree Complex Wavelet Transform (DTCWT) to capture scene dynamics from six distinct orientations, thereby eliminating the checkerboard artifacts and ghosting associated with DWT [S1].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A set of posed 2D images of a dynamic scene, each associated with a specific timestamp.
*   **Outputs:** Rendered images at arbitrary camera poses and timestamps.

**Data Structures:**
The core data structure is the direction-aware representation of the 4D spatiotemporal volume $D$. The volume is decomposed into a sum of three terms, each consisting of the Hadamard product of two 2D plane-based representations and a learned vector [S1].

The layout of the parameters is defined as follows:
*   $M_{rAB} \in \mathbb{R}^{A \times B}$: Learned 2D plane-based representations. The dimensions $(A, B)$ are pairs from $\{X, Y, Z, T\}^2$ such that $A \neq B$.
*   $v_{ri} \in \mathbb{R}^F$: Learned vectors along the feature axes.
*   $R_1, R_2, R_3$: The number of low-rank components for each of the three decomposition terms.
*   $R$: Total number of low-rank components, defined as $R = R_1 + R_2 + R_3$.
*   $F$: Feature size, representing RGB colors. Typical value is 3 [S1].
*   $N$: Spatial resolution.
*   $T$: Temporal resolution.

**Formulas:**
The 4D volume $D$ is represented as [S1]:

$$
D = \sum_{r=1}^{R_1} M_{rXY} \circ M_{rZT} \circ v_{r1} + \sum_{r=1}^{R_2} M_{rXZ} \circ M_{rYT} \circ v_{r2} + \sum_{r=1}^{R_3} M_{rYZ} \circ M_{rXT} \circ v_{r3}
$$

where $\circ$ denotes the Hadamard product (element-wise multiplication) [S1].

**Parameters:**
*   $F = 3$ (RGB colors) [S1].
*   $R \ll N$ (Total low-rank components are much smaller than spatial resolution) [S1].

**Steps:**
1.  **Decomposition:** Decompose the 4D spatiotemporal volume $D$ into the sum of three terms defined by the formula above, utilizing the direction-aware representation to capture dynamics from six directions [S1].
2.  **Feature Computation:** For a given space-time point, compute density and appearance features by:
    a. Extracting feature vectors from paired planes (e.g., $XY$ and $ZT$).
    b. Multiplying the feature vectors from the paired planes.
    c. Concatenating the multiplied results into a single vector.
    d. Multiplying this concatenated vector by $V$, a 2D tensor that stacks all learned vectors $v_{ri}$ [S1].
3.  **Opacity Query:** Query point opacities directly from the computed density features [S1].
4.  **Color Regression:** Regress RGB color values using a compact Multi-Layer Perceptron (MLP). The inputs to this MLP are the appearance features and the view directions [S1].
5.  **Rendering:** Synthesize final images via differentiable volumetric rendering along rays, optimizing the model parameters using a photometric loss function comparing rendered images with ground-truth images [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward(point, view_direction, params):
    """
    point: (x, y, z, t) coordinates of the query point
    view_direction: (vx, vy, vz) view vector
    params: Dictionary containing learned planes M, vectors v, and MLP weights
    """
    # 1. Retrieve plane features for the point
    # M_rAB is a 2D plane. We sample the value at the projected coordinates.
    # For term 1: XY and ZT planes
    val_xy_1 = sample_plane(params['M_1_XY'], point.x, point.y)
    val_zt_1 = sample_plane(params['M_1_ZT'], point.z, point.t)
    
    val_xz_1 = sample_plane(params['M_1_XZ'], point.x, point.z)
    val_yt_1 = sample_plane(params['M_1_YT'], point.y, point.t)
    
    val_yz_1 = sample_plane(params['M_1_YZ'], point.y, point.z)
    val_xt_1 = sample_plane(params['M_1_XT'], point.x, point.t)

    # 2. Compute contributions from all three terms
    # Note: The formula sums over r. Here we assume r=1 for simplicity of structure,
    # but in practice, loops over R1, R2, R3 are required.
    
    # Term 1
    term1 = val_xy_1 * val_zt_1 * params['v_1_1']
    
    # Term 2
    term2 = val_xz_1 * val_yt_1 * params['v_1_2']
    
    # Term 3
    term3 = val_yz_1 * val_xt_1 * params['v_1_3']
    
    # Sum of terms (simplified for single r, actual implementation sums over R)
    # In the actual model, this vector is concatenated and multiplied by V
    # The note states: "concatenates the multiplied results into a single vector, 
    # and then multiplies them by V RF, which stacks all vri into a 2D tensor."
    
    # Let's represent the concatenated feature vector before V multiplication
    # Assuming the Hadamard products produce scalar values per component r,
    # we stack them. However, the note implies the planes produce feature vectors
    # or the multiplication results are vectors. Given M is R^{AxB}, it's likely
    # the planes store feature vectors at each coordinate.
    
    # Re-reading carefully: "multiplies the feature vectors extracted from paired planes... 
    # concatenates... multiplies by V".
    # If M_rAB is a plane of features, let's assume it returns a vector of size F_plane.
    # But the formula shows M * M * v. v is R^F.
    # Standard plane-based NeRF (like Instant-NGP or similar baselines) often uses
    # element-wise multiplication of scalars from planes if planes are scalar fields,
    # or concatenation if they are feature fields.
    # The note says "Hadamard product of two 2D plane-based representations and a learned vector".
    # If M is R^{AxB}, it's a grid. If it stores features, it's R^{AxBxF}.
    # The formula M_{rXY} \circ M_{rZT} \circ v_{r1} suggests element-wise ops.
    # If M is scalar field, result is scalar. If M is feature field, result is vector.
    # Given "concatenates the multiplied results", it implies the result of M*M is a vector.
    
    # Let's assume M_rAB returns a feature vector f_rAB at the point.
    # Then f_xy = M_rXY(point.xy)
    # f_zt = M_rZT(point.zt)
    # The "Hadamard product" in the formula likely refers to the interaction of these features.
    # However, Hadamard product is element-wise. If f_xy and f_zt are vectors,
    # f_xy * f_zt is a vector.
    
    # Let's stick to the explicit steps in the note:
    # 1. Multiply feature vectors from paired planes.
    # 2. Concatenate multiplied results.
    # 3. Multiply by V.
    
    # Implementation of Step 2 & 3:
    features = []
    
    # Term 1
    f_xy = sample_plane_features(params['M_1_XY'], point.x, point.y)
    f_zt = sample_plane_features(params['M_1_ZT'], point.z, point.t)
    # Element-wise multiplication of feature vectors
    prod1 = f_xy * f_zt 
    features.append(prod1)
    
    # Term 2
    f_xz = sample_plane_features(params['M_1_XZ'], point.x, point.z)
    f_yt = sample_plane_features(params['M_1_YT'], point.y, point.t)
    prod2 = f_xz * f_yt
    features.append(prod2)
    
    # Term 3
    f_yz = sample_plane_features(params['M_1_YZ'], point.y, point.z)
    f_xt = sample_plane_features(params['M_1_XT'], point.x, point.t)
    prod3 = f_yz * f_xt
    features.append(prod3)
    
    # Concatenate results
    combined_features = concatenate(features)
    
    # Multiply by V (stacked v_r vectors)
    # V is RF x F_out? Or RF x F?
    # Note says "V RF, which stacks all vri into a 2D tensor".
    # If v_r is R^F, and we have R1+R2+R3 vectors, V is (R1+R2+R3) x F.
    # combined_features needs to align with the rows of V.
    # If prod1, prod2, prod3 are size F, then combined_features is 3*F.
    # This doesn't match V dimensions directly unless V is applied differently.
    # However, the note says "multiplies them by V".
    # Let's assume the standard plane-based fusion:
    # The "multiplied results" might be scalars if M is scalar, but "feature vectors" implies vectors.
    # If M is scalar, prod is scalar. Concatenated is vector of scalars.
    # V maps this to feature space.
    
    # Given the ambiguity of "feature vectors" vs scalar planes in the formula notation,
    # we follow the text description strictly:
    # "multiplies the feature vectors... concatenates... multiplies by V"
    
    # Let's assume the output of the plane sampling is a vector of size F_plane.
    # And the Hadamard product is element-wise.
    # The concatenation creates a vector of size 3 * F_plane.
    # V is a matrix of size (3 * F_plane) x F_feature.
    
    # For the purpose of the article, we define the operation as:
    density_feature = V @ combined_features # Matrix multiplication
    
    # 3. Query Opacity
    opacity = query_opacity(density_feature)
    
    # 4. Color Regression
    color = color_mlp(combined_features, view_direction)
    
    return opacity, color

def sample_plane(plane, u, v):
    """Bilinear interpolation of 2D plane at coordinates (u, v)"""
    # Standard bilinear interpolation
    pass

def sample_plane_features(plane, u, v):
    """Bilinear interpolation of feature vector from 2D plane"""
    # Standard bilinear interpolation for each feature channel
    pass

def query_opacity(density_feature):
    """Map density feature to opacity"""
    # Typically a sigmoid or similar activation on a linear projection
    pass

def color_mlp(features, view_direction):
    """Compact MLP for color regression"""
    # MLP taking features and view direction as input
    pass
```

**Edge Cases and Pitfalls:**
*   **Shift Variance:** Traditional 2D DWT suffers from shift variance, where small shifts in the input signal disrupt wavelet oscillation patterns, causing poor results in dynamic regions with motion or camera movement [S1]. DaReNeRF mitigates this using DTCWT properties [S1].
*   **Direction Ambiguity:** Traditional 2D DWT has poor direction selectivity, producing a checkered pattern that blends representations from $\pm 45^\circ$, leading to ghosting artifacts around moving objects [S1]. DaReNeRF avoids this by capturing features from six distinct orientations [S1].
*   **Memory Efficiency:** The method relies on $R \ll N$. If $R$ is not sufficiently small relative to $N$, the memory complexity reduction from $O(N^3 T F)$ to $O(R N^2 T F)$ may not be significant [S1].

## Operating characteristics
*   **Memory Complexity:** Reduced from $O(N^3 T F)$ for direct 4D volume modeling to $O(R N^2 T F)$ using the plane-based decomposition, where $N$ is spatial resolution, $T$ is temporal resolution, $F$ is feature size, and $R$ is the total number of low-rank components [S1].
*   **Performance:** Achieves state-of-the-art performance in novel view synthesis for dynamic scenes [S1].
*   **Strengths:** Robust to shift variance and direction ambiguity; efficient memory usage compared to direct 4D modeling; effective at modeling complex dynamic scenes without ghosting artifacts [S1].
*   **Limitations:** Introduces a 2D redundancy (with $d=2$ for plane-based decomposition) compared to DWT representation, resulting in lower storage efficiency than pure DWT-based methods, though this is traded for better accuracy [S1].
*   **When to use:** Suitable for novel view synthesis of dynamic scenes where high fidelity is required and artifacts from motion (ghosting) must be minimized [S1].
*   **When not to use:** When storage efficiency is the primary constraint and motion is minimal or static, simpler DWT-based or static NeRF methods might be preferred [S1].

## Background
DaReNeRF was introduced to address the limitations of implicit representations and straightforward 4D volume decomposition for re-rendering high-fidelity scenes with complex motions [S1]. It builds on the baseline plane-based 4D volume decomposition [7] and incorporates the Dual-Tree Complex Wavelet Transform (DTCWT) [47] to create a direction-aware representation [S1]. The method is inspired by the need to overcome shift variance and lack of direction selectivity in Discrete Wavelet Transforms (DWT) [S1]. It is used in the context of computer vision for reconstruction and re-rendering of 3D scenes from 2D images, with implications for AR/VR applications [S1].

## Sources
[S1] DaReNeRF: Direction-aware Representation for Dynamic Scenes, https://doi.org/10.1109/cvpr52733.2024.00481, fetched 2026-10-27
