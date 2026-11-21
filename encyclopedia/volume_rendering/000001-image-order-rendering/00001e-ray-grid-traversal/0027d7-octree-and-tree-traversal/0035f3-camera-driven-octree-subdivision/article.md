# Camera-Driven Octree Subdivision

Also called: Frustum-Based Octree Subdivision, Octree Space Subdivision

## Overview
Camera-Driven Octree Subdivision is a scene representation method that constructs a hierarchical spatial partitioning based on camera visibility rather than geometric occupancy. It solves the problem of efficiently organizing large-scale scenes with arbitrary camera trajectories by subdividing space only where cameras are present and close enough to require detail. It sets itself apart from sibling methods like [DDA Tree Traversal with Mini-Stack](5244-dda-tree-traversal-with-mini-stack.md) by focusing on the *construction* of the octree structure driven by camera positions, whereas siblings focus on the *traversal* of an existing structure. It differs from [Octree Texture Subdivision](13613-octree-texture-subdivision.md) by operating in geometric world space rather than texture coordinate space.

## How it works
The method constructs an octree where leaf nodes define regions for warping functions. The construction is driven by the positions and frustums of visible cameras.

**Inputs:**
- Set of camera centers $\mathcal{C} = \{c_1, \dots, c_N\}$.
- Set of camera view frustums $\mathcal{F} = \{f_1, \dots, f_N\}$.
- Parameters: $\lambda = 3$ (subdivision threshold multiplier), $n_c = 4$ (max cameras per leaf).

**Outputs:**
- An octree structure with leaf nodes $\{S_i\}$.
- For each leaf node $S_i$, a subset of selected cameras $\mathcal{C}_{S_i} \subseteq \mathcal{C}$ with $|\mathcal{C}_{S_i}| \le n_c$.

**Data Structures:**
- **Octree Node:**
  - `center`: $\mathbf{c} \in \mathbb{R}^3$ (world coordinates).
  - `side_length`: $s \in \mathbb{R}$ (scalar).
  - `children`: Array of 8 child nodes (or null if leaf).
  - `selected_cameras`: List of camera indices (only for leaves).

**Algorithm Steps:**

1.  **Initialize Root:**
    Compute the bounding box $\mathcal{B}_{cam}$ of all camera centers $\mathcal{C}$.
    Let $W$ be the side length of $\mathcal{B}_{cam}$.
    Create root node with side length $s_{root} = 512 \cdot W$.
    Center the root node to encompass $\mathcal{B}_{cam}$ with significant margin [S1].

2.  **Recursive Subdivision:**
    Define function `Subdivide(node, cameras)`:
    - Let $s$ be `node.side_length`.
    - Let $\mathbf{c}$ be `node.center`.
    - Filter `cameras` to keep only those whose view frustum $f_j$ intersects `node`'s bounding box [S1]. Let this set be $\mathcal{C}_{vis}$.
    - If $\mathcal{C}_{vis}$ is empty, mark `node` as leaf with no cameras and return [G].
    - Check subdivision condition:
      For each camera $c_j \in \mathcal{C}_{vis}$, calculate distance $d_j = \|c_j - \mathbf{c}\|$.
      If $\exists c_j$ such that $d_j \le \lambda \cdot s$:
        - Subdivide `node` into 8 children.
        - Each child has side length $s/2$.
        - Center each child appropriately relative to $\mathbf{c}$.
        - For each child, call `Subdivide(child, cameras)` [S1].
      Else:
        - Mark `node` as leaf.
        - If $|\mathcal{C}_{vis}| > n_c$:
          - Select a subset $\mathcal{C}_{sel} \subseteq \mathcal{C}_{vis}$ of size $n_c$ such that the minimal pairwise distance between selected cameras is maximized [S1].
          - Store $\mathcal{C}_{sel}$ in `node.selected_cameras`.
        - Else:
          - Store all $\mathcal{C}_{vis}$ in `node.selected_cameras`.
        - Return [S1].

3.  **Camera Selection Strategy (for leaves with > $n_c$ cameras):**
    - Given a set of $K > n_c$ visible cameras, find indices $I \subset \{1, \dots, K\}$ with $|I|=n_c$.
    - Optimize: $\max_{I} \min_{i, j \in I, i \neq j} \|c_i - c_j\|$ [S1].
    - Note: The exact optimization algorithm is in supplementary material [S1]. A greedy approach or k-means-like clustering can be used as an approximation [G].

**Parameters:**
- $\lambda = 3$: Threshold for subdivision. If a camera is within $3s$ of the node center, subdivide [S1].
- $n_c = 4$: Maximum number of cameras associated with a single leaf node for warping function construction [S1].

**Edge Cases and Pitfalls:**
- **Root Size:** The root must be sufficiently large ($512 \times$ camera bounding box) to contain far-away objects like sky [S1]. Failure to do so causes clipping of distant geometry.
- **Empty Nodes:** Nodes with no intersecting frustums should be pruned or marked as empty to save memory [G].
- **Precision:** Distance calculations $d_j$ should use double precision if the scene scale is large to avoid incorrect subdivision decisions due to floating-point error [G].

## Subtopics

### [Octree Texture Subdivision](13613-octree-texture-subdivision.md)
Operates on texture coordinate space rather than geometric voxel grid, allowing hardware texture interpolation to handle hierarchical sampling. Choose when working with texture-mapped volumes or when hardware interpolation is desired.

## Operating characteristics
- **Time Complexity:** Construction is $O(N \cdot L)$ where $N$ is the number of cameras and $L$ is the average number of nodes visited per camera during the check-and-subdivide process. The depth of the tree depends on the distribution of cameras and the $\lambda$ threshold.
- **Memory Complexity:** Proportional to the number of leaf nodes and the number of cameras stored per leaf. The large root bounding box ensures no spatial gaps but may increase initial memory allocation if not managed carefully [S1].
- **Strengths:** Efficiently handles large scenes with sparse camera coverage by not subdividing empty regions. Adapts resolution to camera density.
- **Limitations:** The subdivision is static based on camera positions; it does not adapt to scene geometry complexity directly. The choice of $\lambda$ and $n_c$ significantly impacts the tree structure and subsequent warping accuracy [S1].
- **When to use:** Large-scale scene reconstruction with known camera trajectories (e.g., NeRF training with free camera paths) where spatial organization is needed for efficient warping or sampling [S1].

## Background
The method originates from F2-NeRF, a technique for fast Neural Radiance Field training with free camera trajectories. It was introduced to handle the computational challenges of large-scale scenes by organizing the space based on camera visibility rather than static geometry [S1]. The approach allows for the construction of warping functions tailored to local camera configurations, improving training speed and reconstruction quality for dynamic or large-scale environments.

## Sources
[S1] F2-NeRF: Fast Neural Radiance Field Training with Free Camera Trajectories, https://doi.org/10.1109/cvpr52729.2023.00404, fetched 2026-10-26
