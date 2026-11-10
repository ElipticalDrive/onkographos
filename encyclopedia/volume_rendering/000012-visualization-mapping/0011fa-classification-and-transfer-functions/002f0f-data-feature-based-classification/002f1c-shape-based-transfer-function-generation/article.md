# Shape-based Transfer Function Generation

Also called: Shape classification, Shape-based TF (Skeleton), Shape-based Transfer Functions, Curve-Skeleton Shape Classification TF, Shape classification for volume visualization, Skeleton-based shape descriptor classification

## Overview
Shape-based Transfer Function Generation classifies volumetric features based on their 3D geometric shape rather than scalar intensity or gradient magnitude, solving the problem of distinguishing structures with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) [S3]. Unlike per-voxel classification methods that rely on local filters, this method assigns a uniform shape descriptor to each entire volumetric feature (segment) derived from curve-skeletons, allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels [S3]. It distinguishes itself from siblings like [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) by operating on segmented regions and skeletons rather than local voxel neighborhoods, and from [High-Dimensional Classification with Color Data](6327-high-dimensional-classification-with-color-data.md) by using explicit geometric descriptors (tubiness, surfaceness, blobbiness) instead of statistical or learned feature vectors.

## How it works
The method classifies volumetric data into three shape classes: longitudinal/tubular, surface-like, and blobby [S3]. It operates on pre-segmented regions, ignoring background voxels [S3].

**Inputs and Outputs:**
*   **Input:** Volume data $V$, pre-segmentation mask $M$ (binary volume specifying surface voxels of objects of interest).
*   **Output:** For each voxel in the pre-segmentation, a shape descriptor triple $(\tau, surf, blob)$ representing membership in tubiness, surfaceness, and blobbiness classes. Background voxels have descriptor $(0, 0, 0)$ [S3].

**Data Structures:**
*   **Pre-segmentation Mask:** Binary volume $M \in \{0, 1\}^{N_x \times N_y \times N_z}$.
*   **Skeleton Segments:** List of polygonal chains. Each segment is a list of 3D points $(x, y, z)$.
*   **Skeleton Regions:** Partition of the pre-segmented volume. Each region $R$ is a set of voxel coordinates $\{\vec{r}\}$ associated with a specific skeleton segment.
*   **Distance Map:** Volume $D$ where $D(\vec{v})$ is the minimal distance from voxel $\vec{v}$ to the nearest skeleton point in its region.

**Parameters:**
*   **Field Strength:** Low value (e.g., 4) for potential field method to minimize skeleton complexity [S3].
*   **High Divergence Points:** Percentage (default 20%) used in skeleton computation [S3].
*   **Segment Length Threshold:** 5–10 voxels for splitting skeletons in datasets of size $256^3$ to $512^3$ [S3].

**Steps:**

1.  **Pre-segmentation:** Perform rough threshold-based pre-segmentation (windowing) to obtain surface information. This can be automatic if intensity ranges are known (e.g., Hounsfield scale) [S3]. Add extra layers of voxels at the surface of objects to handle cavities caused by noise/artifacts, which may smooth objects and cause unwanted merging of adjacent features [S3].
2.  **Skeleton Computation:** Compute curve-skeletons for each volumetric structure using the potential field method (e.g., Cornea’s pfSkel) [S3]. Input is the binary pre-segmentation. Output is multiple skeleton segments (polygonal chains) [S3].
3.  **Skeleton Normalization:**
    *   **Gap Closing:** Connect end nodes of the skeleton graph with the next neighboring skeleton point outside the end node's segment if the connecting line lies completely within the pre-segmented object [S3].
    *   **Splitting:** Split curve-skeleton segments not exceeding the length threshold (5–10 voxels) [S3].
    *   **Pruning:** Discard degenerated skeleton branches shorter than the length threshold [S3].
4.  **Decomposition:** Divide the pre-segmented volume into 'skeleton regions' $R$, where each region corresponds to one segment of the curve-skeleton. Compute the distance map $D$ for each region [S3].
5.  **Merging:** Merge adjacent skeleton regions based on criteria to improve classifiability, as ad-hoc decomposition may not yield meaningful shapes [S3].
6.  **Shape Analysis:** Compute shape scores for each merged region $R$:
    *   **Tubiness $\tau(R)$:**
        $$ \tau(R) := \frac{1}{\max\left(\sqrt{\frac{1}{|\partial R|} \sum_{\vec{v} \in \partial R} (D(\vec{v}) - \bar{d})^2}, 1\right)} \in [0; 1] $$
        where $\partial R$ is the set of surface voxels of $R$, $D$ is the distance map, and $\bar{d}$ is the mean skeleton distance of all $\vec{v} \in \partial R$ [S3].
    *   **Planarity $plan(R)$:**
        $$ plan(R) := \text{clamp}\left(\frac{1}{5} \frac{b_m(R)}{b_s(R)} - 1, 0, 1\right) \in [0; 1] $$
        where $b_m(R)$ is the second shortest side and $b_s(R)$ is the shortest side of the minimal oriented bounding box of $R$. Regions with $b_m \ge 10 \cdot b_s$ are maximal flat; $b_m \le 5 \cdot b_s$ are not flat [S3].
    *   **Convexity $conv(R)$:** Select an arbitrary set of pairs of surface points. For each line segment, determine the fraction that runs inside the shape. $conv(R)$ is the average of these fractions [S3].
    *   **Surfaceness $surf(R)$:**
        $$ surf(R) := (plan(R) + conv(R)) \cdot (1 - \tau(R)) \in [0; 2] $$
        Weighted by inverse tubiness to avoid misclassifying curved longitudinal shapes [S3].
    *   **Blobbiness $blob(R)$:**
        $$ blob(R) := \frac{\sigma^2_{sph}(vol)}{\sigma^2(R)} \in [0; 1] $$
        where $\sigma^2(R)$ is the variance of the region and $\sigma^2_{sph}(vol)$ is the variance of a sphere with the same volume [S3].
        *   Variance $\sigma^2(R)$:
            $$ \sigma^2(R) := \sum_{\vec{r} \in R} |\vec{r} - m|^2 $$
            where $m = \frac{1}{|R|} \sum_{\vec{r} \in R} \vec{r}$ is the center of mass [S3].
        *   Sphere Variance $\sigma^2_{sph}(vol)$:
            $$ \sigma^2_{sph}(vol) = \frac{4}{5} \pi \left(\frac{3}{4\pi} vol\right)^{5/3} $$
            where $vol = |R|$ (assuming unit voxel volume) [S3].
7.  **Transfer Function Specification:** Use the shape scores to allow the user to assign optical properties (color, opacity) to the shape classes via a continuous triangle-shaped plot [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def compute_shape_descriptor(region_voxels, skeleton_points, pre_seg_mask):
    """
    Computes tubiness, surfaceness, and blobbiness for a merged skeleton region.
    
    Args:
        region_voxels: List of (x,y,z) tuples representing voxels in the region R.
        skeleton_points: List of (x,y,z) tuples representing the skeleton segment for R.
        pre_seg_mask: 3D array or function to check if a voxel is in the pre-segmentation.
        
    Returns:
        Tuple (tubiness, surfaceness, blobbiness)
    """
    if not region_voxels:
        return (0.0, 0.0, 0.0)

    # 1. Compute Distance Map for the region
    # D(v) is minimal distance from v to any point in skeleton_points
    # For implementation, compute Euclidean distance to nearest skeleton point
    distances = []
    surface_voxels = []
    
    # Identify surface voxels: voxels in region that have at least one neighbor outside region
    # Or simply use all voxels if D is defined for all, but formula uses surface voxels for tubiness
    # The source defines D as distance map from skeleton region growing. 
    # We approximate D(v) as min_dist(v, skeleton_points)
    
    for v in region_voxels:
        min_dist = float('inf')
        for s in skeleton_points:
            d = math.sqrt((v[0]-s[0])**2 + (v[1]-s[1])**2 + (v[2]-s[2])**2)
            if d < min_dist:
                min_dist = d
        distances.append(min_dist)
        
        # Identify surface voxels (simplified: voxels on the boundary of the region)
        # In practice, this requires checking neighbors against pre_seg_mask or region set
        # For this pseudocode, we assume a helper function is_available
        if is_surface_voxel(v, region_voxels, pre_seg_mask):
            surface_voxels.append(v)

    if not surface_voxels:
        # Fallback if no surface voxels detected, use all voxels for mean dist calculation
        # But tubiness specifically uses surface voxels. 
        # If no surface, tubiness is undefined or 0.
        pass

    # 2. Compute Tubiness
    if surface_voxels:
        surface_distances = [distances[region_voxels.index(v)] for v in surface_voxels]
        mean_d = sum(surface_distances) / len(surface_distances)
        variance_d = sum((d - mean_d)**2 for d in surface_distances) / len(surface_distances)
        std_d = math.sqrt(variance_d)
        tubiness = 1.0 / max(std_d, 1.0)
    else:
        tubiness = 0.0

    # 3. Compute Planarity
    # Compute minimal oriented bounding box (OOBB)
    # This is complex; using axis-aligned bounding box (AABB) as a proxy if OOBB library not available
    # Source uses Har-Peled's technique for OOBB. 
    # Here we compute AABB extents for simplicity in pseudocode, noting it's an approximation.
    xs = [v[0] for v in region_voxels]
    ys = [v[1] for v in region_voxels]
    zs = [v[2] for v in region_voxels]
    
    min_x, max_x = min(xs), max(xs)
    min_y, max_y = min(ys), max(ys)
    min_z, max_z = min(zs), max(zs)
    
    extents = [max_x - min_x, max_y - min_y, max_z - min_z]
    extents.sort()
    bs = extents[0] # Shortest side
    bm = extents[1] # Second shortest side
    
    if bs == 0:
        planarity = 0.0
    else:
        ratio = bm / bs
        planarity = clamp((1.0/5.0) * ratio - 1.0, 0.0, 1.0)

    # 4. Compute Convexity
    # Select arbitrary pairs of surface points
    # For efficiency, sample a subset of surface voxels
    num_pairs = 100
    if len(surface_voxels) < 2:
        convexity = 0.0
    else:
        inside_count = 0
        total_count = 0
        # Simple sampling
        sampled_points = surface_voxels[:min(len(surface_voxels), 20)]
        for i in range(len(sampled_points)):
            for j in range(i + 1, len(sampled_points)):
                p1 = sampled_points[i]
                p2 = sampled_points[j]
                # Check fraction of line segment inside shape
                # Approximate by checking midpoints
                steps = 10
                inside_steps = 0
                for k in range(1, steps):
                    t = k / steps
                    mid_x = p1[0] + t * (p2[0] - p1[0])
                    mid_y = p1[1] + t * (p2[1] - p1[1])
                    mid_z = p1[2] + t * (p2[2] - p1[2])
                    if is_inside_shape((mid_x, mid_y, mid_z), pre_seg_mask):
                        inside_steps += 1
                fraction = inside_steps / steps
                inside_count += fraction
                total_count += 1
        
        if total_count > 0:
            convexity = inside_count / total_count
        else:
            convexity = 0.0

    # 5. Compute Surfaceness
    surfaceness = (planarity + convexity) * (1.0 - tubiness)

    # 6. Compute Blobbiness
    # Variance of region
    center_of_mass = [
        sum(v[0] for v in region_voxels) / len(region_voxels),
        sum(v[1] for v in region_voxels) / len(region_voxels),
        sum(v[2] for v in region_voxels) / len(region_voxels)
    ]
    
    variance_r = sum(
        (v[0]-center_of_mass[0])**2 + 
        (v[1]-center_of_mass[1])**2 + 
        (v[2]-center_of_mass[2])**2 
        for v in region_voxels
    )
    
    # Volume (number of voxels)
    vol = len(region_voxels)
    
    # Variance of sphere with same volume
    # r = (3 * vol / (4 * pi))^(1/3)
    # sigma_sph^2 = (4/5) * pi * r^5
    if vol > 0:
        r = (3.0 * vol / (4.0 * math.pi)) ** (1.0/3.0)
        variance_sph = (4.0/5.0) * math.pi * (r ** 5)
        
        if variance_r == 0:
            blobbiness = 1.0 # Perfectly compact (point)
        else:
            blobbiness = variance_sph / variance_r
    else:
        blobbiness = 0.0

    return (tubiness, surfaceness, blobbiness)

def clamp(value, min_val, max_val):
    return max(min_val, min(value, max_val))

def is_surface_voxel(v, region_voxels_set, pre_seg_mask):
    # Check if v is in region and has a neighbor not in region or outside pre_seg_mask
    # Implementation depends on data structure
    pass

def is_inside_shape(point, pre_seg_mask):
    # Check if point is inside the pre-segmented object
    # Implementation depends on data structure
    pass
```

## Subtopics

### [Barycentric Shape Selection Widget](5970-barycentric-shape-selection-widget.md)
The Barycentric Shape Selection Widget is a user interface component that maps 4D transfer function space (tubiness, surfaceness, blobbiness, and intensity) onto a 2D equilateral triangle using barycentric coordinates, allowing users to intuitively select shape classes for volume rendering. It distinguishes itself from scalar-only transfer functions by enabling direct manipulation of geometric shape memberships (tubiness, surfaceness, blobbiness) derived from curve skeletons, serving as the first stage of classification before intensity-based optical property assignment.

## Operating characteristics
*   **Time Complexity:** Dominated by skeleton computation (potential field method is slow) and distance map calculation. Skeleton normalization and merging are linear in the number of voxels/skeleton points. Shape analysis is linear in the number of surface voxels and pairs for convexity.
*   **Memory Complexity:** Requires storage for the volume, pre-segmentation mask, distance map, and skeleton data structures.
*   **Measured Performance:** The potential field method is the slowest among examined skeletonization techniques but produces the cleanest and smoothest curve-skeletons [S3]. Preprocessing time is not a main issue for the use case [S3].
*   **Strengths:** Intuitive user interface; distinguishes features with overlapping intensities; robust to noise via normalization; handles curved longitudinal shapes [S3].
*   **Limitations:** Requires pre-segmentation; imperfect pre-segmentations cause instabilities; tubiness classifier reliability depends on skeleton quality; heavily branching skeletons cause problems for decomposition and merging [S3].
*   **When to use:** When features have similar intensities but different shapes (e.g., vessels vs. bones); when intuitive shape-based classification is desired.
*   **When not to use:** When real-time performance is critical and preprocessing is prohibitive; when pre-segmentation is impossible or highly inaccurate.

## Background
The concept of axial description of shape (skeleton) was proposed in 2D by Blum [2]. Binford [1] proposed generalized cylinders in 3D. Cornea et al. [4, 5] introduced hierarchical curve-skeletons using repulsive force fields. The method was described by Praßni et al. [17] [S1, unofficial][S3].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S3] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
