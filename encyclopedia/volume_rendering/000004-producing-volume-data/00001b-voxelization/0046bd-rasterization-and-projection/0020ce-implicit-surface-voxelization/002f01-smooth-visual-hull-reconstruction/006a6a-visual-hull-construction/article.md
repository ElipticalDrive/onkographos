# Visual Hull Construction

Also called: Octree Visual Hull Construction, Octree carving, Photo Hull, Shape from Silhouette, Shape from Silhouette (SfS) / Octree Visual Hull Construction, Silhouette Intersection, Visual hull reconstruction

## Overview
Visual Hull Construction generates a 3D representation of an object by intersecting multiple 3D cones created by back-projecting 2D silhouettes from different views onto 3D space [S1]. It is a Shape from Silhouette (SfS) technique that represents the resulting volume using an octree, where the object is defined as the intersection of all view-consistent volumes [S1]. Unlike [Smooth Visual Hull Reconstruction](5917-smooth-visual-hull-reconstruction.md), which uses a continuous product of filtered silhouette values to define an implicit surface, Visual Hull Construction relies on discrete octree subdivision and binary classification of spatial regions [S1].

## How it works
**Inputs:** A set of $N$ 2D images of the object taken from different viewpoints, camera projection matrices for each view, and an initial 3D bounding cube enclosing the object.
**Outputs:** An octree data structure representing the visual hull, where nodes are classified as `inside`, `outside`, or `intersection`.

**Data Structures:**
*   **Octree Node:** Represents a cubic volume in 3D space.
    *   `status`: Enum {OUTSIDE, INSIDE, INTERSECTION}.
    *   `children`: Array of 8 child nodes (for INTERSECTION nodes).
    *   `bbox`: Bounding box of the cube (min/max coordinates).
*   **Silhouette Mask:** Binary 2D image where 1 indicates the object silhouette and 0 indicates background.

**Parameters:**
*   $\epsilon$: Threshold for binary mask generation. Typical value: 0.04 [S2].
*   $\sigma_{img}$: Blur kernel size for image masks. Typical value: 1 pixel [S2].
*   $\sigma_{vol}$: Blur kernel size for volume hulls. Typical value: 0.5 cells [S2].
*   $K$: Camera intrinsic matrix. Constant if the same camera is used [S1].
*   $\theta$: Rotational angle from reference position [S1].
*   $t_x, t_y$: X and Y translations at $\theta=0$ [S1].

**Steps:**

1.  **Preprocessing:**
    *   For each input image, perform background subtraction to isolate the object.
    *   Convert the result to a binary mask using threshold $\epsilon$. Pixels with intensity $> \epsilon$ are set to 1, others to 0 [S2].
    *   Apply a Gaussian blur with $\sigma_{img}$ to the binary masks to smooth edges [S2].

2.  **Camera Model Setup:**
    *   Define the projection matrix $P(\theta)$ for each view. For circular motion (e.g., turntable), use:
        $$ P(\theta) = K \begin{bmatrix} \cos \theta & \sin \theta & 0 & t_x \\ -\sin \theta & \cos \theta & 0 & t_y \\ 0 & 0 & 1 & 0 \end{bmatrix} $$
        where $\theta$ is the rotational angle, and $t_x, t_y$ are translations at $\theta=0$ [S1].
    *   If only a single view is available, generate 4 auxiliary views by rotating the camera around the central Y-axis. Mirror the masks from these auxiliary views at the projected rotation axis to reduce cutoffs [S2].

3.  **Octree Construction (Carving):**
    *   Initialize the root node with the initial bounding cube. Set its status to `INSIDE`.
    *   Recursively process each node:
        a.  **Project:** Project the 3D bounding box of the current node onto the 2D image plane for all $N$ views using $P(\theta)$.
        b.  **Intersect:** For each view, check if the projected 2D bounding box intersects the corresponding preprocessed silhouette mask.
        c.  **Classify:**
            *   If the projection intersects the silhouette in **all** views, the node is potentially inside.
            *   If the projection does **not** intersect the silhouette in **any** view, classify as `OUTSIDE` and prune (do not create children).
            *   If the projection intersects the silhouette in **some but not all** views, or if the intersection is partial/ambiguous, classify as `INTERSECTION`.
        d.  **Subdivide:** If status is `INTERSECTION` and the node size is above a minimum resolution threshold, split the cube into 8 octants (children). Set children status to `INSIDE` initially. Recursively process each child.
        e.  **Leaf:** If status is `INSIDE` or `OUTSIDE`, or if minimum resolution is reached, stop subdivision.

4.  **Post-processing (Optional):**
    *   Apply a Gaussian blur with $\sigma_{vol}$ to the resulting binary volume representation to smooth the hull [S2].

5.  **Surface Extraction:**
    *   Extract the surface from the final octree using the Marching Cubes (MC) algorithm [S1].
    *   MC estimates surface triangles from `INTERSECTION` octants. The location of triangles is determined by the configuration of inside vertices [S1].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def construct_visual_hull(images, masks, projection_matrices, initial_bbox, min_size):
    """
    images: List of 2D images (not strictly needed if masks are precomputed)
    masks: List of binary 2D masks (1 for object, 0 for background)
    projection_matrices: List of 3x4 projection matrices P
    initial_bbox: Bounding box (min_x, min_y, min_z, max_x, max_y, max_z)
    min_size: Minimum edge length for octree leaves
    """
    root = OctreeNode(bbox=initial_bbox, status='INSIDE')
    queue = [root]
    
    while queue:
        node = queue.pop(0)
        
        # Project node's bounding box to 2D for all views
        projections = []
        for P in projection_matrices:
            proj_2d = project_bbox_3d_to_2d(node.bbox, P)
            projections.append(proj_2d)
            
        # Classify node based on intersection with masks
        status = classify_node(projections, masks)
        node.status = status
        
        if status == 'OUTSIDE':
            continue
        elif status == 'INSIDE':
            continue
        else: # INTERSECTION
            if is_leaf(node.bbox, min_size):
                continue
            else:
                # Split into 8 octants
                children = split_octant(node.bbox)
                for child in children:
                    child.parent = node
                    child.status = 'INSIDE' # Initial assumption
                    queue.append(child)
                    
    return root

def project_bbox_3d_to_2d(bbox, P):
    """
    Projects the 8 corners of a 3D bounding box to 2D image plane.
    Returns the 2D bounding box (min_u, min_v, max_u, max_v) enclosing the projected points.
    """
    corners = get_bbox_corners(bbox)
    projected_points = []
    for corner in corners:
        # Homogeneous coordinates
        point_h = [corner[0], corner[1], corner[2], 1.0]
        # Project
        proj_h = matrix_multiply(P, point_h)
        # Perspective division
        if proj_h[2] != 0:
            u = proj_h[0] / proj_h[2]
            v = proj_h[1] / proj_h[2]
            projected_points.append((u, v))
        else:
            # Handle points at infinity (should not happen for valid bbox)
            pass
            
    if not projected_points:
        return None
        
    min_u = min(p[0] for p in projected_points)
    min_v = min(p[1] for p in projected_points)
    max_u = max(p[0] for p in projected_points)
    max_v = max(p[1] for p in projected_points)
    
    return (min_u, min_v, max_u, max_v)

def classify_node(projections, masks):
    """
    Checks if the projected bounding box intersects the silhouette mask in all views.
    Returns 'INSIDE', 'OUTSIDE', or 'INTERSECTION'.
    """
    all_intersect = True
    any_intersect = False
    
    for proj, mask in zip(projections, masks):
        # Check if the 2D bounding box overlaps with the silhouette
        # A simple check: sample center or check corners. 
        # Robust check: rasterize the projected bbox and check overlap with mask.
        # For simplicity, we assume a point-in-polygon or rasterization check exists.
        intersects = check_2d_intersection(proj, mask)
        
        if intersects:
            any_intersect = True
        else:
            all_intersect = False
            
    if all_intersect:
        return 'INSIDE'
    elif any_intersect:
        return 'INTERSECTION'
    else:
        return 'OUTSIDE'

def check_2d_intersection(proj_2d_bbox, mask):
    """
    Checks if the 2D bounding box intersects the binary mask.
    Returns True if any pixel in the bbox area is 1 in the mask.
    """
    min_u, min_v, max_u, max_v = proj_2d_bbox
    # Iterate over pixels in the bbox range
    for u in range(int(min_u), int(max_u) + 1):
        for v in range(int(min_v), int(max_v) + 1):
            if 0 <= u < mask.width and 0 <= v < mask.height:
                if mask[v, u] == 1:
                    return True
    return False

def split_octant(bbox):
    """
    Splits a 3D bounding box into 8 smaller octants.
    Returns list of 8 new BoundingBox objects.
    """
    min_x, min_y, min_z, max_x, max_y, max_z = bbox
    mid_x = (min_x + max_x) / 2
    mid_y = (min_y + max_y) / 2
    mid_z = (min_z + max_z) / 2
    
    octants = []
    for i in range(2):
        for j in range(2):
            for k in range(2):
                new_min = [min_x + i*(mid_x-min_x), min_y + j*(mid_y-min_y), min_z + k*(mid_z-min_z)]
                new_max = [mid_x + i*(max_x-mid_x), mid_y + j*(max_y-mid_y), mid_z + k*(max_z-mid_z)]
                # Simplified logic for new min/max based on octant index
                # Correct logic:
                x_start = min_x if i == 0 else mid_x
                x_end = mid_x if i == 0 else max_x
                y_start = min_y if j == 0 else mid_y
                y_end = mid_y if j == 0 else max_y
                z_start = min_z if k == 0 else mid_z
                z_end = mid_z if k == 0 else max_z
                
                octants.append(BoundingBox(x_start, y_start, z_start, x_end, y_end, z_end))
    return octants

def is_leaf(bbox, min_size):
    """
    Checks if the bounding box is smaller than min_size.
    """
    min_x, min_y, min_z, max_x, max_y, max_z = bbox
    size_x = max_x - min_x
    size_y = max_y - min_y
    size_z = max_z - min_z
    return size_x < min_size or size_y < min_size or size_z < min_size
```

**Ambiguity and Pitfalls:**
*   **Inside Vertex Identification:** Ambiguity arises when only part of an octant edge is within a silhouette (case b) or the silhouette is entirely within an octant (case c) [S1]. These cases frequently arise from imperfect silhouettes, projection matrix errors, or insufficient octree resolution [S1].
*   **Projection Errors:** Errors in $\theta$ or wobbly rotational axes propagate to surface construction errors [S1].
*   **Noise Robustness:** Octree construction is robust against image noise because an octant is only removed if no nonzero-valued pixels are found in its projection [S1]. However, simple thresholding preserves non-convex details but degrades the subsequent Marching Cubes surface [S1].

## Operating characteristics
*   **Time Complexity:** Depends on the depth of the octree and the number of views. Each node requires projection and intersection tests for all views. Complexity is roughly $O(N \cdot V)$ where $N$ is the number of views and $V$ is the number of voxels/nodes in the octree.
*   **Memory Complexity:** Proportional to the number of nodes in the octree. Sparse octrees save memory compared to dense voxel grids.
*   **Strengths:** Robust against image noise; simple thresholding preserves non-convex details during construction [S1]. Efficient representation for large volumes with sparse object presence.
*   **Limitations:** Surface quality depends on octree resolution and the accuracy of silhouette extraction. Marching Cubes on octree leaves can produce degraded surfaces if silhouettes are imperfect [S1]. Single-view reconstruction requires assumptions (symmetry) which may not hold for arbitrary shapes [S2].
*   **When to Use:** When multiple views are available and computational efficiency is prioritized over high-fidelity surface detail. Suitable for objects with clear silhouettes.
*   **When Not to Use:** When high-fidelity surface reconstruction is required from imperfect silhouettes without post-processing. Not suitable for single-view reconstruction of asymmetric objects without auxiliary assumptions [S2].

## Background
The concept of the 3D visual hull was introduced as the intersection of multiple 3D cones created by back-projecting 2D silhouettes [S1]. This technique is part of the broader Shape from Silhouette (SfS) family [S1]. The use of octrees for representing visual hulls became widely adopted due to their efficiency in handling sparse 3D data [S1]. The Marching Cubes algorithm, originally developed for medical imaging, is commonly used for surface extraction from these octrees [S1]. Recent applications include fluid reconstruction, where visual hulls guide density estimation [S2].

## Sources
[S1] Local hull-based surface construction of volumetric data from silhouettes, https://doi.org/10.1109/tip.2008.926149, fetched 2026-10-30
[S2] Global Transport for Fluid Reconstruction with Learned Self-Supervision, https://doi.org/10.1109/cvpr46437.2021.00168, fetched 2026-10-30
