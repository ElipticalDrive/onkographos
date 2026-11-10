# Shape-based Transfer Function Generation

Also called: Shape classification, Shape-based TF (Skeleton), Shape-based Transfer Functions, Curve-Skeleton Shape Classification TF, Shape classification for volume visualization, Skeleton-based shape descriptor classification

## Overview
Shape-based Transfer Function Generation classifies volumetric features based on their 3D geometric shape rather than solely on intensity or gradient magnitude, solving the problem of distinguishing structures with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) [S3]. Unlike per-voxel classification methods that rely on local filters, this method assigns a uniform shape descriptor to each entire volumetric feature (segment) derived from curve-skeletons, allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels [S3]. It distinguishes itself from [Distance-based Transfer Function](5451-distance-based-transfer-function.md) by using topological skeletons for shape metrics rather than distance transforms alone, and from [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) by classifying complete volumetric features rather than per-voxel local structures [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** Raw volume data $V$, a binary pre-segmentation mask $M$ specifying surface voxels of objects of interest.
*   **Output:** A shape descriptor triple $(\tau, \sigma, \beta)$ for each voxel, where $\tau$ is tubiness, $\sigma$ is surfaceness, and $\beta$ is blobbiness. Background voxels have descriptor $(0,0,0)$.

**Data Structures:**
*   **Pre-segmentation Mask:** Binary volume $M \in \{0, 1\}^N$.
*   **Curve-Skeleton:** A set of polygonal chains (segments). Each segment $S_k$ is a list of 3D points $p_1, \dots, p_m$.
*   **Skeleton Regions:** A partition of the volume voxels belonging to the pre-segmentation. Each voxel $v$ is assigned to a region $R_i$ corresponding to a skeleton segment $S_i$.
*   **Distance Map:** $D(v)$, the minimal Euclidean distance from voxel $v$ to the nearest point on the curve-skeleton of its region.

**Parameters:**
*   **Skeleton Length Threshold ($L_{thresh}$):** 5 to 10 voxels for $256^3$ to $512^3$ datasets [S3].
*   **Field Strength:** 4 (for potential field method) [S3].
*   **High Divergence Points:** 20% (default for potential field method) [S3].

**Steps:**

1.  **Pre-segmentation:**
    Perform a rough threshold-based pre-segmentation (windowing) of the volume data to obtain surface information. This defines the binary mask $M$. Simple methods like windowing are sufficient; full segmentation is not required [S3].
    *   *Pitfall:* Imperfect pre-segmentations due to noise or low signal-to-noise ratio cause instabilities in shape-skeletons. Add extra layers of voxels at the surface of objects to handle cavities caused by noise, though this may smooth objects and merge adjacent features [S3].

2.  **Skeleton Computation:**
    Compute curve-skeletons for each volumetric structure in $M$. The method uses the potential field method (e.g., Cornea’s pfSkel) [S3].
    *   Input: Binary volume $M$.
    *   Output: Multiple skeleton segments (polygonal chains).
    *   Parameters: Field strength = 4, High divergence points = 20% [S3].

3.  **Skeleton Normalization:**
    Normalize the computed skeletons to reduce artifacts from imperfect segmentations [S3].
    *   **Gap Closing:** For thin longitudinal structures, connect each end node of a skeleton graph with its next neighboring skeleton point outside the end node's segment, if the connecting line lies completely within the pre-segmented object $M$. This handles gaps that boost standard deviation in tubiness calculation [S3].
    *   **Segment Splitting:** Split curve-skeletons into segments not exceeding length $L_{thresh}$ (5–10 voxels). This prevents blobby shapes transitioning into tubular structures from being classified as a single ambiguous shape [S3].
    *   **Pruning:** Discard degenerated skeleton branches shorter than $L_{thresh}$ [S3].

4.  **Decomposition:**
    Decompose the pre-segmented volume based on the normalized curve-skeleton segments. Divide the volume into 'skeleton regions' $R_i$, where each region corresponds to one segment of the curve-skeleton. A voxel $v$ belongs to region $R_i$ if it is in $M$ and its closest point on the skeleton is in segment $S_i$ [S3].

5.  **Merging:**
    Merge adjacent skeleton regions based on criteria to improve classifiability, as ad-hoc decomposition may yield regions without meaningful shape. The merging process considers both geometry and shape properties of skeleton regions [S3].

6.  **Shape Analysis:**
    Assign shape scores (membership values) to the merged skeleton regions. No binary decisions are made; scores specify how much the object part matches each shape class. Compute the following descriptors for each region $R$:

    *   **Tubiness $\tau(R)$:**
        Measures longitudinal/tubular membership. Defined as the inverse of the standard deviation of surface voxels' minimal skeleton distances [S3].
        $$ \tau(R) := \frac{1}{\max\left(\sqrt{\frac{1}{|\partial R|} \sum_{\vec{v} \in \partial R} (D(\vec{v}) - \bar{d})^2}, 1\right)} \in [0, 1] $$
        Where $\partial R$ is the set of surface voxels of $R$, $D(\vec{v})$ is the distance map, and $\bar{d}$ is the mean skeleton distance of all $\vec{v} \in \partial R$. Reliability depends on skeleton quality; incomplete skeletons boost standard deviation [S3].

    *   **Surfaceness $\sigma(R)$:**
        Measures surface-like membership. Combines planarity and convexity, weighted by inverse tubiness to avoid misclassifying curved longitudinal shapes [S3].
        $$ \sigma(R) := (plan(R) + conv(R)) \cdot (1 - \tau(R)) \in [0, 2] $$
        
        *   **Planarity $plan(R)$:** Based on the minimal oriented bounding box's second shortest side $b_m(R)$ and shortest side $b_s(R)$ [S3]:
            $$ plan(R) := \text{clamp}\left(\frac{1}{5} \frac{b_m(R)}{b_s(R)} - 1, 0, 1\right) $$
            Regions with $b_m \ge 10 \cdot b_s$ are maximal flat; regions with $b_m \le 5 \cdot b_s$ are not flat at all [S3].
            
        *   **Convexity $conv(R)$:** For curved surfaces, select an arbitrary set of pairs of surface points. For each line segment formed by a point pair, determine the fraction that runs inside the shape. $conv(R)$ is the average of these fractions [S3].

    *   **Blobbiness $\beta(R)$:**
        Measures blobby/compact membership. Defined as the region's variance relative to the variance of a sphere with the same volume [S3].
        $$ \beta(R) := \frac{\sigma^2_{sph}(vol)}{\sigma^2(R)} \in [0, 1] $$
        
        *   **Variance $\sigma^2(R)$:** Measures compactness.
            $$ \sigma^2(R) := \sum_{\vec{r} \in R} |\vec{r} - m|^2 $$
            Where $\vec{r}$ are voxel coordinates and $m$ is the center of mass: $m = \frac{1}{|R|} \sum_{\vec{r} \in R} \vec{r}$ [S3].
            
        *   **Sphere Variance $\sigma^2_{sph}(vol)$:**
            Radius $r(vol) = \left(\frac{3}{4\pi} vol\right)^{1/3}$.
            Variance of a sphere with radius $r_0$ centered at origin: $\sigma^2_{r0} = \int_0^{r0} (4\pi r^2) r^2 dr = \frac{4}{5} \pi r_0^5$ [S3].
            Substituting $r_0 = r(vol)$:
            $$ \sigma^2_{sph}(vol) = \frac{4}{5} \pi \left(\frac{3}{4\pi} vol\right)^{5/3} $$

7.  **Transfer Function Specification:**
    Use the shape scores to allow the user to assign optical properties (color, opacity) to the shape classes via a continuous triangle-shaped plot showing the occurrence of detected shape classes [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shape_descriptor(region_voxels, skeleton_segment, pre_seg_mask):
    """
    Computes tubiness, surfaceness, and blobbiness for a merged skeleton region.
    
    Args:
        region_voxels: List of 3D coordinates (x,y,z) of voxels in the region.
        skeleton_segment: List of 3D coordinates forming the polygonal chain.
        pre_seg_mask: Binary volume or function to check if a point is in the object.
        
    Returns:
        tuple: (tubiness, surfaceness, blobbiness)
    """
    if not region_voxels:
        return (0.0, 0.0, 0.0)

    # 1. Compute Distance Map and Surface Voxels
    # D(v) is min distance from v to skeleton_segment
    # Surface voxels are those in region_voxels adjacent to background (not in pre_seg_mask)
    # Note: In implementation, 'surface' usually means boundary voxels of the region
    surface_voxels = get_surface_voxels(region_voxels, pre_seg_mask)
    
    if not surface_voxels:
        return (0.0, 0.0, 0.0)

    # Compute distances from surface voxels to skeleton
    distances = []
    for v in surface_voxels:
        min_dist = infinity
        for s_point in skeleton_segment:
            d = euclidean_distance(v, s_point)
            if d < min_dist:
                min_dist = d
        distances.append(min_dist)

    # 2. Compute Tubiness
    mean_d = sum(distances) / len(distances)
    variance_d = sum((d - mean_d)**2 for d in distances) / len(distances)
    std_d = sqrt(variance_d)
    tubiness = 1.0 / max(std_d, 1.0)

    # 3. Compute Surfaceness
    # 3a. Planarity
    bbox = compute_oriented_bounding_box(region_voxels)
    side_lengths = sorted(bbox.sides) # [shortest, middle, longest]
    b_s = side_lengths[0]
    b_m = side_lengths[1]
    
    if b_s == 0:
        planarity = 0.0
    else:
        ratio = b_m / b_s
        planarity = clamp((1.0 / 5.0) * ratio - 1.0, 0.0, 1.0)

    # 3b. Convexity
    # Select arbitrary pairs of surface points (e.g., random sampling or all pairs if small)
    # For simplicity, assume we sample N pairs
    num_pairs = 100
    inside_fractions = []
    for _ in range(num_pairs):
        p1 = random.choice(surface_voxels)
        p2 = random.choice(surface_voxels)
        # Determine fraction of line segment p1->p2 inside the shape
        # This requires ray-casting or voxel traversal
        fraction = compute_line_inside_fraction(p1, p2, pre_seg_mask)
        inside_fractions.append(fraction)
    
    convexity = sum(inside_fractions) / len(inside_fractions)

    surfaceness = (planarity + convexity) * (1.0 - tubiness)

    # 4. Compute Blobbiness
    # 4a. Variance of region
    center_of_mass = compute_center_of_mass(region_voxels)
    variance_r = sum(euclidean_squared_distance(r, center_of_mass) for r in region_voxels)
    
    # 4b. Variance of sphere with same volume
    vol = len(region_voxels) # Assuming unit voxel volume
    r_sphere = (3.0 * vol / (4.0 * pi)) ** (1.0 / 3.0)
    variance_sphere = (4.0 / 5.0) * pi * (r_sphere ** 5)
    
    if variance_r == 0:
        blobbiness = 0.0
    else:
        blobbiness = variance_sphere / variance_r

    return (tubiness, surfaceness, blobbiness)

def normalize_skeleton(skeleton_segments, pre_seg_mask):
    """
    Normalizes skeleton segments by closing gaps, splitting, and pruning.
    """
    normalized_segments = []
    
    # Step 1: Close gaps in longitudinal structures
    # Connect end nodes to next neighbor if line lies within pre_seg_mask
    # Implementation detail: Iterate through segments, identify ends, check connectivity
    connected_segments = close_skeleton_gaps(skeleton_segments, pre_seg_mask)
    
    # Step 2 & 3: Split and Prune
    L_thresh = 10 # Example threshold
    for seg in connected_segments:
        if length(seg) > L_thresh:
            # Split into chunks of max length L_thresh
            chunks = split_segment(seg, L_thresh)
            normalized_segments.extend(chunks)
        # Discard segments shorter than L_thresh (pruning)
        
    return normalized_segments
```

## Subtopics

### [Barycentric Shape Selection Widget](5970-barycentric-shape-selection-widget.md)
The Barycentric Shape Selection Widget is a user interface component that maps 4D transfer function space (tubiness, surfaceness, blobbiness, and intensity) onto a 2D equilateral triangle using barycentric coordinates, allowing users to intuitively select shape classes for volume rendering. It distinguishes itself from scalar-only transfer functions by enabling direct manipulation of geometric shape memberships (tubiness, surfaceness, blobbiness) derived from curve skeletons, serving as the first stage of classification before intensity-based optical property assignment.

## Operating characteristics
*   **Time Complexity:** Dominated by skeleton computation (potential field method is slowest among examined techniques) and distance map calculation [S3]. Preprocessing time is not a main issue for the use case [S3].
*   **Memory Complexity:** Requires storage for the volume, binary mask, distance map, and skeleton data structures.
*   **Measured Performance:** Not given in the sources.
*   **Scaling:** Length thresholds for skeleton splitting (5–10 voxels) are dependent on dataset size ($256^3$ to $512^3$) [S3].
*   **Strengths:** Allows visual separation of objects with similar intensities but different shapes (e.g., contrast-filled heart vs. vessels/ribs) [S3]. Intuitive user interface for domain experts [S3]. Robust to moderate within-class deformation [S3].
*   **Limitations:** Requires proper pre-segmentation/masking, which can be challenging [S1, unofficial]. Relies on skeleton quality; highly degenerated tubular regions may be misclassified [S3]. Potential field method is slow [S3].
*   **When to use:** When features have overlapping intensity ranges but distinct 3D shapes (e.g., medical imaging of vessels vs. bones/organs) [S3].
*   **When not to use:** When pre-segmentation is impossible or highly unreliable, or when per-voxel local texture/curvature is more important than global shape [S3].

## Background
The concept of axial description of shape (skeleton) was proposed in 2D by Blum [2]. Binford [1] proposed generalized cylinders in 3D. Cornea et al. [4, 5] introduced hierarchical curve-skeletons using repulsive force fields. The specific method of Shape-based Transfer Function Generation was proposed by Praßni et al. [S3][S1, unofficial]. It is described in reference [17] [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S3] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
