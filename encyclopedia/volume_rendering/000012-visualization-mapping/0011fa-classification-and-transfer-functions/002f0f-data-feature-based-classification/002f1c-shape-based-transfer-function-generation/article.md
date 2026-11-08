# Shape-based Transfer Function Generation

Also called: Shape classification, Shape classification for volume visualization, Shape descriptors, Shape-based Transfer Function, Skeleton-based shape descriptor classification

## Overview
Shape-based Transfer Function Generation classifies volumetric features based on their 3D geometric shape rather than solely on intensity or gradient magnitude, solving the problem of distinguishing structures with overlapping intensity ranges (e.g., contrast-filled vessels vs. bones) [S2]. Unlike per-voxel classification methods that use local filters, this method assigns a uniform shape descriptor to each entire volumetric feature (segment) derived from curve-skeletons, allowing for a more intuitive user interface where optical properties are assigned to shape classes rather than individual voxels [S2]. It builds on [Data Feature-Based Classification](5934-data-feature-based-classification.md) by using shape descriptors as the primary discriminative feature.

## How it works
The method classifies pre-segmented volumetric regions into three independent shape classes: longitudinal/tubular (tubiness), surface-like (surfaceness), and blobby (blobbiness) [S2]. The pipeline consists of pre-segmentation, skeleton computation, normalization, decomposition, merging, and shape analysis [S2].

### Inputs and Outputs
- **Input**: Raw volume data $V$, pre-segmentation mask $M$ (binary volume specifying surface voxels of objects of interest) [S2].
- **Output**: For each voxel $i$, a shape descriptor vector $\mathbf{s}_i = (\tau_i, \text{surf}_i, \text{blob}_i)$ and a class label (or membership scores) used to map to opacity $\alpha$ and color $\mathbf{c}$ [S2]. Background voxels ($M_i=0$) have descriptor $(0,0,0)$ [S2].

### Data Structures
- **Pre-segmentation Mask**: Binary volume $M \in \{0, 1\}^{N_x \times N_y \times N_z}$.
- **Curve-Skeleton**: List of segments, where each segment is a polygonal chain of 3D points $P = \{p_1, \dots, p_k\}$ [S2].
- **Skeleton Regions**: Partition of the pre-segmented volume $M$ into disjoint sets $R_j$, where each $R_j$ corresponds to one skeleton segment [S2].
- **Shape Descriptor**: For each region $R$, a tuple $(\tau(R), \text{surf}(R), \text{blob}(R))$ [S2].

### Steps

1. **Pre-segmentation**: Perform rough threshold-based pre-segmentation (windowing) to obtain surface information [S2]. If intensity ranges are known (e.g., Hounsfield units for CT), this can be automatic [S2]. Add extra layers of voxels at the surface of objects to handle cavities caused by noise/artifacts, which may smooth the object [S2].
2. **Skeleton Computation**: Compute curve-skeletons for each volumetric structure using the potential field method (e.g., Cornea’s pfSkel) [S2]. Input is the binary volume $M$. Output is multiple skeleton segments (polygonal chains) [S2].
    - *Parameters*: Field strength = 4 (for minimal complexity); Percentage of high divergence points = 20% [S2].
3. **Skeleton Normalization**:
    - *Gap Closing*: Connect end nodes of the skeleton graph with the next neighboring skeleton point outside the segment if the connecting line lies completely within $M$ [S2].
    - *Segmentation*: Split curve-skeleton into segments not exceeding a length threshold $L_{max}$ (typically 5–10 voxels for $256^3$ to $512^3$ data) [S2].
    - *Pruning*: Discard degenerated skeleton branches shorter than a length threshold [S2].
4. **Decomposition**: Divide the pre-segmented volume $M$ into skeleton regions $R_j$, where each region corresponds to one normalized skeleton segment [S2].
5. **Merging**: Merge adjacent skeleton regions based on criteria to improve classifiability, as ad-hoc decomposition may yield regions with no meaningful shape [S2].
    - *Criteria*: Merge if tubiness threshold > 0.8 and maximum inner-surface-to-outer-surface ratio < 0.1 [S2].
6. **Shape Analysis**: Compute shape scores for each merged region $R$ [S2].

### Formulas

**Tubiness** $\tau(R)$:
Measures the degree of longitudinal/tubular shape. It is the inverse of the standard deviation of surface voxels' minimal skeleton distances [S2].
$$ \tau(R) := \frac{1}{\max\left(\sqrt{\frac{1}{|\partial R|} \sum_{\vec{v} \in \partial R} (D(\vec{v}) - \bar{d})^2}, 1\right)} \in [0, 1] $$
where $\partial R$ is the set of surface voxels of $R$, $D(\vec{v})$ is the distance map from skeleton region growing, and $\bar{d}$ is the mean skeleton distance of all $\vec{v} \in \partial R$ [S2].

**Planarity** $\text{plan}(R)$:
Based on the minimal oriented bounding box's second shortest side $b_m(R)$ and shortest side $b_s(R)$ [S2].
$$ \text{plan}(R) := \text{clamp}\left(\frac{1}{5} \frac{b_m(R)}{b_s(R)} - 1, 0, 1\right) \in [0, 1] $$
Regions with $b_m \ge 10 \cdot b_s$ are maximal flat; regions with $b_m \le 5 \cdot b_s$ are not flat at all [S2].

**Convexity** $\text{conv}(R)$:
For curved surfaces, compute convexity by selecting an arbitrary set of pairs of surface points. For each line segment formed by a point pair, determine the fraction that runs inside the shape. $\text{conv}(R)$ is the average of these fractions [S2].

**Surfaceness** $\text{surf}(R)$:
Sum of planarity and convexity weighted by inverse tubiness [S2].
$$ \text{surf}(R) := (\text{plan}(R) + \text{conv}(R)) \cdot (1 - \tau(R)) \in [0, 2] $$

**Blobbiness** $\text{blob}(R)$:
Region's variance $\sigma^2(R)$ relative to the variance of a sphere with the same volume $\sigma^2_{\text{sph}}(\text{vol})$ [S2].
$$ \text{blob}(R) := \frac{\sigma^2_{\text{sph}}(\text{vol})}{\sigma^2(R)} \in [0, 1] $$
where $\text{vol} = |R|$ (number of voxels).

**Variance** $\sigma^2(R)$:
Measures compactness [S2].
$$ \sigma^2(R) := \sum_{\vec{r} \in R} |\vec{r} - m|^2 $$
where $\vec{r}$ are voxel coordinates and $m$ is the center of mass: $m = \frac{1}{|R|} \sum_{\vec{r} \in R} \vec{r}$ [S2].

**Sphere Variance** $\sigma^2_{\text{sph}}(\text{vol})$:
$$ \sigma^2_{\text{sph}}(\text{vol}) = \frac{4}{5} \pi \left(\frac{3}{4\pi} \text{vol}\right)^{5/3} $$
Derived from sphere radius $r(\text{vol}) = \left(\frac{3}{4\pi} \text{vol}\right)^{1/3}$ and integral $\int_0^{r_0} (4\pi r^2) r^2 dr = \frac{4}{5} \pi r_0^5$ [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_shape_descriptor(region_voxels, skeleton_segment, pre_seg_mask):
    """
    Computes tubiness, surfaceness, and blobbiness for a merged skeleton region.
    region_voxels: list of 3D coordinates (x,y,z) of voxels in the region R.
    skeleton_segment: list of 3D coordinates of the skeleton points for R.
    pre_seg_mask: binary volume or function to check if a point is in the object.
    """
    if not region_voxels:
        return (0.0, 0.0, 0.0)

    # 1. Compute Surface Voxels and Distance Map
    # Identify surface voxels: voxels in region that have at least one neighbor outside the region
    surface_voxels = []
    for v in region_voxels:
        if not is_interior(v, region_voxels): # [G] Helper to check if all neighbors are in region
            surface_voxels.append(v)
    
    # Compute distance map D for surface voxels to skeleton
    # D(v) = min distance from v to any point in skeleton_segment
    distances = []
    for v in surface_voxels:
        min_dist = infinity
        for s in skeleton_segment:
            dist = euclidean_distance(v, s)
            if dist < min_dist:
                min_dist = dist
        distances.append(min_dist)
    
    if not distances:
        return (0.0, 0.0, 0.0)

    # 2. Compute Tubiness
    mean_d = sum(distances) / len(distances)
    variance_d = sum((d - mean_d)**2 for d in distances) / len(distances)
    std_d = sqrt(variance_d)
    tubiness = 1.0 / max(std_d, 1.0)

    # 3. Compute Surfaceness
    # Planarity
    bbox = compute_oriented_bounding_box(region_voxels)
    sides = sorted(bbox.sides) # [G] Returns list of 3 side lengths sorted ascending
    b_s = sides[0]
    b_m = sides[1]
    
    if b_s == 0:
        planarity = 0.0
    else:
        ratio = b_m / b_s
        planarity = clamp((1.0 / 5.0) * ratio - 1.0, 0.0, 1.0)

    # Convexity
    # Select arbitrary pairs of surface points (e.g., random subset or all pairs if small)
    # For implementation, use a fixed number of random pairs if surface_voxels is large
    num_pairs = min(100, len(surface_voxels) * (len(surface_voxels) - 1) // 2)
    if num_pairs == 0:
        convexity = 0.0
    else:
        inside_count = 0
        total_count = 0
        # [G] Simplified pair selection for pseudocode
        pairs = sample_pairs(surface_voxels, num_pairs)
        for p1, p2 in pairs:
            line_segment = generate_line_segment(p1, p2)
            # Count voxels on line segment that are inside the region
            voxels_on_line = get_voxels_on_line(line_segment)
            inside_voxels = [v for v in voxels_on_line if v in region_voxels]
            if len(voxels_on_line) > 0:
                fraction = len(inside_voxels) / len(voxels_on_line)
                inside_count += fraction
                total_count += 1
        
        if total_count > 0:
            convexity = inside_count / total_count
        else:
            convexity = 0.0

    surfaceness = (planarity + convexity) * (1.0 - tubiness)

    # 4. Compute Blobbiness
    # Variance of region
    center_of_mass = compute_center_of_mass(region_voxels)
    variance_r = sum(euclidean_squared_distance(v, center_of_mass) for v in region_voxels)
    
    # Variance of sphere with same volume
    vol = len(region_voxels)
    # sigma^2_sph(vol) = (4/5) * pi * ( (3/(4*pi)) * vol )^(5/3)
    radius_factor = (3.0 / (4.0 * pi)) * vol
    sigma_sph_sq = (4.0 / 5.0) * pi * (radius_factor ** (5.0 / 3.0))
    
    if variance_r == 0:
        blobbiness = 1.0 # Perfectly compact (point)
    else:
        blobbiness = sigma_sph_sq / variance_r

    return (tubiness, surfaceness, blobbiness)

def is_interior(v, region_voxels):
    # Check if all 6 neighbors of v are in region_voxels
    neighbors = get_6_neighbors(v)
    region_set = set(region_voxels) # [G] For O(1) lookup
    return all(n in region_set for n in neighbors)

def compute_center_of_mass(voxels):
    n = len(voxels)
    if n == 0: return (0,0,0)
    sum_x = sum(v[0] for v in voxels)
    sum_y = sum(v[1] for v in voxels)
    sum_z = sum(v[2] for v in voxels)
    return (sum_x/n, sum_y/n, sum_z/n)

def euclidean_squared_distance(p1, p2):
    return (p1[0]-p2[0])**2 + (p1[1]-p2[1])**2 + (p1[2]-p2[2])**2
```

## Subtopics

### Barycentric Shape Selection Widget
A user interface widget using an equilateral triangle based on barycentric coordinates to represent the 4D transfer function space (tubiness, surfaceness, blobbiness, intensity) and allow intuitive selection of shape classes [S2]. Place a marker inside the triangle for each computed shape segment, with position determined by the three normalized shape classifiers [S2]. The user selects markers to constrain shape classes (first stage), then uses a conventional 1D transfer function for intensity-based optical properties (second stage) [S2]. If the data has no planar structures, surfaceness can be omitted, collapsing the triangle to a line [S2].

## Operating characteristics
- **Time Complexity**: Dominated by skeleton computation (potential field method is slow) and region merging [S2]. Preprocessing time is not a main issue for the use case [S2].
- **Memory Complexity**: Requires storage for the volume, pre-segmentation mask, skeleton segments, and region assignments [S2].
- **Performance**: Tested on synthetic and real-world medical data (e.g., NCAT phantom, angiography) [S2].
- **Strengths**: Intuitive user interface; distinguishes features with overlapping intensities (e.g., vessels vs. bones); robust to imperfect pre-segmentation via normalization [S2].
- **Limitations**: Relies on quality of curve-skeletons; highly degenerated tubular regions may be misclassified; cavities in pre-segmentation can disturb skeleton computation; potential field method is slow [S2].
- **When to use**: When features have similar intensities but distinct shapes (e.g., medical imaging with contrast agents) [S2].
- **When not to use**: When features are defined primarily by texture or when high-speed interactive preprocessing is required [S2].

## Background
The concept of axial description of shape (skeleton) was proposed in 2D by Blum [S2]. Binford proposed generalized cylinders in 3D [S2]. Cornea et al. introduced hierarchical curve-skeletons using repulsive force fields [S2]. The specific shape-based transfer function method was proposed by Praßni et al. [S2]. Sato et al. used eigenvalue analysis for per-voxel classification [S1, unofficial].

## Sources
[S1] MULTIVARIATE TRANSFER FUNCTION DESIGN, https://www.sci.utah.edu/~lzhou/dissertation_final_zhoul.pdf, fetched 2026-10-27 (unofficial)
[S2] Shape-based Transfer Functions for Volume Visualization, https://www.diva-portal.org/smash/get/diva2:623421/FULLTEXT01.pdf, fetched 2026-10-29
