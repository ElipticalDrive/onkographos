# Silhouette-Based Bounding Box Estimation

Also called: Visual Hull, shape-from-silhouette

## Overview
Silhouette-Based Bounding Box Estimation computes a 3D volume that strictly encloses a target object by intersecting the back-projected foreground regions from multiple binary segmentation masks [S1, S2, unofficial]. It differs from [Octree-based Bounding Volume Computation](6270-octree-based-bounding-volume-computation.md) by relying solely on 2D image masks rather than hierarchical spatial partitioning, and from [Silhouette-based Bounding Box Initialization](5448-silhouette-based-bounding-box-initialization.md) by explicitly constructing the full volumetric hull rather than just an initial box [S1, S2, unofficial].

## How it works
**Inputs:**
*   `train_cameras`: Array of camera poses (extrinsics) and intrinsics for $N$ training views [S2, unofficial].
*   `masks`: Array of binary foreground-background segmentation masks, one per view [S1, S2, unofficial].
*   `silhouetteRadius`: Scalar expansion radius for the final bounds [S2, unofficial].
*   `skipIndex`/`skips`: Optional subsampling indices/strides for views [S2, unofficial].

**Outputs:**
*   `scene.bbox_depth_min/max`, `scene.bbox_width_min/max`, `scene.bbox_height_min/max`: The final axis-aligned bounding box coordinates [S2, unofficial].

**Data Structures:**
*   **Mask**: Binary array of size $H \times W$ (or $W \times H$), where 1 indicates foreground and 0 indicates background [S1, S2, unofficial].
*   **Camera Pose**: $4 \times 4$ matrix $P$ mapping world coordinates to camera coordinates (or inverse, depending on convention; S2 implies $c2w$ or similar transformation logic) [S2, unofficial].
*   **Voxel Grid**: Optional intermediate representation. S1 uses a $400 \times 400 \times 400$ voxel grid to store the hull [S1, unofficial].

**Steps:**
1.  **Initialize Bounds**: Set global minimums ($x_{min}, y_{min}, z_{min}$) to $+\infty$ and maximums ($x_{max}, y_{max}, z_{max}$) to $-\infty$ [S2, unofficial].
2.  **Iterate Views**: Loop through each training view $i$ from $0$ to $N-1$, applying `skipIndex`/`skips` if provided [S2, unofficial].
3.  **Extract Mask**: Retrieve the binary mask $M_i$ for view $i$ [S1, S2, unofficial].
    *   *Source Note*: Masks are derived from alpha channels if available, or via background separation (e.g., OpenCV for white backgrounds) or segmentation models [S1, unofficial].
4.  **Backproject Silhouette**: For each pixel $(u, v)$ in $M_i$ where $M_i[u, v] == 1$:
    *   Construct the ray $r$ originating from the camera center $C_i$ and passing through the pixel $(u, v)$ using camera intrinsics [S1, S2, unofficial].
    *   The ray represents a cone of possible object locations. The intersection of all such rays from all foreground pixels forms the visual hull [S1, unofficial].
5.  **Accumulate Volume**:
    *   *Method A (Voxel-based)*: Cast rays into a coarse voxel grid (e.g., $400^3$). Mark voxels intersected by any foreground ray as occupied [S1, unofficial].
    *   *Method B (Direct Bounds)*: Project the silhouette edges or extreme points of the foreground mask into 3D space to update the global min/max bounds [S2, unofficial].
6.  **Compute Min/Max**: Determine the bounding box enclosing all occupied voxels or projected points along the X, Y, and Z axes [S2, unofficial].
7.  **Expand Bounds**: Apply `silhouetteRadius` to the computed bounds to account for discretization errors and ensure the object is fully contained [S2, unofficial].
8.  **Store**: Write the final bounds to `scene.bbox_depth_min/max`, `scene.bbox_width_min/max`, `scene.bbox_height_min/max` [S2, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_silhouette_bbox(train_cameras, masks, silhouette_radius, skip_index=None):
    """
    Computes a bounding box enclosing the visual hull of an object.
    
    Args:
        train_cameras: List of camera objects with 'intrinsics' and 'extrinsics' (world-to-camera or camera-to-world).
        masks: List of binary numpy arrays (H, W) for each view.
        silhouette_radius: Float, expansion factor for the final bounds.
        skip_index: Optional list of indices to subsample views.
        
    Returns:
        dict: {'bbox_min': [x, y, z], 'bbox_max': [x, y, z]}
    """
    if skip_index is not None:
        cameras = [train_cameras[i] for i in skip_index]
        view_masks = [masks[i] for i in skip_index]
    else:
        cameras = train_cameras
        view_masks = masks

    # Initialize global bounds
    # Note: Exact coordinate system (left-handed/right-handed) depends on camera convention.
    # Assuming standard OpenGL-like or similar where we track min/max in world space.
    global_min = [float('inf'), float('inf'), float('inf')]
    global_max = [float('-inf'), float('-inf'), float('-inf')]

    for cam, mask in zip(cameras, view_masks):
        # Get camera parameters
        # K: 3x3 intrinsic matrix
        # R, t: Rotation and Translation from world to camera (or inverse)
        K = cam.intrinsics
        # Assuming cam.extrinsics provides the transformation from World to Camera
        # If it provides Camera to World, invert it.
        # For silhouette backprojection, we need rays in World space.
        # Ray direction in camera space: d_cam = K^-1 * [u, v, 1]
        # Ray direction in world space: d_world = R^T * d_cam
        # Ray origin in world space: O_world = -R^T * t (if t is world-to-camera translation)
        
        # Extract mask foreground pixels
        # Optimization: Only process boundary pixels or sample densely?
        # S1 implies full backprojection or voxel marking. S2 implies accumulating points.
        # We will implement the direct bound update approach for efficiency, 
        # approximating the hull by the extreme rays.
        
        # Find foreground pixels
        ys, xs = np.where(mask > 0)
        
        if len(xs) == 0:
            continue
            
        # Convert pixel coordinates to normalized device coordinates or similar
        # Assuming K maps [u, v, 1] to ray direction in camera space
        # u, v are pixel coordinates.
        
        # Vectorize operations for speed
        # Points in camera space: [u, v, 1]
        # Inverse intrinsics to get direction
        # Note: K usually includes focal length and principal point.
        # d_cam = inv(K) @ [u, v, 1]
        
        # To ensure we capture the *entire* volume, we must consider the extreme rays.
        # The visual hull is the intersection of back-projected cones.
        # The bounding box of the visual hull is determined by the extreme rays 
        # from the camera center through the silhouette boundary.
        
        # For simplicity and robustness (S1's voxel approach is safer for non-convexity 
        # but S2's bbox approach suggests direct min/max).
        # We will use the voxel approach logic conceptually but output bounds.
        
        # Let's use the extreme points of the mask to define the bounding frustum slice.
        # However, the visual hull is the intersection of ALL foreground rays.
        # The tightest AABB containing the intersection of cones is found by 
        # intersecting the AABBs of each cone? No, intersection of cones is complex.
        # S2 says: "Accumulate 3D points/volumes. Compute min/max bounds".
        # This implies sampling or voxelization.
        
        # Implementation of Voxel-based accumulation (S1 style) for correctness:
        # Define a coarse voxel grid covering the scene.
        # Mark voxels intersected by any foreground ray.
        # Then compute AABB of occupied voxels.
        
        # Since we don't have a pre-defined grid in the function signature, 
        # we estimate the grid bounds from the camera frustums or previous steps.
        # For this self-contained function, we assume a global scene bounds 
        # or we compute the union of all camera frustums first.
        
        # Alternative: Direct Ray Casting for Bounds
        # For each foreground pixel, the ray is a line. The visual hull is the 
        # intersection of all half-spaces/cones.
        # The bounding box of the intersection of convex cones is convex.
        # We can approximate the hull by sampling points along the rays.
        
        # Let's stick to the S2 description: "Backproject mask pixels... Accumulate... Compute min/max"
        # We will sample a set of points along each foreground ray and update global bounds.
        
        # To be precise and avoid infinite rays, we need a max depth or scene bounds.
        # S1 uses a fixed 400x400x400 voxel grid. We will assume a normalized 
        # or pre-computed scene bounding box for ray clipping.
        
        # For this implementation, we will return the bounds of the union of 
        # back-projected silhouettes, expanded by radius.
        
        # Get extreme rays for this view
        # The silhouette boundary defines the cone.
        # We can compute the AABB of the cone from this view.
        # Cone origin: Camera Center (World Space)
        # Cone direction: Normal to the silhouette edge? No, it's a set of rays.
        
        # Simplified robust method:
        # 1. Get Camera Center in World Space (C_w).
        # 2. For each foreground pixel, compute Ray (O, D).
        # 3. The visual hull is inside this ray.
        # 4. The intersection of all such rays is the hull.
        # 5. The AABB of the hull is the intersection of the AABBs of all rays? 
        #    No, AABB of intersection is not intersection of AABBs.
        #    However, the AABB of the intersection is contained within the AABB of the union.
        #    S2 says "tightest volume containing all visible geometry".
        #    This is the Visual Hull.
        
        # Correct approach for AABB of Visual Hull:
        # The Visual Hull is a convex polyhedron (if the object is convex) or a 
        # non-convex shape bounded by planar faces (from the silhouettes).
        # Actually, the visual hull is the intersection of back-projected silhouettes.
        # Each silhouette back-projects to a cone (or pyramid).
        # The intersection of pyramids is a convex polyhedron? 
        # No, the intersection of convex sets is convex. A pyramid is convex.
        # So the Visual Hull is always convex? 
        # S1 says: "inability to correctly restore dented parts of non-convex shapes".
        # This implies the Visual Hull is the *convex hull* of the object? 
        # No, it's the intersection of view cones. It is a convex approximation.
        # It contains the object but may include extra volume (overcutting? No, S1 says "contains 100% of the object" and "never overcuts" in the sense of excluding object parts, but it "overcuts" in the sense of including empty space? 
        # S1: "never overcuts (contains 100% of the object)". This phrasing is tricky. 
        # Usually "overcut" means removing too much. "Undercut" means leaving too much.
        # S1 says it "cannot correctly restore dented parts" -> it fills dents -> it includes empty space.
        # So it is a conservative estimate (superset).
        
        # Therefore, the AABB of the Visual Hull is the AABB of the intersection of all view cones.
        # To compute this AABB:
        # We can sample points on the boundary of the intersection.
        # Or, simpler: The Visual Hull is contained within the AABB of the union of all camera centers and the extreme rays?
        # No.
        
        # Let's use the Voxel approach (S1) as it is robust and cited.
        # We need a voxel grid. We will define a grid based on the union of all camera frustums 
        # or a fixed large volume.
        
        # For the purpose of this algorithm description, we assume a global `voxel_grid` 
        # is maintained or passed. If not, we compute a coarse grid.
        
        # Since we cannot pass a grid in the signature easily without state, 
        # we will compute the bounds by intersecting the AABBs of the back-projected cones?
        # No, intersection of AABBs is not the AABB of the intersection.
        
        # Correct Algorithm for AABB of Intersection of Cones:
        # 1. Initialize AABB to infinity.
        # 2. For each view, the back-projected silhouette forms a cone.
        # 3. The Visual Hull is the intersection of all cones.
        # 4. The AABB of the intersection is the intersection of the AABBs of the cones?
        #    Let V = Intersection(Cone_i).
        #    AABB(V) = Intersection(AABB(Cone_i))?
        #    Yes, for any set of points P, if P is in Intersection(C_i), then P is in C_i for all i.
        #    Thus P is in AABB(C_i) for all i.
        #    Thus P is in Intersection(AABB(C_i)).
        #    So AABB(Intersection(C_i)) is a subset of Intersection(AABB(C_i)).
        #    Is it equal? Not necessarily. The intersection of AABBs might be larger than the AABB of the intersection.
        #    However, S2 says "tightest volume".
        #    S1 uses voxels.
        
        # We will implement the Voxel-based method for accuracy.
        
        pass # Placeholder for voxel logic below

    # Re-implementing with Voxel Logic for correctness
    # We need to define the voxel grid.
    # Assume we have a global bounding box for the scene or we compute it from cameras.
    # For this function, we will return the bounds of the visual hull.
    
    # Let's assume a coarse voxel grid of resolution 400x400x400 (S1) 
    # covering a unit cube or scene-specific bounds.
    # Since scene bounds are not known a priori in this pure function, 
    # we will compute the bounds of the *union* of back-projected silhouettes 
    # and then refine? No.
    
    # Let's go with the S2 "Accumulate 3D points" method, but sample the silhouette boundary.
    
    final_min = [float('inf')] * 3
    final_max = [float('-inf')] * 3
    
    # We need a max depth to clip rays. 
    # Assume a large enough depth or compute from camera distances.
    MAX_DEPTH = 100.0 
    
    for cam, mask in zip(cameras, view_masks):
        # Get Camera Center in World Space
        # If extrinsics are World->Camera (R, t), then Camera Center C_w = -R^T t
        R = cam.extrinsics[:3, :3]
        t = cam.extrinsics[:3, 3]
        C_w = -R.T @ t
        
        # Get Intrinsics K
        K = cam.intrinsics
        
        # Find foreground pixels
        ys, xs = np.where(mask > 0)
        
        if len(xs) == 0:
            continue
            
        # Convert to homogeneous coordinates in camera space
        # u, v
        u = xs.astype(float)
        v = ys.astype(float)
        
        # Ray directions in camera space
        # d_cam = inv(K) @ [u, v, 1]
        # We can compute inv(K) once per camera
        inv_K = np.linalg.inv(K)
        
        # Create homogeneous vectors
        pts_cam = np.vstack([u, v, np.ones_like(u)])
        d_cam = inv_K @ pts_cam # Shape (3, N)
        
        # Transform to World Space
        # d_world = R^T @ d_cam
        d_world = R.T @ d_cam # Shape (3, N)
        
        # Normalize directions
        norms = np.linalg.norm(d_world, axis=0)
        d_world = d_world / norms
        
        # Sample points along rays to approximate the hull boundary
        # The visual hull is the intersection. 
        # To find the AABB of the intersection, we can sample points on the 
        # "inner" boundary of each cone and intersect?
        # Actually, the Visual Hull is the set of points visible from ALL cameras.
        # A point P is in the Visual Hull if for all i, P projects to foreground in i.
        
        # We can estimate the AABB by finding the min/max of the intersection of the cones.
        # This is hard analytically.
        # S1 uses voxels. We will simulate the voxel check.
        
        # For each ray, we know the object is somewhere along it.
        # The Visual Hull is the intersection of all such "tubes".
        # The AABB of the intersection is bounded by the extreme rays.
        
        # Let's sample a set of points along each ray and keep only those 
        # that are "inside" all other views? No, that's reconstruction.
        
        # S2 says: "Accumulate 3D points/volumes. Compute min/max bounds."
        # This implies we just take the union of all back-projected rays?
        # No, "intersecting the possible areas".
        
        # If we intersect the areas, the resulting volume is smaller.
        # The AABB of the intersection is contained in the AABB of the union.
        # S2 likely approximates the Visual Hull by the AABB of the intersection 
        # of the bounding boxes of each back-projected cone.
        
        # Let's compute the AABB of each back-projected cone (frustum slice).
        # Cone i: Origin C_i, Direction D_i (set of rays).
        # The cone is bounded by the silhouette.
        # The AABB of Cone i can be computed by projecting the silhouette vertices 
        # to 3D and finding min/max.
        
        # Get silhouette boundary pixels (simplified: use all foreground pixels)
        # Project each foreground pixel to 3D at a fixed depth or using ray intersection with a plane?
        # No, the ray is a line. The "volume" is the cone.
        # The AABB of a cone is defined by the origin and the extreme rays.
        
        # For each view, compute the AABB of the cone formed by the foreground rays.
        # This AABB is an over-approximation of the visual hull slice.
        # Intersecting these AABBs gives a conservative bound.
        
        # Compute extreme rays for this view
        # The silhouette is the boundary of the foreground.
        # We can find the convex hull of the foreground pixels in 2D, 
        # then back-project those vertices to 3D to form a pyramid.
        
        # Get 2D convex hull of foreground pixels
        points_2d = np.column_stack([u, v])
        from scipy.spatial import ConvexHull
        hull = ConvexHull(points_2d)
        hull_indices = hull.vertices
        hull_pts_2d = points_2d[hull_indices]
        
        # Back-project hull vertices to 3D rays
        # We need a depth to form a pyramid. 
        # The visual hull extends from the camera to infinity (or scene bounds).
        # We can clip at a max depth.
        
        # Project hull vertices to 3D at depth 1 (direction) and depth MAX_DEPTH
        # Ray: O + t * D
        # At t=1: P1 = C_w + D
        # At t=MAX_DEPTH: P2 = C_w + MAX_DEPTH * D
        
        # Compute P1 and P2 for all hull vertices
        # d_world is (3, N_hull)
        P1 = C_w[:, np.newaxis] + d_world[:, hull_indices] # (3, N_hull)
        P2 = C_w[:, np.newaxis] + MAX_DEPTH * d_world[:, hull_indices] # (3, N_hull)
        
        # Combine P1 and P2 to form the pyramid vertices
        pyramid_pts = np.hstack([P1, P2]) # (3, 2*N_hull)
        
        # Compute AABB of this pyramid
        min_x = np.min(pyramid_pts[0])
        max_x = np.max(pyramid_pts[0])
        min_y = np.min(pyramid_pts[1])
        max_y = np.max(pyramid_pts[1])
        min_z = np.min(pyramid_pts[2])
        max_z = np.max(pyramid_pts[2])
        
        # Intersect with global bounds
        final_min[0] = min(final_min[0], min_x)
        final_max[0] = min(final_max[0], max_x) # Wait, intersection of AABBs?
        # If V = Intersection(Cone_i), then AABB(V) subset Intersection(AABB(Cone_i)).
        # So we intersect the AABBs.
        
        final_min[0] = max(final_min[0], min_x) # Intersection: max of mins
        final_max[0] = min(final_max[0], max_x) # Intersection: min of maxs
        final_min[1] = max(final_min[1], min_y)
        final_max[1] = min(final_max[1], max_y)
        final_min[2] = max(final_min[2], min_z)
        final_max[2] = min(final_max[2], max_z)

    # Apply silhouette radius expansion
    final_min = [m - silhouette_radius for m in final_min]
    final_max = [m + silhouette_radius for m in final_max]
    
    return {'bbox_min': final_min, 'bbox_max': final_max}
```

**Parameters:**
*   `silhouetteRadius`: Expansion radius to ensure the object is fully contained despite discretization [S2, unofficial]. Typical value: Not given in the sources.
*   `skipIndex`: Indices of views to skip for subsampling [S2, unofficial].
*   `voxelResolution`: S1 uses $400 \times 400 \times 400$ [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Non-Convex Shapes**: The visual hull cannot restore dented parts (undercuts) of non-convex shapes, resulting in a volume that contains the object but also includes empty space (overcutting in terms of volume, but undercuts in terms of detail) [S1, unofficial].
*   **Mask Quality**: Requires accurate binary foreground-background segmentation. Errors in mask generation (e.g., noise, incomplete background separation) directly corrupt the hull [S1, unofficial].
*   **Precision**: Discretization in voxel grids or ray sampling can lead to slight inaccuracies in the bounds [S1, unofficial].

## Operating characteristics
*   **Time Complexity**: $O(N \cdot P)$, where $N$ is the number of views and $P$ is the number of foreground pixels or voxels processed [S1, S2, unofficial]. S1 notes it can fit 100 images in about 10 seconds using JAX [S1, unofficial].
*   **Memory Complexity**: $O(V)$ for voxel-based implementation, where $V$ is the number of voxels (e.g., $400^3$) [S1, unofficial].
*   **Performance**: Provides a 2-8x speedup in NeRF training by eliminating unnecessary network evaluations outside the hull [S1, unofficial].
*   **Strengths**: Fast, simple, guarantees the object is fully contained (100% recall) [S1, unofficial].
*   **Limitations**: Cannot represent concavities; requires accurate segmentation masks [S1, unofficial].

## Background
*   **Origin**: Visual Hull is a classic 3D reconstruction technique based on shape-from-silhouette [S1, S2, unofficial].
*   **History**: It predates NeRF and has been applied in many classic 3D reconstruction contexts [S1, unofficial].
*   **Usage**: Recently revisited in VaxNeRF to accelerate NeRF training by providing a coarse in-out field separation [S1, unofficial].

## Sources
[S1] VaxNeRF: Revisiting the Classic for Voxel-Accelerated Neural Radiance Field, https://doi.org/10.48550/arxiv.2111.13112, fetched 2026-10-26 (unofficial)
[S2] nortonBryan/Clear-plenoxels, https://github.com/nortonBryan/Clear-plenoxels, fetched 2026-10-27 (unofficial)
