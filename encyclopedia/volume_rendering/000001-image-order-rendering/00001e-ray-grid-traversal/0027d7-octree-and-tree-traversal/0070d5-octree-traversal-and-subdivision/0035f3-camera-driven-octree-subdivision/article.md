# Camera-Driven Octree Subdivision

Also called: Adaptive Subdivision, Frustum-Based Octree Subdivision, Leaf Node Construction, Octree Space Subdivision

## Overview
Camera-Driven Octree Subdivision constructs a hierarchical spatial partitioning for scene representation by subdividing space based on camera visibility and distance thresholds [S1][S2, unofficial]. It sets itself apart from sibling methods by driving the octree construction directly from camera positions and view frustums rather than geometric occupancy or traversal [S1][S2, unofficial].

## How it works
**Inputs:**
- Set of camera centers $\mathcal{C} = \{c_1, \dots, c_N\}$ [S1][S2, unofficial].
- Set of camera view frustums $\mathcal{F} = \{f_1, \dots, f_N\}$ [S1][S2, unofficial].
- Parameters: $\lambda = 3$ (subdivision threshold multiplier), $n_c = 4$ (maximum cameras per leaf) [S1][S2, unofficial].

**Outputs:**
- An octree structure with leaf nodes $\{S_i\}$ [S1][S2, unofficial].
- For each leaf node $S_i$, a subset of selected cameras $\mathcal{C}_{S_i} \subseteq \mathcal{C}$ with $|\mathcal{C}_{S_i}| \le n_c$ [S1][S2, unofficial].

**Data Structures:**
- **Octree Node:**
  - `center`: $\mathbf{c} \in \mathbb{R}^3$ (world coordinates) [G].
  - `side_length`: $s \in \mathbb{R}$ (scalar) [G].
  - `children`: Array of 8 child nodes (or `None` if leaf) [G].
  - `selected_cameras`: List of camera indices (populated only for leaves) [G].

**Algorithm Steps:**
1. **Initialize Root:** Compute the bounding box size $W$ of all camera centers $\mathcal{C}$ [S1][S2, unofficial]. Create a root node with side length $s_{\text{root}} = 512 \cdot W$ [S1][S2, unofficial].
2. **Recursive Subdivision:** Starting from the root, for each node with side length $s$ and center $\mathbf{c}$:
   - Retrieve all visible cameras whose view frustums intersect the node's bounding box [S1][S2, unofficial].
   - If any visible camera center $c_j$ satisfies $\|c_j - \mathbf{c}\| \le \lambda \cdot s$ (with $\lambda=3$), subdivide the node into 8 children with side length $s/2$ and recurse on each child [S1][S2, unofficial].
   - Otherwise, mark the node as a leaf and stop subdividing [S1][S2, unofficial].
3. **Camera Selection:** For each leaf node visible by more than $n_c=4$ cameras, select exactly $n_c$ cameras such that the minimal pairwise distance between the selected cameras is maximized [S1][S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def build_octree(camera_centers, camera_frustums, lambda_val=3, nc=4):
    # Compute bounding box size of all camera centers [G]
    bbox_size = compute_aabb_size(camera_centers)
    root_side = 512.0 * bbox_size
    root_center = compute_aabb_center(camera_centers)
    root = Node(center=root_center, side_length=root_side)
    subdivide(root, camera_centers, camera_frustums, lambda_val, nc)
    return root

def subdivide(node, centers, frustums, lambda_val, nc):
    s = node.side_length
    c = node.center
    # Retrieve visible cameras whose frustums intersect the node [S1][S2, unofficial]
    vis_indices = [i for i, f in enumerate(frustums) if intersects(f, node.bbox)]
    if not vis_indices:
        node.children = None
        node.selected_cameras = []
        return

    # Check if any visible camera center is within lambda * s of node center [S1][S2, unofficial]
    should_subdivide = any(distance(centers[i], c) <= lambda_val * s for i in vis_indices)

    if should_subdivide:
        # Subdivide into 8 children with side length s/2 [S1][S2, unofficial]
        node.children = [None] * 8
        child_side = s / 2.0
        for i in range(8):
            child_center = compute_child_center(c, i, child_side)
            child = Node(center=child_center, side_length=child_side)
            node.children[i] = child
            subdivide(child, centers, frustums, lambda_val, nc)
    else:
        # Mark as leaf node [S1][S2, unofficial]
        node.children = None
        node.selected_cameras = vis_indices
        if len(vis_indices) > nc:
            # Select nc cameras maximizing minimal pairwise distance [S1][S2, unofficial]
            node.selected_cameras = select_max_min_dist_cameras(vis_indices, centers, nc)
```

**Parameters:**
- $\lambda = 3$: Threshold for subdivision. If a camera is within $3s$ of the node center, subdivide [S1][S2, unofficial].
- $n_c = 4$: Maximum number of cameras associated with a single leaf node for warping function construction [S1][S2, unofficial].

**Edge Cases and Pitfalls:**
- **Root Size:** The root must be sufficiently large ($512 \times$ camera bounding box) to contain far-away objects like sky [S1][S2, unofficial].
- **Camera Selection:** The exact optimization algorithm for selecting cameras is not provided in the main text and is deferred to supplementary material [S1][S2, unofficial].

## Subtopics

### [Octree Texture Subdivision](13613-octree-texture-subdivision.md)
Octree Texture Subdivision is a method that subdivides the texture space into an octree to skip empty regions and use lower-resolution textures for distant or low-interest regions. It sets itself apart from sibling methods by operating directly on the texture coordinate space rather than the geometric voxel grid, allowing hardware texture interpolation to handle the hierarchical sampling.

### [Distance-Based Octree LOD Management](20878-distance-based-octree-lod-management.md)
Distance-Based Octree LOD Management optimizes rendering load by dynamically subdividing or merging octree chunks based on the camera's distance to the chunk center. It splits chunks into eight children when the camera is close and merges them back when the camera is far, serving as the core subdivision logic for [Camera-Driven Octree Subdivision](7636-camera-driven-octree-subdivision.md). Unlike [Octree Texture Subdivision](13613-octree-texture-subdivision.md), which manages LOD via texture mipmaps, this method explicitly creates and destroys geometric child nodes.

## Operating characteristics
Not given in the sources.

## Background
The method originates from F2-NeRF, a technique for fast Neural Radiance Field training with free camera trajectories [S1][S2, unofficial]. It was introduced to organize space based on camera visibility rather than static geometry for scene representation [S1][S2, unofficial].

## Sources
[S1] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
[S2] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://openaccess.thecvf.com/content/CVPR2023/html/Wang_F2-NeRF_Fast_Neural_Radiance_Field_Training_With_Free_Camera_Trajectories_CVPR_2023_paper.html, fetched 2026-10-27 (unofficial)
