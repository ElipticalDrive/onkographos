# Adaptive Volume Subdivision
Also called: Adaptive Octree Subdivision
## Overview
Adaptive Volume Subdivision manages the resolution of irradiance volumes by using an octree structure to store samples, subdividing only in regions where irradiance changes rapidly and maintaining coarse resolution where it changes slowly [S1, unofficial]. It solves the problem of uniform grids becoming unwieldy for large, complex scenes by allowing variable granularity [S1, unofficial]. It differs from uniform grid approaches by dynamically adjusting voxel size based on scene complexity and irradiance gradients, and from static hierarchical methods by relying on specific heuristics like scene depth to drive subdivision decisions [S1, unofficial].
## How it works
**Inputs and Outputs:**
*   Specify a minimum and maximum subdivision level for the octree, and allow thresholds to be specified for each subdivision heuristic [S1, unofficial].
*   The output is an octree of irradiance samples where leaf nodes represent voxels of varying sizes, optimized for storage and retrieval of irradiance data [S1, unofficial].

**Data Structures:**
*   **Octree Node:** Contains a flag indicating if it is a leaf or internal node, and a list of child pointers [S1, unofficial]. Leaf nodes store irradiance samples, typically spherical harmonic coefficients [S1, unofficial].
*   **Voxel:** The spatial unit represented by a leaf node. Its size is determined by its depth in the octree [S1, unofficial].

**Steps:**
1.  **Initialize Octree:** Create the root node of the octree [S1, unofficial].
2.  **Flag Nodes for Subdivision:** Iterate through the scene or existing samples to determine which voxels require further subdivision using one or more heuristics [S1, unofficial]:
    *   **Geometry Check:** Subdivide any voxels that contain scene geometry [S1, unofficial].
    *   **Harmonic Mean of Scene Depth:** Compute the harmonic mean of scene depth at a sample point to determine if subdivision is needed [S1, unofficial].
3.  **Allocate Subnode Tiles:** For each flagged node, allocate child nodes up to the maximum subdivision level, respecting the heuristics [S1, unofficial].
4.  **Initialize New Nodes:** Initialize the new child nodes with appropriate data structures [S1, unofficial].
5.  **Sample the Volume:** Search the octree for the voxel containing the object's centroid or sample point [S1, unofficial]. Use surrounding samples to determine irradiance via trilinear interpolation of spherical harmonic coefficients or a weighted sum of surrounding samples weighted by $1/distance$ [S1, unofficial].
6.  **Cull Redundant Samples:** After fully sampling the volume, iterate through the octree to reject redundant samples; if a voxel has been subdivided and its children do not differ significantly from the parent, the children's samples may be culled [S1, unofficial].

**Formulas:**
*   **Harmonic Mean of Scene Depth:**
    $$ HM = \frac{N}{\sum_{i=1}^{N} \frac{1}{d_i}} $$
    Where $N$ is the total number of rays fired, and $d_i$ is the distance the $i$-th ray traveled before intersecting scene geometry [S1, unofficial].
*   **Subdivision Condition:** If neighboring irradiance samples are further away than the computed Harmonic Mean ($HM$), their associated voxels should be subdivided [S1, unofficial].

**Parameters:**
*   **Min/Max Subdivision Level:** Defines the range of octree depths allowed [S1, unofficial].
*   **Heuristic Thresholds:** Values used to determine if the difference between parent and children, or the scene depth, warrants subdivision [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Infinite Depth:** The harmonic mean is chosen over the arithmetic mean because large depth values (due to infinite depth if no geometry exists in a given direction) would quickly bias the arithmetic mean to a large value [S1, unofficial].
*   **Voxel Geometry Containment:** Determining if a voxel contains scene geometry can be done by sampling scene depth at voxel corners using a cubemap, or by using occlusion queries with the camera at the voxel center [S1, unofficial].

**Pseudocode:**
```python
import math

def adaptive_volume_subdivision(scene, min_level, max_level, depth_threshold):
    class Node:
        def __init__(self, level, center, size):
            self.level = level
            self.center = center
            self.size = size
            self.children = None
            self.sample = None

    root = Node(level=0, center=(0, 0, 0), size=1.0)

    def compute_harmonic_mean_depth(center):
        # GPU: render scene into each face of a floating point cubemap
        # using a shader that outputs 1/depth. Read cubemap back to memory. [S1, unofficial]
        # Compute harmonic mean of distances from cubemap data. [S1, unofficial]
        # Not given in the sources how cubemap data is converted to distances. [G]
        distances = [1.0, 2.0, 3.0] # Example cubemap ray distances [G]
        N = len(distances)
        if N == 0: return float('inf')
        sum_inv = sum(1.0 / d for d in distances if d > 0)
        return N / sum_inv if sum_inv > 0 else float('inf') # [S1, unofficial]

    def voxel_contains_geometry(node):
        # Method 1: Sample scene depth at voxel corners using cubemap texels. [S1, unofficial]
        # Method 2: Occlusion query: camera at voxel center, render voxel quads then scene. [S1, unofficial]
        # If any draw call passes, geometry exists inside. [S1, unofficial]
        return False # [G]

    def get_neighboring_samples(node):
        # Search octree for surrounding samples. [S1, unofficial]
        return [] # [G]

    def compute_irradiance(center):
        # Search octree for voxel containing center. [S1, unofficial]
        # Use surrounding samples: trilinear interpolation of spherical harmonic coefficients
        # or weighted sum of surrounding samples weighted by 1/distance. [S1, unofficial]
        return 0.0 # [G]

    def cull_redundant_samples(node):
        # After fully sampling, reject redundant samples. [S1, unofficial]
        # If subdivided and children don't differ enough from parent, cull children. [S1, unofficial]
        if node.children is None: return
        for child in node.children:
            if child.sample is not None and node.sample is not None:
                if abs(child.sample - node.sample) < depth_threshold:
                    child.sample = None
            cull_redundant_samples(child)

    def process_node(node):
        if node.level >= max_level: return
        if voxel_contains_geometry(node):
            node.children = [Node(node.level + 1, (0,0,0), node.size / 2) for _ in range(8)] # [G]
            for child in node.children: process_node(child) # [G]
            return
        hm = compute_harmonic_mean_depth(node.center)
        neighbors = get_neighboring_samples(node)
        needs_subdivision = any(math.dist(node.center, n.center) > hm for n in neighbors)
        if needs_subdivision:
            node.children = [Node(node.level + 1, (0,0,0), node.size / 2) for _ in range(8)] # [G]
            for child in node.children: process_node(child) # [G]
        else:
            node.sample = compute_irradiance(node.center) # [S1, unofficial]

    process_node(root)
    cull_redundant_samples(root)
    return root
## Subtopics

### Harmonic Mean of Scene Depth
A heuristic to determine when subdivision is needed by computing the harmonic mean of scene depth at a sample point; areas with more geometry have more rapidly changing irradiance [S1, unofficial]. It is chosen over the arithmetic mean to avoid bias from infinite depth values [S1, unofficial]. It is implemented by shooting rays from the sample position, computing the harmonic mean of distances to intersection, and using this as an upper bound for sample usefulness; if neighboring samples are further away than this bound, subdivision is triggered [S1, unofficial]. On the GPU, this is done by rendering the scene into a floating-point cubemap with a shader outputting $1/depth$, then reading the cubemap back to find the mean [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources. [S1, unofficial]
*   **Memory Complexity:** Not given in the sources. [S1, unofficial]
*   **Measured Performance:** Not given in the sources. [S1, unofficial]
*   **Scaling:** Not given in the sources. [S1, unofficial]
*   **Strengths:** Allows finer resolution in areas with quickly changing irradiance and coarser resolution in slowly changing areas, avoiding the unwieldiness of uniform grids for large, complex scenes [S1, unofficial].
*   **Limitations:** Determining which areas need subdivision is a challenging problem [S1, unofficial].
*   **When to use:** When dealing with large, complex scenes where irradiance varies significantly across different regions [S1, unofficial].
## Background
The method was presented in the GDC 2005 paper "Irradiance Volumes for Games" by Chris Oat [S1, unofficial]. It was developed to address the limitations of uniform grids in game engines, specifically for global illumination techniques like irradiance volumes [S1, unofficial].
## Sources
[S1] Irradiance Volumes for Games, https://chrisoat.com/papers/Oat_GDC2005_IrradianceVolumesForGames.pdf, fetched 2026-10-28 (unofficial)
