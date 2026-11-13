# Topological Voxelization with Intersection Targets

Also called: Intersection Target Voxelization

## Overview
Topological Voxelization with Intersection Targets determines which voxels are intersected by input geometry by testing primitives against specific geometric targets within each voxel, rather than relying on geometric proximity measures [S1, unofficial]. It sets itself apart from sibling methods like Point-Tessellated Voxelization by guaranteeing topological properties (connectivity and separability) through the dimensionality of these targets, making it independent of input tessellation and suitable for curved primitives [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A set of geometric primitives (points, curves, surfaces) and a voxel grid; the effective dimension $d_{eff}$ of the input geometry must be known or estimated [S1, unofficial].
*   **Output:** A set of voxels $S_d$ included in the voxelization [S1, unofficial].

**Data Structures:**
*   **Voxel Grid:** A 3D array or sparse structure representing the voxel space; each voxel $V$ is identified by integer coordinates $(i, j, k)$ [S1, unofficial].
*   **Intersection Target:** A geometric shape defined within the unit cube of a voxel; its dimension depends on $d_{eff}$ and the desired connectivity [S1, unofficial].
*   **Supercover:** A bounding volume or list of voxels that contain the input geometry's supercover (the set of voxels intersected by the input geometry's thickened version) [S1, unofficial]. Only voxels in the supercover need to be tested [S1, unofficial].

**Steps:**
1.  **Determine Effective Dimension:** Identify the effective dimension $d_{eff}$ of the input geometry (0D for points, 1D for curves/strands, 2D for surfaces, 3D for volumes) [S1, unofficial].
2.  **Select Intersection Target:** Choose an intersection target shape within the voxel corresponding to $d_{eff}$ and the desired connectivity (e.g., 6-connected, 18-connected, or 26-connected in 3D) [S1, unofficial].
    *   For $d_{eff}=1$ (e.g., curves), the target must be at least 2D to ensure connectivity [S1, unofficial].
    *   For $d_{eff}=3$ (e.g., volumes), the target is typically 0D (a point, usually the center) [S1, unofficial].
3.  **Identify Supercover Voxels:** Determine the set of voxels belonging to the supercover of the input geometry; intersection targets in voxels outside this set cannot intersect the input and can be skipped [S1, unofficial].
4.  **Test Intersections:** For each voxel $V$ in the supercover, test if the input primitive intersects the target $T_V$ [S1, unofficial].
5.  **Include Voxels:** If an intersection occurs, include $V$ in the output set $S_d$ [S1, unofficial].

**Formulas:**
*   **Intersection Condition:** $V \in S_d \iff \text{Input} \cap T_V \neq \emptyset$ [S1, unofficial].
*   **Perturbation:** To resolve degenerate intersections, apply an infinitesimal perturbation vector $\vec{p}$ to the input or target. A common example is $\vec{p} = (\epsilon, 2\epsilon, 3\epsilon)$ where $\epsilon$ is a small positive number [S1, unofficial].

**Parameters:**
*   **Effective Dimension ($d_{eff}$):** Determines the dimension of the intersection target [S1, unofficial].
*   **Perturbation Vector ($\vec{p}$):** Used to resolve boundary degeneracies. Example: $(\epsilon, 2\epsilon, 3\epsilon)$ [S1, unofficial].
*   **Connectivity:** The desired adjacency property (e.g., 6, 18, 26 in 3D) [S1, unofficial].

**Pitfalls:**
*   **Mismatched Dimensions:** Using a 0D target (point) for 1D input (curves) results in intersection only by chance, failing to preserve connectivity [S1, unofficial]. The target dimension must be at least 2D for 1D inputs to establish 6 or 26-connectivity [S1, unofficial].
*   **Degenerate Intersections:** Input boundaries touching voxel boundaries can cause spurious holes or thick results [S1, unofficial]. This is resolved by testing shifts in $+x, +y, +z$ directions until a non-degenerate intersection is found [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def topological_voxelization(input_primitives, voxel_grid, effective_dimension, connectivity):
    # Step 1: Determine intersection target shape based on effective dimension and connectivity
    target_shape = get_intersection_target_shape(effective_dimension, connectivity)
    
    # Step 2: Identify voxels in the supercover of the input
    supercover_voxels = compute_supercover(input_primitives, voxel_grid)
    
    # Step 3: Define perturbation vector for degenerate cases
    epsilon = 1e-6
    perturbation_vectors = [
        (epsilon, 2*epsilon, 3*epsilon),
        (epsilon, 0, 0),
        (0, epsilon, 0),
        (0, 0, epsilon)
    ]
    
    output_voxels = set()
    
    for voxel in supercover_voxels:
        voxel_bounds = voxel.get_bounds()
        
        # Create the intersection target within this voxel
        target = create_target(target_shape, voxel_bounds)
        
        intersection_found = False
        
        # Test for intersection, handling degeneracies
        for p in perturbation_vectors:
            # Apply perturbation to input primitives or target
            perturbed_primitives = [p.translate(perturbation) for p in input_primitives]
            if any(primitive.intersects(target) for primitive in perturbed_primitives):
                intersection_found = True
                break
        
        if intersection_found:
            output_voxels.add(voxel)
            
    return output_voxels

def get_intersection_target_shape(effective_dimension, connectivity):
    # [Comment: Standard operation to map (effective_dimension, connectivity) to a geometric target shape]
    return None

def compute_supercover(input_primitives, voxel_grid):
    # [Comment: Standard operation to compute the set of voxels intersected by the input geometry's thickened version]
    return []

def create_target(target_shape, voxel_bounds):
    # [Comment: Standard operation to instantiate the target_shape within voxel_bounds]
    return None
```

## Subtopics

### [Intersection Target Method](5655-intersection-target-method.md)
The Intersection Target Method is a topological voxelization strategy for effectively 1D input (lines or curves) that guarantees 8-connectivity (adjacent voxels share an edge or corner) and 4-separability (no 4-connected path of background voxels separates foreground components). It differs from geometric proximity methods by including a voxel if and only if the input intersects a specific geometric "target" shape within that voxel, ensuring topological correctness through the connectivity of these targets.

## Operating characteristics
*   **Time Complexity:** Linear with respect to the number of voxels in the supercover and the number of primitives tested against each target [S1, unofficial].
*   **Memory Complexity:** Linear with respect to the number of voxels in the supercover [S1, unofficial].
*   **Strengths:** Guarantees topological correctness (connectivity and separability); independent of input tessellation; supports curved primitives; simple proofs of properties [S1, unofficial].
*   **Limitations:** Requires knowledge of the effective dimension of the input; may produce thicker voxelizations than proximity-based methods if the target is large; degenerate cases require careful handling [S1, unofficial].
*   **When to Use:** When topological correctness is critical (e.g., for simulation or connectivity-dependent applications); when input geometry is curved or tessellation-independent representation is desired [S1, unofficial].

## Background
*   **Origin:** Proposed by Samuli Laine at NVIDIA [S1, unofficial].
*   **Paper:** "A Topological Approach to Voxelization" [S1, unofficial].
*   **History:** The method builds on the concept of supercover [COK95] and aims to provide a rigorous topological foundation for voxelization, addressing issues with proximity-based methods like Varadhan et al.'s max-norm distance approach [VKK*03] [S1, unofficial].

## Sources
[S1] A Topological Approach to Voxelization - NVIDIA, https://research.nvidia.com/sites/default/files/pubs/2013-06_A-Topological-Approach/laine2013egsr_paper.pdf, fetched 2026-10-29 (unofficial)
