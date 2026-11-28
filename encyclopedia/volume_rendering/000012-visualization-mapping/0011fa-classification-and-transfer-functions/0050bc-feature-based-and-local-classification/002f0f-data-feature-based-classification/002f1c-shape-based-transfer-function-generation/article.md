# Shape-based Transfer Function Generation

Also called: Shape classification, Shape-based TF (Skeleton), Shape-based Transfer Functions, Curve-Skeleton Shape Classification TF, Shape classification for volume visualization, Skeleton-based shape descriptor classification

## Overview
Shape-based Transfer Function Generation is a classification method that assigns optical properties to volumetric features based on their 3D shape rather than solely on intensity or gradient magnitude [S3]. It solves the problem of distinguishing features with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) by using a shape descriptor derived from curve-skeletons [S3]. Unlike per-voxel classification methods that rely on local filters [S3], this method assigns a uniform shape descriptor to each entire volumetric feature (segment), allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels [S3].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume dataset $V$ and a pre-segmentation mask $M$ (binary volume specifying surface voxels of objects of interest) [S3].
*   **Output:** A shape descriptor triple $(\tau, \sigma, \beta)$ for each voxel in the pre-segmented region, where $\tau$ is tubiness, $\sigma$ is surfaceness, and $\beta$ is blobbiness [S3]. Background voxels have descriptor $(0,0,0)$ [S3].

**Data Structures:**
*   **Pre-segmentation Mask:** Binary volume $M \in \{0, 1\}^N$ specifying surface voxels [S3].
*   **Curve-Skeleton:** A set of polygonal chains of seed points [S3].
*   **Skeleton Regions:** A partition of the pre-segmented voxels, where each region corresponds to one segment of the curve-skeleton [S3].
*   **Distance Map:** A volume $D$ where $D(\vec{v})$ is the minimal Euclidean distance from voxel $\vec{v}$ to the nearest point on the curve-skeleton [S3].

**Parameters:**
*   **Skeleton Length Threshold ($L_{thresh}$):** Used to split long segments. Typical value: 5–10 voxels for $256^3$ to $512^3$ datasets [S3].
*   **Field Strength:** For potential field skeletonization. Typical value: 4 (low value for minimal complexity) [S3].
*   **High Divergence Points Percentage:** For skeletonization. Typical value: 20% [S3].

**Steps:**

1.  **Pre-segmentation:**
    Perform a rough threshold-based pre-segmentation (windowing) of the volume data to obtain surface information about objects of interest [S3]. This can be automatic if intensity ranges are known (e.g., Hounsfield scale for CT) [S3].
    *   *Pitfall:* Imperfect pre-segmentations (noise, low SNR) cause instabilities in shape-skeletons [S3].
    *   *Mitigation:* To prevent cavities inside the pre-segmentation from disturbing skeleton computation, add extra layers of voxels at the surface of objects [S3]. This may smooth the object and cause unwanted merging of adjacent features [S3].

2.  **Skeleton Computation:**
    Compute the curve-skeletons for each resulting volumetric structure using the potential field method (e.g., Cornea’s pfSkel) [S3]. Input is the binary volume $M$. Output is multiple skeleton segments (polygonal chains of seed points) [S3].

3.  **Skeleton Normalization:**
    Normalize the computed skeletons to reduce artifacts caused by imperfect segmentations [S3].
    *   **Step 3a (Gap Closing):** For thin longitudinal structures, close gaps in the skeleton. Connect each end node of the skeleton graph with its next neighboring skeleton point outside the end node's segment, if the connecting line lies completely within the pre-segmented object [S3].
    *   **Step 3b (Splitting):** Split curve-skeleton segments that exceed length $L_{thresh}$ [S3].
    *   **Step 3c (Pruning):** Discard degenerated skeleton branches shorter than $L_{thresh}$ [S3].

4.  **Decomposition:**
    Decompose the pre-segmented volume based on the curve-skeleton segments. Divide the volume into 'skeleton regions' $R$, where each region corresponds to one segment of the curve-skeleton [S3]. This is typically done by assigning each voxel in $M$ to the nearest point on the skeleton [G].

5.  **Merging:**
    Merge adjacent skeleton regions based on criteria to improve classifiability, as individual skeleton regions from ad-hoc decomposition may not have a meaningful shape [S3]. The fusion process takes into account the shape properties of skeleton regions [S3].

6.  **Shape Analysis:**
    Incorporate both curve-skeleton and volume data to assign shape scores (membership values) to the merged skeleton regions [S3]. No binary decisions are made; scores specify how much the object part matches each shape class [S3].

**Formulas:**

*   **Tubiness $\tau(R)$:**
    The inverse of the standard deviation of surface voxels' minimal skeleton distances [S3].
    $$ \tau(R) := \frac{1}{\max\left(\sqrt{\frac{1}{|\partial R|} \sum_{\vec{v} \in \partial R} (D(\vec{v}) - \bar{d})^2}, 1\right)} \in [0; 1] $$
    Where $\partial R$ is the set of surface voxels of $R$, $D$ is the distance map, and $\bar{d}$ is the mean skeleton distance of all $\vec{v} \in \partial R$ [S3].

*   **Planarity $plan(R)$:**
    Based on the minimal oriented bounding box's second shortest side $b_m(R)$ and shortest side $b_s(R)$ [S3].
    $$ plan(R) := \text{clamp}\left(\frac{1}{5} \frac{b_m(R)}{b_s(R)} - 1, 0, 1\right) \in [0; 1] $$
    Regions with $b_m \ge 10 \cdot b_s$ are maximal flat; regions with $b_m \le 5 \cdot b_s$ are not flat at all [S3].

*   **Convexity $conv(R)$:**
    For curved surfaces, compute convexity by selecting an arbitrary set of pairs of surface points. For each line segment formed by a point pair, determine the fraction that runs inside the shape. $conv(R)$ is the average of these fractions [S3].

*   **Surfaceness $surf(R)$:**
    The sum of planarity and convexity weighted by inverse tubiness [S3].
    $$ surf(R) := (plan(R) + conv(R)) \cdot (1 - \tau(R)) \in [0; 2] $$

*   **Blobbiness $blob(R)$:**
    The region's variance $\sigma^2(R)$ relative to the variance of a sphere with the same volume $\sigma^2_{sph}(vol)$ [S3].
    $$ blob(R) := \frac{\sigma^2_{sph}(vol)}{\sigma^2(R)} \in [0; 1] $$
    Where variance $\sigma^2(R)$ measures compactness:
    $$ \sigma^2(R) := \sum_{\vec{r} \in R} |\vec{r} - m|^2 $$
    And center of mass $m$:
    $$ m = \frac{1}{|R|} \sum_{\vec{r} \in R} \vec{r} $$
    The variance of a sphere with volume $vol$ is:
    $$ \sigma^2_{sph}(vol) = \frac{4}{5} \pi \left(\frac{3}{4\pi} vol\right)^{5/3} $$

7.  **Transfer Function Specification:**
    Use the shape scores to allow the user to assign optical properties (color, opacity) to the shape classes via a continuous triangle-shaped plot showing the occurrence of detected shape classes [S3].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def shape_based_transfer_function(volume_data, mask):
    # Step 1: Pre-segmentation is provided as binary mask [S3]
    # To prevent cavities inside the pre-segmentation from disturbing skeleton computation,
    # add extra layers of voxels at the surface of objects [S3].
    # Note: This may smooth the object and cause unwanted merging of adjacent features [S3].

    # Step 2: Skeleton Computation
    # Compute curve-skeletons using the potential field method (e.g., Cornea's pfSkel) [S3].
    # Input: binary mask. Output: list of polygonal chains of seed points [S3].
    skeleton_segments = compute_potential_field_skeleton(mask, field_strength=4, high_divergence_pct=0.2)

    # Step 3: Skeleton Normalization
    normalized_segments = []
    for segment in skeleton_segments:
        # 3a: Gap Closing
        segment = close_gaps(segment, mask)

        # 3b: Splitting
        split_segments = split_by_length(segment, threshold=7) # 5-10 voxels typical for 256^3 to 512^3 datasets [S3]
        normalized_segments.extend(split_segments)

        # 3c: Pruning
        # Discard degenerated skeleton branches shorter than threshold [S3]

    # Step 4: Decomposition
    # Assign each voxel in mask to the nearest point on the skeleton [G]
    # Create distance map D where D(v) is minimal Euclidean distance to skeleton [S3]
    D = compute_distance_map(normalized_segments, mask.shape)

    # Create skeleton regions R: partition of pre-segmented voxels [S3]
    skeleton_regions = decompose_volume(mask, normalized_segments, D)

    # Step 5: Merging
    # Merge adjacent skeleton regions based on criteria to improve classifiability [S3]
    merged_regions = merge_regions(skeleton_regions)

    # Initialize output descriptor array
    descriptors = [[[0.0, 0.0, 0.0] for _ in range(mask.shape[2])] for _ in range(mask.shape[1])] for _ in range(mask.shape[0])]

    # Step 6: Shape Analysis
    for region in merged_regions:
        # Get surface voxels of the region [S3]
        surface_voxels = get_surface_voxels(region, mask)

        # Compute Tubiness
        if len(surface_voxels) > 0:
            distances = [D[v] for v in surface_voxels]
            mean_d = sum(distances) / len(distances)
            std_d = math.sqrt(sum((d - mean_d)**2 for d in distances) / len(distances))
            tubiness = 1.0 / max(std_d, 1.0)
        else:
            tubiness = 0.0

        # Compute Surfaceness
        # Compute minimal oriented bounding box [S3]
        bbox = compute_oriented_bounding_box(region)
        b_m = bbox.second_shortest_side
        b_s = bbox.shortest_side

        if b_s > 0:
            planarity = max(0.0, min(1.0, (1.0/5.0) * (b_m / b_s) - 1.0))
        else:
            planarity = 0.0

        # Compute Convexity
        convexity = compute_convexity(surface_voxels, region)

        surfaceness = (planarity + convexity) * (1.0 - tubiness)

        # Compute Blobbiness
        # Compute Variance of region
        center_of_mass = [sum(r[i] for r in region) / len(region) for i in range(3)]
        variance = sum(sum((r[i] - center_of_mass[i])**2 for i in range(3)) for r in region)

        # Compute Variance of sphere with same volume
        vol = len(region)
        r_sph = (3.0 * vol / (4.0 * math.pi))**(1.0/3.0)
        variance_sphere = (4.0/5.0) * math.pi * r_sph**5

        if variance > 0:
            blobbiness = variance_sphere / variance
        else:
            blobbiness = 0.0

        # Assign descriptors to voxels in region
        for v in region:
            descriptors[v[0]][v[1]][v[2]] = [tubiness, surfaceness, blobbiness]

    return descriptors

def compute_potential_field_skeleton(mask, field_strength, high_divergence_pct):
    # Computes the potential-based vector field of the object specified in the input pre-segmentation
    # to construct the segments of the 3D curve-skeleton [S3].
    # Returns list of polygonal chains of seed points [S3].
    # Sources do not give the exact implementation details [G].
    return []

def close_gaps(segment, mask):
    # Connect each end node of the skeleton graph with its next neighboring skeleton point
    # outside the end node's segment, if the connecting line lies completely within the pre-segmented object [S3]
    # Sources do not give the exact implementation details [G].
    return segment

def split_by_length(segment, threshold):
    # Split curve-skeleton into segments not exceeding threshold length [S3]
    # Sources do not give the exact implementation details [G].
    return [segment]

def compute_distance_map(segments, shape):
    # Compute minimal Euclidean distance from each voxel to nearest skeleton point [S3]
    # Sources do not give the exact implementation details [G].
    return [[[0.0 for _ in range(shape[2])] for _ in range(shape[1])] for _ in range(shape[0])]

def decompose_volume(mask, segments, D):
    # Divide volume into skeleton regions, each corresponding to a segment [S3]
    # Sources do not give the exact implementation details [G].
    return []

def merge_regions(regions):
    # Merge adjacent skeleton regions based on criteria to improve classifiability [S3]
    # Sources do not give the exact implementation details [G].
    return regions

def get_surface_voxels(region, mask):
    # Return voxels in region that are on the boundary of the pre-segmentation [S3]
    # Sources do not give the exact implementation details [G].
    return []

def compute_oriented_bounding_box(region):
    # Compute minimal oriented bounding box and return side lengths [S3]
    # Sources do not give the exact implementation details [G].
    class BBox:
        def __init__(self):
            self.second_shortest_side = 1.0
            self.shortest_side = 1.0
    return BBox()

def compute_convexity(surface_voxels, region):
    # Select arbitrary set of pairs of surface points. For each line segment,
    # determine fraction running inside shape. Return average [S3]
    # Sources do not give the exact implementation details [G].
    return 0.0
```

## Subtopics

### [Barycentric Shape Selection Widget](5970-barycentric-shape-selection-widget.md)
The Barycentric Shape Selection Widget is a user interface component for shape-based transfer function generation that maps three normalized shape attributes (tubiness, surfaceness, blobbiness) into a 2D equilateral triangle using barycentric coordinates. It allows users to visually select specific shape classes by placing markers on the triangle, serving as the first stage of classification before intensity-based optical property assignment. It differs from intensity-only transfer functions by decoupling morphological selection from density selection, and from other shape methods by providing an interactive, geometric selection interface rather than a purely mathematical thresholding approach.

## Operating characteristics
*   **Time Complexity:** The method is computationally expensive due to skeletonization (potential field method is the slowest examined technique) and region merging [S3]. The running time of the preprocessing is not a main issue for the use case [S3].
*   **Memory Complexity:** Requires storage for the volume, mask, distance map, skeleton segments, and skeleton regions [S3].
*   **Strengths:** Allows visual separation of objects with similar intensities but different shapes (e.g., contrast-filled heart vs. vessels) [S3]. Intuitive user interface for domain experts [S3]. Robust to moderate within-class deformation [S3].
*   **Limitations:** Requires proper pre-segmentation, which can be challenging [S1, unofficial]. Imperfect pre-segmentations cause instabilities [S3]. Highly degenerated tubular regions may be misclassified [S3]. Heavily branching skeletons cause problems for decomposition and merging [S3].
*   **When to use:** When features have overlapping intensity ranges and distinct 3D shapes (e.g., medical imaging with contrast agents) [S3].
*   **When not to use:** When pre-segmentation is impossible or highly unreliable [S1, unofficial]. When real-time performance is critical and preprocessing time is prohibitive [S3].

## Background
The concept of axial description of shape (skeleton) was proposed in 2D by Blum [S3]. Binford proposed generalized cylinders in 3D [S3]. Cornea et al. introduced hierarchical curve-skeletons using repulsive force fields [S3]. The specific method of Shape-based Transfer Functions was proposed by Praßni et al. [S3]. It is described in reference [17] [S1, unofficial].

## Sources
[S1] Context-Preserving Volume Image Exploration Using A 3D Painting Metaphor, https://doi.org/10.32920/ryerson.14646888, fetched 2026-10-26 (unofficial)
[S2] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S3] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
