# Adaptive Grid Optical Tomography

Also called: Adaptive Tomography, Adaptive reconstruction algorithm, Octree-based Optical Tomography, Octree-based adaptive CT

## Overview
Adaptive Grid Optical Tomography is a volumetric reconstruction method that represents the 3D density field using an adaptive octree grid of piecewise constant basis functions, allowing for higher effective resolution and better memory efficiency than uniform grids by using fewer basis functions in uniform regions [S1]. It solves the problem of reconstructing dynamic, semi-transparent phenomena (such as smoke and flames) from multi-view video sequences by iteratively refining the grid resolution only where the reconstruction error is high, combined with a visual hull constraint to restrict the solution space [S1]. Unlike uniform grid approaches, which quickly exhaust memory limits for high-resolution multi-camera setups, this method uses 20-25% of the memory while achieving comparable accuracy [S1].

## How it works

### Inputs and Outputs
**Inputs:**
*   A synchronized multi-video sequence of $n_c$ cameras [S1].
*   Camera calibration parameters (intrinsics and extrinsics) for all cameras [G].
*   A set of background images for each camera [S1].
*   A visual hull (or silhouette masks) for each camera, derived from the background-subtracted foreground images [S1].

**Outputs:**
*   A sparse system matrix $S$ representing the projection of the current octree basis functions onto the image planes [S1].
*   A coefficient vector $a$ representing the density values of the active basis functions [S1].
*   The final adaptive octree structure defining the 3D density field [S1].

### Data Structures

**Sparse Matrix $S$:**
*   **Layout:** Index-stored sparse matrix (coordinate format). Stores two indices (row, column) and the matrix value for each non-zero entry [S1].
*   **Rows:** Correspond to pixels in the image plane across all cameras. Total rows $n_p$ is the number of pixels in one frame multiplied by the number of cameras $n_c$ [S1].
*   **Columns:** Correspond to basis functions (octree leaves) in the 3D volume.
*   **Entries:** $S_{ij}$ is the integral of basis function $\phi_j$ over the ray/pixel area for pixel $i$. Non-zero only if the support of $\phi_j$ projects onto pixel $i$ [S1].

**Octree Node Structure:**
*   Used to track the splitting process and identify geometry [S1].
*   **Fields:**
    *   `split_level`: Integer, depth from the root of the octree.
    *   `leaf_index`: Integer, the index the leaf would have in a uniformly split octree under a fixed traversal order (e.g., Morton order) [S1].
*   This structure allows the algorithm to determine the position and size of new basis functions generated during splitting and to compute the integral values for the new columns of $S$ [S1].

**Coefficient Vector $a$:**
*   Vector of density coefficients corresponding to the columns of $S$.
*   Constrained to be non-negative ($a_i \ge 0$) [S1].

**Residual Vector $r$:**
*   $r = p - S a$, where $p$ is the background-subtracted pixel vector [S1].

### Parameters
*   **$k$**: Number of basis functions to split per iteration. Typical value: $\sqrt{n_b}$, where $n_b$ is the current number of basis functions [S1].
*   **Max Split Level**: Maximum depth of the octree [S1].
*   **Non-negative Constraint**: Applied during the solver step [S1].

### Algorithm Steps

1.  **Preprocessing (Background Subtraction):**
    *   Record background images for all cameras [S1].
    *   Compute the median background image and standard deviation per pixel for each camera [S1].
    *   Classify pixels as foreground or background based on the standard deviation [S1].
    *   For foreground pixels, subtract the background value [S1]. For background pixels, set the value to zero [S1]. This computes the target vector $p$ (representing $p_{original} - b$) in advance [S1].

2.  **Initialization:**
    *   Initialize a uniform grid of box basis functions at the coarsest level (root of the octree) [S1].
    *   Construct the initial sparse projection matrix $S$ where rows correspond to pixels and columns to basis functions. $S_{ij}$ is non-zero if basis function $\phi_j$ projects onto pixel $p_i$ [S1].
    *   Apply **Visual Hull Restriction** to $S$ (see Substeps 2.1 and 2.2 below) to remove columns corresponding to basis functions with non-zero support completely outside the visual hull [S1].

    **2.1. Visual Hull Restriction (Column Pruning):**
    *   For each camera, create a binary vector marking basis functions that have non-zero support in the camera's silhouette (generalized cone) [S1].
    *   Compute the element-wise logical AND of all camera binary vectors to find the intersection of generalized cones (the visual hull) [S1].
    *   Restrict matrix $S$ to only include columns (basis functions) that are inside the visual hull intersection [S1].

    **2.2. Visual Hull Restriction (Row Handling):**
    *   Note: Rows corresponding to pixels outside the silhouette are kept but will have zero entries in the right-hand side vector $p$ (after background subtraction) [S1]. The basis functions on the boundary of the visual hull are likely to be split, allowing the boundary to be represented accurately [S1].

3.  **Adaptive Iteration Loop:**
    Repeat the following steps until convergence or maximum split level is reached:

    **Step 1: Solve for Coefficients**
    *   Solve the linear least squares system using the CGLS (Conjugate Gradient Least Squares) solver [S1].
    *   The system is: $S a = p$ [S1].
    *   In every iteration of CGLS, project the solution to the subspace of non-negative solutions by setting all negative entries of the solution vector $a$ to zero [S1].
    *   The theoretical least squares solution is given by:
        $$ a = (S^T S)^{-1} S^T (p - b) $$
        where $p$ is the background-subtracted pixel vector and $b$ is the background (already subtracted in preprocessing, so effectively $p$ in the code represents $p_{orig}-b$) [S1].

    **Step 2: Compute Residual and Error Measure** [S1]
    *   Compute the residual error vector:
        $$ r = p - S a $$
        [S1]
    *   Project the residual error onto the basis functions to get a measure of where to split [S1]. For each basis function $\phi_j$, compute the error measure $e_{\phi_j}$:
        $$ e_{\phi_j} = \frac{\sum_{i} S_{ij} |r_i|}{n_{\phi_j}} $$
        where the sum is over all pixels $i$ for which $S_{ij} \neq 0$, and $n_{\phi_j}$ is the number of cameras in which basis function $\phi_j$ is visible (i.e., the number of non-zero entries in column $j$ of $S$) [S1].
    *   *Note:* This formula weights the residual by the projection strength $S_{ij}$ to provide a better measure than simply summing absolute residuals [S1].

    **Step 3: Split Basis Functions**
    *   Identify the $k$ basis functions with the largest error measures $e_{\phi_j}$. $k$ is typically set to $\sqrt{n_b}$, where $n_b$ is the current number of basis functions [S1].
    *   Split the selected $k$ basis functions uniformly [S1].
        *   For box basis functions, this involves replacing the column in matrix $S$ with columns for the new sub-boxes (children in the octree) [S1].
        *   For linear basis functions, asymmetric basis functions must be introduced where splitting levels overlap [S1].
    *   Update matrix $S$:
        *   Remove the columns corresponding to the split basis functions [S1].
        *   Insert new columns for the new basis functions [S1].
        *   Recalculate the integral values for the new columns using the octree node structure (split level and leaf index) [S1].
    *   Re-apply Visual Hull Restriction to the updated matrix $S$ to ensure new basis functions outside the visual hull are removed [S1].

    **Step 4: Termination Check**
    *   Check if the estimate is good enough [S1].
    *   Stop splitting if:
        *   A maximum split level is reached.
        *   A basis function projects to less than two pixels in all images (Nyquist limit) [S1].
    *   If not converged, go back to Step 1.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_grid_optical_tomography(cameras, background_images, max_split_level):
    """
    Reconstructs a 3D density field from multi-view video using adaptive octree grid.
    
    Args:
        cameras: List of camera objects with calibration and projection functions.
        background_images: List of background images for each camera.
        max_split_level: Maximum depth of the octree.
        
    Returns:
        octree: The final adaptive octree structure.
        coefficients: The density coefficients for the active basis functions.
    """
    
    # 1. Preprocessing
    median_backgrounds = []
    std_devs = []
    for bg_img in background_images:
        median_backgrounds.append(compute_median(bg_img))
        std_devs.append(compute_std(bg_img))
        
    p = []
    for cam_idx, cam in enumerate(cameras):
        current_frame = get_current_frame(cam)
        bg = median_backgrounds[cam_idx]
        std = std_devs[cam_idx]
        for pixel in current_frame:
            if is_foreground(pixel, bg, std):
                p.append(pixel - bg)
            else:
                p.append(0.0)
                
    # 2. Initialization
    octree = Octree(root_level=0)
    S = build_sparse_matrix(cameras, octree)
    S = apply_visual_hull_restriction(S, cameras, octree)
    a = [0.0] * len(S.columns)
    
    # 3. Adaptive Iteration Loop
    while True:
        # Step 1: Solve using CGLS with non-negative projection
        a = cgls_solve(S, p, a, non_negative=True)
        
        # Step 2: Compute Residual and Error Measure
        r = [p[i] - sum(S[i][j] * a[j] for j in range(len(S.columns))) for i in range(len(p))]
        e = compute_error_measure(S, r)
        
        # Step 3: Split Basis Functions
        n_basis = len(S.columns)
        k = int(n_basis ** 0.5)
        split_indices = get_top_k_indices(e, k)
        
        can_split = True
        for idx in split_indices:
            bf = octree.get_basis_function(idx)
            if bf.projects_to_less_than_2_pixels(cameras) or bf.split_level >= max_split_level:
                can_split = False
                break
                
        if not can_split:
            break
            
        S, octree, a = split_basis_functions(S, octree, a, split_indices, cameras)
        S = apply_visual_hull_restriction(S, cameras, octree)
        
        if is_converged(r, S):
            break
            
    return octree, a

def compute_median(image):
    return sorted(image)[len(image) // 2]

def compute_std(image):
    mean = sum(image) / len(image)
    variance = sum((x - mean) ** 2 for x in image) / len(image)
    return variance ** 0.5

def get_current_frame(cam):
    return cam.capture()

def is_foreground(pixel, bg, std):
    return abs(pixel - bg) > 2 * std

class Octree:
    def __init__(self, root_level):
        self.root_level = root_level
        self.nodes = []
    def get_basis_function(self, idx):
        return self.nodes[idx]
    def add_children(self, children):
        self.nodes.extend(children)

def build_sparse_matrix(cameras, octree):
    # Constructs S where S_ij = integral of basis function phi_j over ray/pixel area
    # Implementation involves ray-tracing each basis function through each camera
    pass

def apply_visual_hull_restriction(S, cameras, octree):
    binary_vectors = []
    for cam in cameras:
        vec = [0] * len(S.columns)
        for j in range(len(S.columns)):
            if basis_function_in_silhouette(octree.get_basis_function(j), cam):
                vec[j] = 1
        binary_vectors.append(vec)
    hull_mask = [1] * len(S.columns)
    for vec in binary_vectors:
        hull_mask = [h & v for h, v in zip(hull_mask, vec)]
    S = S[:, [i for i, m in enumerate(hull_mask) if m]]
    return S

def compute_error_measure(S, r):
    n_phi = [sum(1 for val in S[i] if val != 0) for i in range(len(S.columns))]
    n_phi = [n if n > 0 else 1 for n in n_phi]
    weighted_abs_r = [[abs(r[i]) * S[i][j] for j in range(len(S.columns))] for i in range(len(r))]
    sum_weighted = [sum(weighted_abs_r[i][j] for i in range(len(r))) for j in range(len(S.columns))]
    return [sw / n for sw, n in zip(sum_weighted, n_phi)]

def get_top_k_indices(e, k):
    indexed_e = sorted(enumerate(e), key=lambda x: x[1], reverse=True)
    return [idx for idx, _ in indexed_e[:k]]

def split_basis_functions(S, octree, a, split_indices, cameras):
    mask = [1 if i not in split_indices else 0 for i in range(len(S.columns))]
    S = S[:, [i for i, m in enumerate(mask) if m]]
    a = [val for i, val in enumerate(a) if mask[i]]
    new_basis_functions = []
    for idx in split_indices:
        parent = octree.get_basis_function(idx)
        new_basis_functions.extend(parent.split())
    octree.add_children(new_basis_functions)
    new_S_cols = [compute_column_for_basis_function(child, cameras) for child in new_basis_functions]
    S = S + new_S_cols
    a.extend([0.0] * len(new_basis_functions))
    return S, octree, a

def compute_column_for_basis_function(bf, cameras):
    # Computes integral values for new columns using split level and leaf index
    pass

def cgls_solve(S, p, a, non_negative=False):
    # Standard CGLS algorithm solving min ||Sa - p||^2
    # Projects a to non-negative subspace (a = max(a, 0)) in each iteration if non_negative=True
    pass

def basis_function_in_silhouette(bf, cam):
    return bf.support_intersects_silhouette(cam)

def is_converged(r, S):
    return sum(x**2 for x in r) < 1e-6
```

## Operating characteristics

**Time and Memory Complexity:**
*   **Memory:** The method uses 20-25% of the memory required by a uniform grid approach [S1]. A uniform grid with 8 cameras at 320x240 resolution reaches the 2 GB memory limit on a 32-bit machine quickly, whereas the adaptive grid mitigates this by using fewer basis functions [S1].
*   **Time:** The method is more computationally intensive per iteration than filtered backprojection [S1]. The complexity involves solving a sparse linear system (CGLS) and updating the sparse matrix $S$ during splitting. The number of non-zero elements in $S$ is $O(\sqrt{n_b} \cdot n_p)$ per row, where $n_b$ is the number of basis functions and $n_p$ is the number of pixels [S1].

**Scaling:**
*   The number of basis functions grows sub-exponentially with the number of iterations when $k = \sqrt{n_b}$ [S1].
*   The method allows for higher resolution input images and reconstructed models compared to uniform grids due to the memory savings [S1].

**Strengths and Limitations:**
*   **Strengths:**
    *   Higher effective resolution than uniform grids for the same memory budget [S1].
    *   Handles dynamic scenes (time-varying volumetric models) [S1].
    *   Mathematically sound error projection heuristic (though noted as heuristic in future work) [S1].
    *   The visual hull restriction scheme is accurate up to the pixel level [S1].
*   **Limitations:**
    *   More computationally intensive per iteration than traditional methods like filtered backprojection [S1].
    *   The error projection is a heuristic; a mathematically sound error projection is identified as future work [S1].
    *   Splitting linear basis functions requires introducing asymmetric basis functions, complicating the implementation [S1].
    *   Non-negative constraint is enforced by projection, which may not be optimal for all basis function types (e.g., wavelets) [S1].

**When to Use:**
*   Reconstructing thin smoke and flames volumetrically from synchronized multi-video recordings [S1].
*   Scenes where memory is a constraint for high-resolution multi-camera setups [S1].
*   Dynamic scenes requiring adaptive resolution [S1].

**When Not to Use:**
*   When computational speed is more critical than memory usage or resolution [S1].
*   When the scene does not benefit from adaptive resolution (e.g., uniformly dense volumes) [S1].

## Background

**Origin:**
*   The method was presented by Ivo Ihrke and Marcus Magnor from the Max-Planck-Institut für Informatik, Saarbrücken, Germany [S1].
*   It was published in 2005 in the Eurographics proceedings [S1].

**History and Development:**
*   The method builds upon a basic computerized tomography (CT) technique presented in [IM04] (Ihrke and Magnor, 2004), which used an algebraic formulation for fire reconstruction [S1].
*   The adaptive grid approach was developed to overcome the memory limitations of uniform grids and to improve reconstruction quality for dynamic phenomena [S1].
*   Future work mentioned in the source includes deriving a mathematically sound error projection, using wavelet bases, and modeling occluding objects in flames and smoke [S1].

**Usage:**
*   Used for reconstructing dynamic volumetric models of flames and thin smoke from multi-camera video sequences [S1].
*   Applicable to other tomography problems, such as real X-ray data, as suggested by the authors [S1].

## Sources

[S1] Adaptive Grid Optical Tomography, https://core.ac.uk/works/26024869, fetched 2026-11-03
