# CCNeRF

Also called: Compressible-Composable NeRF, Rank-residual Decomposition NeRF

## Overview
CCNeRF is a neural field representation that enables efficient compression and composition of 3D scenes by learning a hybrid tensor rank decomposition of the scene without using neural networks for rendering [S1]. Unlike NeRF variants that rely on a shared MLP renderer (e.g., NSVF) or use MLPs that prevent direct compression/composition (e.g., TensoRF), CCNeRF models the radiance field *only* with tensor rank decomposition, allowing arbitrary composition via rank concatenation and dynamic size adjustment via rank truncation without retraining [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of 2D images of a 3D scene with known camera poses.
*   **Output:** A compressed, composable 3D representation consisting of decomposed rank components and a weight matrix, capable of rendering novel views via standard volume rendering.

**Data Structures:**
The model represents a 3D scene as a 4D feature volume described by rank components and a weight matrix [S1]. The structure consists of:
1.  **Rank Weights Matrix ($W$):** A matrix storing weights for different feature channels.
2.  **Decomposed Rank Components:** A set of components that are either vector-based (CP decomposition) or matrix-based (triple plane variant) [S1].
    *   **Vector-based (CP):** Factor matrices $U^x, U^y, U^z$ for spatial dimensions.
    *   **Matrix-based (Triple Plane):** Factor matrices $\mathcal{U}^{xy}, \mathcal{U}^{xz}, \mathcal{U}^{yz}$ representing 2D planes [S1].
    *   The ratio of vector- to matrix-based components is controllable to trade off between model size and performance [S1].
3.  **Transformation Matrices:** For composition, each object stores a transformation matrix to control its position and orientation in the composed scene [S1].

**Formulas:**
The rendering follows the standard volume rendering equation [S1]:
$$ \hat{C}(r) = \sum_i T_i \alpha_i c_i $$
where $T_i = \prod_{j<i} (1 - \alpha_i)$, $\alpha_i = 1 - \exp(-\sigma_i \delta_i)$, and $\delta_i = t_{i+1} - t_i$. Here, $\sigma_i$ is density, $c_i$ is color, and $t_i$ is the sample position along ray $r$ from origin $o$ with direction $d$ [S1].

The core representation uses a hybrid tensor rank decomposition. For a 3D tensor $T \in \mathbb{R}^{H \times W \times D}$, the Tucker decomposition is defined as [S1]:
$$ T_{i,j,k} = \sum_{p=1}^P \sum_{q=1}^Q \sum_{r=1}^R S_{p,q,r} U^x_{i,p} U^y_{j,q} U^z_{k,r} $$
where $S$ is the core tensor and $U^x, U^y, U^z$ are factor matrices [S1]. CCNeRF employs a **rank-residual learning strategy** to encourage lower ranks to preserve more important information, enabling near-optimal low-rank approximation at any targeted rank via an empirical sort-and-truncate strategy [S1].

**Steps:**
1.  **Scene Representation:** Learn the hybrid tensor rank decomposition of the scene. The scene is modeled by a 4D feature volume described by a set of rank components and a weight matrix [S1].
2.  **Querying a 3D Coordinate:**
    a. Project the 3D coordinate to the decomposed vectors or matrices [S1].
    b. Perform weighted interpolation using the stored rank weights [S1].
3.  **Compression (Rank Truncation):**
    a. Sort the rank components based on their importance (derived from the rank-residual learning).
    b. Truncate the components to a target rank $R_{target}$. This dynamically adjusts model size and detail level without retraining [S1].
4.  **Composition:**
    a. To compose different models, concatenate their rank components along the rank dimension [S1].
    b. Record a transformation matrix for each object to control its position and orientation [S1].
    c. To mitigate storage growth in complex scenes, compress less important objects by applying rank truncation to them [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def query_ccnerf(x, rank_components, weights):
    """
    Query the radiance field at coordinate x.
    x: 3D coordinate (x, y, z)
    rank_components: List of decomposed components (vectors or matrices)
    weights: Matrix of rank weights
    """
    # Project coordinate to decomposed vectors/matrices
    projected_values = []
    for comp in rank_components:
        if comp.type == 'vector':
            # Vector-based (CP) projection
            val = comp.Ux[x.x] * comp.Uy[x.y] * comp.Uz[x.z]
        elif comp.type == 'matrix':
            # Matrix-based (Triple Plane) projection
            val = comp.Uxy[x.x, x.y] + comp.Uxz[x.x, x.z] + comp.Uyz[x.y, x.z]
        projected_values.append(val)
    
    # Weighted interpolation
    # Note: Exact interpolation formula depends on specific hybrid implementation details
    # not fully specified in sources, but generally involves combining projected values
    # with the weight matrix W.
    # [G] Assuming a linear combination weighted by the rank weights for simplicity
    # as the source describes "weighted interpolation".
    result = sum(p * w for p, w in zip(projected_values, weights))
    return result

def compress_ccnerf(rank_components, target_rank):
    """
    Compress the model by truncating rank components.
    rank_components: List of rank components sorted by importance
    target_rank: Number of components to keep
    """
    # Empirical sort-and-truncate strategy
    # [G] Assumes components are pre-sorted by importance metric derived from
    # rank-residual learning.
    return rank_components[:target_rank]

def compose_ccnerf(models):
    """
    Compose multiple CCNeRF models into one scene.
    models: List of model objects, each containing rank_components and transform_matrix
    """
    composed_rank_components = []
    composed_transforms = []
    
    for model in models:
        # Concatenate rank components along the rank dimension
        composed_rank_components.extend(model.rank_components)
        composed_transforms.append(model.transform_matrix)
        
    return {
        'rank_components': composed_rank_components,
        'transforms': composed_transforms
    }
```

**Parameters:**
*   **Target Rank:** The number of components kept in the decomposition determines the model size and detail level. Lower ranks preserve primary information due to rank-residual learning [S1].
*   **Ratio of Vector- to Matrix-based Components:** Controllable to trade off between model size and performance [S1].

**Edge Cases and Pitfalls:**
*   Standard truncated Higher-Order SVD (HOSVD) does not hold the optimal property of truncated SVD, but yields a quasi-optimal solution sufficient for practical use [S1].
*   The method relies on the assumption that the scene exhibits low-rank structure; if the scene is highly complex or noisy, compression artifacts may appear.

## Subtopics

### Rank-Residual Learning
A strategy to encourage lower ranks to preserve more important information, enabling near-optimal low-rank approximation [S1]. Choose when dynamic compression without retraining is required. [CCNeRF](15008-ccnerf.md)

### Hybrid Tensor Decomposition
Combines CP (vector-based) and Triple Plane (matrix-based) decompositions. Choose when a balance between compactness and representational power is needed. [CCNeRF](15008-ccnerf.md)

## Operating characteristics
*   **Time Complexity:** Querying is $O(R)$ where $R$ is the number of rank components, significantly faster than MLP-based NeRFs which are $O(D)$ where $D$ is the MLP dimension. Compression and composition are $O(1)$ for concatenation and $O(R \log R)$ for sorting during truncation.
*   **Memory Complexity:** $O(R \cdot (H+W+D))$ for CP components and $O(R \cdot (HW+HD+WD))$ for Triple Plane components, compared to $O(H \cdot W \cdot D)$ for dense volumes.
*   **Measured Performance:** The method achieves comparable rendering quality to state-of-the-art methods while enabling extra capabilities of compression and composition [S1].
*   **Strengths:** No neural networks for rendering, allowing arbitrary composition without retraining or compatibility constraints; dynamic model size adjustment via rank truncation; near-optimal compression without extra optimization [S1].
*   **Limitations:** Quasi-optimal solution for truncated HOSVD; requires careful selection of target rank and component type ratio [S1].
*   **When to use:** When compression and composition of 3D scenes are required without the overhead of neural network inference for rendering.
*   **When not to use:** When the highest possible rendering quality is required without any compression artifacts, or when the scene does not exhibit low-rank structure.

## Background
CCNeRF was introduced by Jiaxiang Tang, Xiaokang Chen, Jingbo Wang, and Gang Zeng from Peking University and Chinese University of Hong Kong in 2022 [S1]. It addresses the difficulty of manipulating NeRF models (like compression and composition) caused by their implicit representation and shared renderer networks [S1]. The method is motivated by the low-rank approximation property of the SVD algorithm and builds on tensor decomposition techniques used in vision tasks [S1]. It contrasts with TensoRF, which uses MLPs that prevent compression/composition, by modeling the field *only* with tensor rank decomposition [S1].

## Sources
[S1] Compressible-Composable NeRF Via Rank-Residual Decomposition, https://doi.org/10.52202/068431-1076, fetched 2026-10-26
