# Deformation Network (Static)


## Overview
The Deformation Network (Static) is a neural module that maps 3D points from an original camera space to a canonical voxel space by predicting a static displacement vector [S2]. It solves the problem of temporal alignment in dynamic scenes by ensuring that corresponding 3D points from different time steps map to the same position in the canonical space [S2]. Unlike variants that predict time-varying deformations, this method assumes the geometric deformation is fixed or approximates the dominant static structure, relying on shallow MLPs to enhance features for time-varying color and density [S2].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D point coordinate $\mathbf{p} = (x, y, z)$ and, in dynamic variants, a time index $t_i$ [S2]. In the static implementation, only the 3D coordinates are typically used for the core deformation [S1, unofficial].
*   **Output:** A deformed 3D point $\mathbf{p}' = (x', y', z')$ in the canonical space [S2].

**Data Structures:**
*   **Grid Field:** A 3D feature grid storing latent features at voxel locations. Accessed via trilinear interpolation or nearest-neighbor lookup based on $\mathbf{p}$ [S1, unofficial].
*   **Static MLP:** A shallow Multi-Layer Perceptron (typically 2-3 layers) that takes grid features as input and outputs a 3D displacement vector [S2].

**Steps:**
1.  **Feature Query:** Query the grid field for features at the input 3D point coordinates $\mathbf{p}$ [S1, unofficial]. Let this feature vector be $\mathbf{f}$ [S1, unofficial].
    $$ \mathbf{f} = \text{GridQuery}(\mathbf{p}) $$
2.  **Displacement Prediction:** Pass the queried features $\mathbf{f}$ through the static MLP to predict the displacement vector $\Delta \mathbf{p} = (\Delta x, \Delta y, \Delta z)$ [S2].
    $$ \Delta \mathbf{p} = \text{StaticMLP}(\mathbf{f}) $$
3.  **Deformation:** Add the displacement vector to the original coordinates to obtain the deformed point $\mathbf{p}'$ [S2].
    $$ \mathbf{p}' = \mathbf{p} + \Delta \mathbf{p} $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def forward_static(grid, static_mlp, rays_pts_emb):
    """
    Deforms 3D points from camera space to canonical space.
    
    Args:
        grid: 3D feature grid (e.g., torch.nn.Embedding or TrilinearGrid)
        static_mlp: Shallow MLP (2-3 layers) taking grid features and outputting 3D displacement
        rays_pts_emb: Tensor of shape (N, D) containing ray embeddings. 
                      The first 3 columns are the 3D coordinates (x, y, z).
    
    Returns:
        deformed_pts: Tensor of shape (N, 3) containing deformed 3D coordinates.
    """
    # Step 1: Query grid features at 3D point coordinates
    # rays_pts_emb[:, :3] extracts the (x, y, z) coordinates
    grid_feature = grid(rays_pts_emb[:, :3])
    
    # Step 2: Compute displacement using the static MLP
    # static_mlp takes grid_feature and outputs a 3D vector (dx, dy, dz)
    dx = static_mlp(grid_feature)
    
    # Step 3: Return deformed points by adding displacement to original coordinates
    deformed_pts = rays_pts_emb[:, :3] + dx
    
    return deformed_pts
```

**Parameters:**
*   **MLP Depth:** 2 to 3 layers [S2].
*   **Loss Function (Static Part):** The static deformation is optimized using a composite loss:
    $$ \mathcal{L}^s = \mathcal{L}^s_c + \lambda_{\text{reproj}}^s \mathcal{L}_{\text{reproj}}^s + \lambda_{\text{disp}}^s \mathcal{L}_{\text{disp}}^s + \lambda_{\text{monodepth}}^s \mathcal{L}_{\text{monodepth}}^s $$
    where $\mathcal{L}^s_c$ is the color reconstruction loss, and the other terms are reprojection, disparity, and monodepth losses weighted by $\lambda$ [S2].

**Edge Cases and Pitfalls:**
*   **Precision:** Adding small displacement vectors to large coordinate values may suffer from floating-point precision issues if not handled carefully [G].
*   **Boundary Conditions:** The grid query must handle points outside the defined voxel bounds, typically by clamping or returning zero features [G].

## Subtopics

### [ObjectNet Training](19834-objectnet-training.md)
ObjectNet Training introduces a category-specific prior by inserting a dedicated network (ObjectNet) before the PatchNet to regress patch latent codes and extrinsics from a shared object latent code. This architecture increases network capacity and establishes coarse geometric correspondences across different instances of the same category. It is distinct from standard PatchNet training by decoupling the object-level identity from patch-level details via a two-stage latent regression process.

### Deformation MLP
A neural network that deforms 3D points from the original camera space to a canonical voxel space, indicating that the voxel is a canonical space where corresponding 3D points from different times point to the same position [S2]. It typically consists of shallow MLPs (two to three layers) that enhance queried features from the canonical voxel to predict time-varying color and density [S2]. Most time-varying effects are carried out by the coordinate deformation MLP [S2]. The process involves: 1. Passing 3D coordinates $(x, y, z)$ and time index $t_i$ to a coordinate deformation MLP. 2. The MLP predicts 3D time-varying deformation vectors $(\Delta x, \Delta y, \Delta z)$. 3. Adding these deformations to original coordinates to get deformed coordinates $(x', y', z')$ [S2].

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of sampled points $N$, as each point requires one grid query and one MLP forward pass [S2].
*   **Memory Complexity:** Dominated by the storage of the 3D feature grid and the MLP weights [S2].
*   **Strengths:** Efficiently handles dynamic scenes by mapping them to a static canonical space, allowing the use of standard static rendering techniques on the deformed points [S2]. The shallow MLPs reduce computational overhead compared to deeper networks [S2].
*   **Limitations:** Assumes a static or dominant static deformation; may struggle with highly complex, non-rigid, or purely temporal deformations that cannot be captured by a single canonical space [S2].
*   **When to Use:** Suitable for dynamic view synthesis where a canonical representation can effectively align temporal data [S2].
*   **When Not to Use:** When the scene exhibits significant non-rigid deformation that varies significantly over time and cannot be approximated by a static displacement field [S2].

## Background
The Deformation Network (Static) originates from research in dynamic radiance field reconstruction, specifically in methods that handle temporal information by mapping points to a canonical space [S2]. The approach was introduced in "Robust Dynamic Radiance Fields" (CVPR 2023) [S2]. It builds on the idea that dynamic scenes can be represented more efficiently if points from different times are aligned in a canonical coordinate system [S2]. The static variant focuses on the geometric deformation component, while time-varying appearance is handled by subsequent shallow MLPs [S2].

## Sources
[S1] ElenaViewSynthesis/4dgs, https://github.com/ElenaViewSynthesis/4dgs, fetched 2026-10-28 (unofficial)
[S2] Robust Dynamic Radiance Fields, https://doi.org/10.1109/cvpr52729.2023.00010, fetched 2026-10-26
