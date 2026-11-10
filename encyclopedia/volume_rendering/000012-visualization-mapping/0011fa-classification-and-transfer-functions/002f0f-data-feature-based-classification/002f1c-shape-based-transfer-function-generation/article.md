# Shape-based Transfer Function Generation

Also called: Shape classification, Shape-based TF (Skeleton), Shape-based Transfer Functions, Curve-Skeleton Shape Classification TF, Shape classification for volume visualization, Skeleton-based shape descriptor classification

## Overview
Shape-based Transfer Function Generation is a classification method that assigns optical properties to volumetric features based on their 3D shape rather than just intensity or gradient magnitude [S3]. It solves the problem of distinguishing features with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) by using a shape descriptor derived from curve-skeletons [S3]. Unlike per-voxel classification methods such as [Curvature-based Transfer Function](5452-curvature-based-transfer-function.md) which use multi-scale filters, this method classifies complete volumetric features using curve-skeletons, allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A scalar volume dataset $V$ and a binary pre-segmentation mask $M$ specifying surface voxels of objects of interest.
*   **Output:** A shape descriptor triple $(\tau, \sigma, \beta)$ for each voxel, where $\tau$ is tubiness, $\sigma$ is surfaceness, and $\beta$ is blobbiness. Background voxels have descriptor $(0,0,0)$.

**Data Structures:**
*   **Pre-segmentation Mask:** Binary volume $M \in \{0, 1\}^N$.
*   **Curve-Skeleton:** A set of polygonal chains (segments). Each segment is a list of 3D points $(x, y, z)$.
*   **Skeleton Regions:** A partition of the volume where each voxel belongs to at most one skeleton region $R_i$.
*   **Distance Map:** $D(\vec{v})$ stores the minimal Euclidean distance from surface voxel $\vec{v}$ to the skeleton.

**Numbered Steps:**

1.  **Pre-segmentation:** Perform a rough, threshold-based pre-segmentation (windowing) of the volume data to obtain surface information about objects of interest [S3]. If intensity ranges are known (e.g., Hounsfield scale for CT), this can be automatic [S3].
2.  **Skeleton Computation:** Compute the curve-skeletons for each resulting volumetric structure using the potential field method (e.g., Cornea’s pfSkel) [S3].
    *   Input: Binary volume $M$.
    *   Parameters: Field strength = 4; Percentage of high divergence points = 20% [S3].
    *   Output: Multiple skeleton segments (polygonal chains) [S3].
    *   *Note:* Add extra layers of voxels at the surface of objects to prevent cavities caused by noise from disturbing skeleton computation [S3].
3.  **Skeleton Normalization:** Normalize the computed skeletons to reduce artifacts caused by imperfect segmentations [S3].
    *   **Gap Closing:** Connect each end node of the skeleton graph with its next neighboring skeleton point outside the end node's segment, if the connecting line lies completely within the pre-segmented object [S3].
    *   **Segmentation:** Split the curve-skeleton into segments not exceeding a certain length. For datasets of size $256^3$ to $512^3$, length thresholds of 5 to 10 voxels are effective [S3].
    *   **Pruning:** Discard degenerated skeleton branches shorter than the length threshold [S3].
4.  **Decomposition:** Decompose the pre-segmented volume based on the curve-skeleton segments. Divide the volume into 'skeleton regions' $R$, where each region corresponds to one segment of the curve-skeleton [S3].
5.  **Merging:** Merge adjacent skeleton regions based on criteria to improve classifiability, as individual skeleton regions from ad-hoc decomposition may not have a meaningful shape [S3].
6.  **Shape Analysis:** Assign shape scores (membership values) to the merged skeleton regions [S3]. Compute the three shape descriptors for each region $R$:

    **Tubiness $\tau(R)$:**
    Defined as the inverse of the standard deviation of surface voxels' minimal skeleton distances [S3].
    $$ \tau(R) := \frac{1}{\max\left(\sqrt{\frac{1}{|\partial R|} \sum_{\vec{v} \in \partial R} (D(\vec{v}) - \bar{d})^2}, 1\right)} \in [0, 1] $$
    Where $\partial R$ is the set of surface voxels of $R$, $D$ is the distance map, and $\bar{d}$ is the mean skeleton distance of all $\vec{v} \in \partial R$ [S3].

    **Surfaceness $surf(R)$:**
    Defined as the sum of planarity and convexity weighted by inverse tubiness [S3].
    $$ surf(R) := (plan(R) + conv(R)) \cdot (1 - \tau(R)) \in [0, 2] $$
    
    *   **Planarity $plan(R)$:** Based on the minimal oriented bounding box's second shortest side $b_m(R)$ and shortest side $b_s(R)$ [S3]:
        $$ plan(R) := \text{clamp}\left(\frac{1}{5} \frac{b_m(R)}{b_s(R)} - 1, 0, 1\right) \in [0, 1] $$
        Regions with $b_m \ge 10 \cdot b_s$ are maximal flat; regions with $b_m \le 5 \cdot b_s$ are not flat at all [S3].
        
    *   **Convexity $conv(R)$:** Select an arbitrary set of pairs of surface points. For each line segment formed by a point pair, determine the fraction that runs inside the shape. $conv(R)$ is the average of these fractions [S3].

    **Blobbiness $blob(R)$:**
    Defined as the region's variance $\sigma^2(R)$ relative to the variance of a sphere with the same volume $\sigma^2_{sph}(vol)$ [S3].
    $$ blob(R) := \frac{\sigma^2_{sph}(vol)}{\sigma^2(R)} \in [0, 1] $$
    
    *   **Variance $\sigma^2(R)$:** Measures compactness [S3]:
        $$ \sigma^2(R) := \sum_{\vec{r} \in R} |\vec{r} - m|^2 $$
        Where $\vec{r}$ are voxel coordinates and $m$ is the center of mass: $m = \frac{1}{|R|} \sum_{\vec{r} \in R} \vec{r}$ [S3].
        
    *   **Sphere Variance $\sigma^2_{sph}(vol)$:**
        Radius $r(vol) = \left(\frac{3}{4\pi} vol\right)^{1/3}$ [S3].
        Variance of a sphere with radius $r_0$: $\sigma^2_{r0} = \frac{4}{5} \pi r_0^5$ [S3].
        Substituting $r_0 = r(vol)$:
        $$ \sigma^2_{sph}(vol) = \frac{4}{5} \pi \left(\frac{3}{4\pi} vol\right)^{5/3} $$
        Where $vol = |R|$ (number of voxels in region) [S3].

7.  **Transfer Function Specification:** Use the shape scores to allow the user to assign optical properties (color, opacity) to the shape classes via a continuous triangle-shaped plot showing the occurrence of detected shape classes [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shape_descriptor(region_voxels, surface_voxels, skeleton_segment, distance_map):
    """
    Computes tubiness, surfaceness, and blobbiness for a merged skeleton region.
    
    Args:
        region_voxels: List of 3D coordinates (x,y,z) of all voxels in the region R.
        surface_voxels: List of 3D coordinates of surface voxels in R.
        skeleton_segment: List of 3D coordinates of the skeleton segment points.
        distance_map: Function/Array providing D(v) for any voxel v.
        
    Returns:
        Tuple (tubiness, surfaceness, blobbiness)
    """
    if not region_voxels:
        return (0.0, 0.0, 0.0)

    # 1. Compute Tubiness
    # Calculate distances from surface voxels to the skeleton segment
    # D(v) is assumed to be precomputed or computed via distance_map
    distances = [distance_map(v) for v in surface_voxels]
    
    if not distances:
        return (0.0, 0.0, 0.0)

    mean_d = sum(distances) / len(distances)
    variance_d = sum((d - mean_d) ** 2 for d in distances) / len(distances)
    std_d = variance_d ** 0.5
    
    # Clamp denominator to 1 to avoid division by zero if std is 0
    tubiness = 1.0 / max(std_d, 1.0)

    # 2. Compute Surfaceness
    # Planarity
    # Compute minimal oriented bounding box (MOBB) for region_voxels
    # This requires an external algorithm (e.g., Har-Peled's technique)
    # Returns dimensions (d1, d2, d3) sorted such that d1 <= d2 <= d3
    dims = compute_mobb_dimensions(region_voxels)
    bs = dims[0] # Shortest side
    bm = dims[1] # Second shortest side
    
    if bs == 0:
        planarity = 0.0
    else:
        ratio = bm / bs
        planarity = clamp((ratio / 5.0) - 1.0, 0.0, 1.0)

    # Convexity
    # Select arbitrary pairs of surface points
    # For simplicity, assume a fixed number of random pairs or all pairs if small
    # Fraction of line segment inside the shape
    num_pairs = 100 # Arbitrary sample size
    convexity_sum = 0.0
    for _ in range(num_pairs):
        p1 = random.choice(surface_voxels)
        p2 = random.choice(surface_voxels)
        inside_fraction = compute_line_segment_inside_fraction(p1, p2, region_voxels)
        convexity_sum += inside_fraction
    
    convexity = convexity_sum / num_pairs

    surfaceness = (planarity + convexity) * (1.0 - tubiness)

    # 3. Compute Blobbiness
    # Variance of the region
    center_of_mass = compute_center_of_mass(region_voxels)
    variance_r = sum((v - center_of_mass) ** 2 for v in region_voxels) # Sum of squared distances
    
    # Variance of a sphere with same volume
    volume = len(region_voxels)
    # r = (3 * vol / (4 * pi))^(1/3)
    radius = (3.0 * volume / (4.0 * math.pi)) ** (1.0/3.0)
    # sigma_sph^2 = (4/5) * pi * r^5
    variance_sphere = (4.0 / 5.0) * math.pi * (radius ** 5)
    
    if variance_r == 0:
        blobbiness = 0.0 # Or 1.0 if perfectly compact? Source says [0,1]. 
                         # If variance is 0, it's a point. Sphere variance > 0. 
                         # Ratio -> infinity? Source says [0,1]. 
                         # Assuming if variance_r is very small, blobbiness is high.
                         # Formula: sigma_sph / sigma_r. If sigma_r -> 0, blobbiness -> inf.
                         # But source says [0,1]. 
                         # Re-reading source: "blobbiness ... in [0,1]".
                         # If variance_r < variance_sphere, ratio > 1.
                         # The source formula is blob(R) := sigma_sph^2 / sigma^2(R).
                         # Wait, source Eq 9: blob(R) := sigma_sph^2(vol) / sigma^2(R).
                         # If R is a sphere, sigma^2(R) = sigma_sph^2, so blob=1.
                         # If R is spread out, sigma^2(R) > sigma_sph^2, so blob < 1.
                         # If R is a point, sigma^2(R) = 0. Division by zero.
                         # Pitfall: "Highly degenerated tubular regions ... may be misclassified."
                         # We assume non-degenerate regions for this calculation.
        blobbiness = 0.0 
    else:
        blobbiness = variance_sphere / variance_r
        # Clamp to [0,1] if necessary, though mathematically <= 1 for compact shapes?
        # Actually, if shape is less compact than sphere, sigma^2 > sigma_sph^2, so ratio < 1.
        # If shape is more compact (impossible for discrete voxels vs continuous sphere?), ratio > 1.
        # Clamp to 1.
        blobbiness = min(blobbiness, 1.0)

    return (tubiness, surfaceness, blobbiness)

def compute_center_of_mass(voxels):
    n = len(voxels)
    if n == 0:
        return (0,0,0)
    sum_x = sum(v[0] for v in voxels)
    sum_y = sum(v[1] for v in voxels)
    sum_z = sum(v[2] for v in voxels)
    return (sum_x/n, sum_y/n, sum_z/n)

def compute_line_segment_inside_fraction(p1, p2, region_voxels_set):
    # Approximate by sampling points along the segment
    # Check if each sample point is in region_voxels_set
    # This is a placeholder for the actual geometric intersection test
    # [G] Implementation detail: Use ray casting or voxel traversal.
    # For brevity, assume a helper exists.
    pass

def clamp(value, min_val, max_val):
    return max(min_val, min(value, max_val))
```

## Subtopics

### [Barycentric Shape Selection Widget](5970-barycentric-shape-selection-widget.md)
The Barycentric Shape Selection Widget is a user interface component that maps 4D transfer function space (tubiness, surfaceness, blobbiness, and intensity) onto a 2D equilateral triangle using barycentric coordinates, allowing users to intuitively select shape classes for volume rendering. It distinguishes itself from scalar-only transfer functions by enabling direct manipulation of geometric shape memberships (tubiness, surfaceness, blobbiness) derived from curve skeletons, serving as the first stage of classification before intensity-based optical property assignment.

## Operating characteristics
*   **Time Complexity:** The method is computationally expensive due to skeleton computation (potential field method is the slowest examined technique) and bounding box/convexity calculations [S3]. Preprocessing time is not a main issue for the use case [S3].
*   **Memory Complexity:** Requires storage for the volume, binary mask, distance map, and skeleton data structures.
*   **Strengths:** Allows visual separation of objects with similar intensities but different shapes (e.g., contrast-filled heart vs. vessels) without complex segmentation [S3]. The notion of shape is intuitive for domain experts [S3].
*   **Limitations:** Requires proper masking/pre-segmentation, which can be challenging [S1, unofficial]. Imperfect pre-segmentations cause instabilities in shape-skeletons [S3]. Tubiness classifier reliability depends on skeleton quality; incomplete skeletons boost standard deviation [S3]. Heavily branching skeletons cause problems for decomposition and merging [S3].
*   **When to use:** When features have overlapping intensity ranges and distinct 3D shapes (tubular, planar, blobby).
*   **When not to use:** When pre-segmentation is impossible or highly unreliable, or when real-time performance is critical and preprocessing time is prohibitive.

## Background
The concept of axial description of shape (skeleton) was proposed in 2D by Blum [2]. Binford [1] proposed generalized cylinders in 3D. Cornea et al. [4, 5] introduced hierarchical curve-skeletons using repulsive force fields. The specific method of Shape-based Transfer Functions was proposed by Praßni et al. [S3]. It is described in reference [17] [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S3] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
