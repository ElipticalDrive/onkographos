# Sketch and Patch

Also called: Sketch Gaussian Extraction, Sketch and Patch: Efficient 3D Gaussian Representation for Man-Made Scenes

## Overview
Sketch and Patch is a hybrid 3D Gaussian representation method that categorizes Gaussians into two distinct roles: **Sketch Gaussians**, which define scene boundaries and high-frequency features, and **Patch Gaussians**, which cover smoother, broader regions [S1]. It solves the problem of storage inefficiency and redundancy in standard 3D Gaussian Splatting (3DGS), where adaptive density control overestimates densification needs in high-frequency areas, leading to redundant Gaussian clusters along edges [S1]. Unlike conventional 3D downsampling or tiling which degrade visual quality due to the tightly coupled, heterogeneous nature of 3DGS Gaussians, this method separates Gaussians by role to allow specialized encoding [S1].

## How it works
**Inputs:** Multi-view images and camera poses.
**Outputs:** A unified hybrid representation consisting of encoded Sketch Gaussians and optimized Patch Gaussians.

### Data Structures
1.  **3D Line Segment ($L_i$):** Defined by endpoints $p_{start}, p_{end} \in \mathbb{R}^3$ and parameter $t \in [0, 1]$.
    *   Layout: Two 3D vectors ($p_{start}, p_{end}$).
2.  **Sketch Gaussian Candidate Set ($S_i$):** A subset of original 3DGS Gaussians associated with line segment $L_i$.
3.  **Parametric Model:** A function (e.g., polynomial or spline) encoding the attributes of Sketch Gaussians along the line.
4.  **Patch Gaussian Set:** The remaining Gaussians after Sketch extraction, subject to pruning.

### Numbered Steps

1.  **Extract 3D Line Segments:**
    Process input images and camera poses using a geometry-guided line reconstruction method, specifically **Line3D++** [S1]. This abstracts object boundaries into discrete 2D line segments and back-projects them into 3D space to obtain a set of 3D line segments $\{L_i\}$ [S1].

2.  **Identify Sketch Gaussians:**
    For each 3D line segment $L_i$, identify candidate Gaussians concentrated along the line.
    *   **Radius Search:** A Gaussian splat $G_j$ with center $\mu_j \in \mathbb{R}^3$ is included in the candidate set $S_i$ if its distance to the line segment is within a radius $r$:
        $$ d(\mu_j, L_i) = \min_{t \in [0,1]} \|\mu_j - L_i(t)\| \le r $$
        where the line segment is parameterized as:
        $$ L_i(t) = (1 - t)p_{start} + t p_{end}, \quad t \in [0, 1] $$
        [S1]
    *   **RANSAC Filtering:** Apply Random Sample Consensus (RANSAC) [4] to the candidate set $S_i$ to ensure coherent attribute values among the selected Sketch Gaussians, refining the set to robustly represent the line [S1].

3.  **Encode Sketch Gaussians:**
    Encode the identified Sketch Gaussians using parametric models (such as polynomials or splines) defined on the given 3D line $L_i$ [S1]. This leverages the geometric coherence of Gaussians along edges to model their attributes compactly, significantly reducing storage requirements [S1].

4.  **Optimize Patch Gaussians:**
    For the remaining smoother regions represented by Patch Gaussians, apply pruning and retraining [S1]. This sparsifies their distribution while maintaining visual fidelity and volumetric consistency [S1].

5.  **Integrate Representation:**
    Combine the encoded Sketch Gaussians and the optimized Patch Gaussians into a unified hybrid representation for efficient storage, transmission, and rendering [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def sketch_and_patch(images, camera_poses, original_gaussians, radius_threshold):
    """
    Implements the Sketch and Patch method for 3D Gaussian Splatting.
    
    Args:
        images: List of input images.
        camera_poses: List of camera poses corresponding to images.
        original_gaussians: List of 3D Gaussian objects from standard 3DGS training.
            Each Gaussian has attributes: center (mu), covariance, opacity, sh_coeffs.
        radius_threshold: Float, distance threshold for radius search.
        
    Returns:
        sketch_gaussians: List of encoded Sketch Gaussian parameters.
        patch_gaussians: List of optimized Patch Gaussian objects.
    """
    
    # Step 1: Extract 3D Line Segments using Line3D++
    # Line3D++ takes images and poses, outputs 3D line segments.
    line_segments = line3d_plus_plus_extract(images, camera_poses)
    
    sketch_gaussians = []
    patch_gaussians = []
    
    # Step 2 & 3: Identify and Encode Sketch Gaussians
    for line in line_segments:
        p_start = line.p_start
        p_end = line.p_end
        
        # Candidate selection via radius search
        candidates = []
        for g in original_gaussians:
            # Calculate distance from Gaussian center to line segment
            # L(t) = (1-t)*p_start + t*p_end for t in [0, 1]
            dist = min_distance_point_to_segment(g.center, p_start, p_end)
            if dist <= radius_threshold:
                candidates.append(g)
        
        # Refine using RANSAC to ensure attribute coherence
        # RANSAC filters outliers based on geometric and attribute consistency
        refined_sketch_candidates = ransac_filter(candidates, line)
        
        # Encode using parametric model (e.g., polynomial regression along the line)
        # This compresses the attributes of the Gaussians along the line
        encoded_sketch = parametric_encode(refined_sketch_candidates, line)
        sketch_gaussians.append(encoded_sketch)
        
        # Mark these Gaussians as processed to exclude from Patch set
        mark_as_processed(refined_sketch_candidates)

    # Step 4: Optimize Patch Gaussians
    # Collect all original Gaussians not marked as Sketch
    unprocessed_gaussians = [g for g in original_gaussians if not g.is_processed]
    
    # Apply pruning and retraining to sparsify and maintain fidelity
    # Pruning removes low-contribution Gaussians
    # Retraining optimizes the remaining ones
    patch_gaussians = prune_and_retrain(unprocessed_gaussians)
    
    # Step 5: Integration is implicit in the return of the two sets
    # Rendering uses both encoded Sketch and optimized Patch sets
    
    return sketch_gaussians, patch_gaussians

def min_distance_point_to_segment(point, p1, p2):
    """
    Calculates the minimum distance from a point to a line segment defined by p1 and p2.
    """
    # Vector from p1 to p2
    line_vec = p2 - p1
    # Vector from p1 to point
    point_vec = point - p1
    
    # Project point_vec onto line_vec to find t
    # t = dot(point_vec, line_vec) / dot(line_vec, line_vec)
    line_len_sq = dot(line_vec, line_vec)
    if line_len_sq == 0:
        return norm(point - p1)
    
    t = dot(point_vec, line_vec) / line_len_sq
    
    # Clamp t to [0, 1] for segment
    t = max(0, min(1, t))
    
    # Closest point on segment
    closest = p1 + t * line_vec
    
    return norm(point - closest)

def ransac_filter(candidates, line):
    """
    Filters candidates using RANSAC to ensure coherent attributes.
    Implementation detail: [G] Standard RANSAC loop fitting a model to attributes.
    """
    # [G] Placeholder for specific RANSAC implementation details not fully specified in source
    # The source states RANSAC is used to ensure coherent attribute values.
    # A typical implementation would fit a model (e.g., linear regression of opacity/sh along line)
    # and keep inliers.
    return candidates # Simplified return for structure; actual filtering logic depends on attribute model

def parametric_encode(candidates, line):
    """
    Encodes Gaussian attributes using a parametric model along the line.
    """
    # [G] Placeholder for specific polynomial/spline fitting logic
    # The source states Sketch Gaussians are encoded via polynomial regression.
    return None # Encoded parameters

def prune_and_retrain(gaussians):
    """
    Prunes and retrains Patch Gaussians.
    """
    # [G] Placeholder for pruning/retraining logic
    # The source states pruning and retraining are applied to sparsify distribution.
    return gaussians
```

### Parameters
*   **Line Reconstruction Method:** Line3D++ [S1].
*   **Radius Threshold ($r$):** Distance threshold for including a Gaussian in the Sketch candidate set. Not given in the sources.
*   **Parametric Model Type:** Polynomials or splines. Not given in the sources.

### Edge Cases and Pitfalls
*   **Redundancy in High-Frequency Areas:** Standard 3DGS adaptive density control relies on 2D positional gradients, which often overestimates the need for densification in high-frequency areas like edges and contours. This leads to dense Gaussian clusters along structural boundaries (architectural edges, furniture boundaries) and increased computational overhead [S1]. Sketch and Patch addresses this by explicitly identifying and encoding these regions rather than densifying blindly.
*   **Scene Type:** The method is especially effective in man-made scenes with well-defined straight edges and planar surfaces (architectural elements, furniture, manufactured objects) [S1]. Performance on natural scenes may vary as the "Sketch" prior relies on geometric coherence along lines.

## Operating characteristics
*   **Storage Efficiency:** For an indoor scene, the model maintains visual quality with 7.9% of the original 3DGS model size [S1].
*   **Visual Quality:** Achieves up to 11.52% improvement in PSNR, 1.83% in SSIM, and 19.20% in LPIPS at equivalent model sizes across diverse indoor and outdoor scenes [S1].
*   **Strengths:** Significant reduction in storage requirements through compact encoding of Sketch Gaussians; improved visual fidelity at low bitrates compared to standard 3DGS or naive downsampling.
*   **Limitations:** Relies on the presence of extractable 3D lines; less effective in scenes lacking well-defined geometric structures (natural environments).
*   **When to Use:** Man-made scenes with clear edges and planar surfaces where storage efficiency is critical.
*   **When Not to Use:** Natural scenes with complex, non-linear, or ambiguous boundaries where the Sketch/Patch separation is less distinct.

## Background
*   **Origin:** Proposed in the paper "Sketch and Patch: Efficient 3D Gaussian Representation for Man-Made Scenes" [S1].
*   **History:** Developed as a response to the storage inefficiencies of 3D Gaussian Splatting, which generates vast numbers of Gaussians. While 3DGS achieves impressive visual fidelity, standard compression techniques like 3D downsampling or tiling are unsuitable due to the heterogeneous and interdependent nature of Gaussian attributes [S1].
*   **Usage:** Applied to efficient storage, transmission, and rendering of 3D scenes, particularly in multimedia streaming applications [S1].

## Sources
[S1] Sketch and Patch: Efficient 3D Gaussian Representation for Man-Made Scenes, https://doi.org/10.1145/3712677.3720466, fetched 2026-10-26
