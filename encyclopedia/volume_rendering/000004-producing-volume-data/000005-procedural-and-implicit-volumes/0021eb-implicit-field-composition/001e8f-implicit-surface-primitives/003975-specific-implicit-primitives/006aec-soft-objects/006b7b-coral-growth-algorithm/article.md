# Coral Growth Algorithm

Also called: Parametrized growth algorithm, Tree of spheres construction

## Overview
The Coral Growth Algorithm is a procedural method for generating organic, branching volume data by constructing a hierarchical tree of spheres [S1, book]. It solves the problem of creating complex, naturalistic coral geometries that serve as implicit fields for soft object rendering [S1, book]. It is distinguished from generic tree generators by its specific use of "soft object" contouring, where the final shape is defined by the sum of spherically symmetric density functions rather than explicit mesh boundaries, and by its strict layer-by-layer generation process governed by collision constraints based on genealogical proximity [S1, book].

## How it works
The algorithm constructs a volume density field defined by a set of spheres. The structure is determined by the centers of these spheres and their outer radii, where the density functions become zero [S1, book]. The resulting contour surface is rendered as a soft object [S1, book].

### Inputs and Outputs
**Inputs:**
*   13 scalar constants (see Parameters).
*   2 boolean flags for optional features.
*   A random seed.

**Outputs:**
*   A list of spheres, each defined by a center position $(x, y, z)$ and a radius $r$.
*   The spheres are organized in a tree structure with a root and generations (layers).

### Data Structures
The core data structure is a list of sphere objects. Each sphere object contains:
*   `center`: A 3D vector $(x, y, z)$.
*   `radius`: A scalar float $r$.
*   `parent_index`: An integer index pointing to the parent sphere in the list (or -1 for the root).
*   `generation`: An integer indicating the layer index $n$ (0 for root).
*   `children`: A list of indices pointing to child spheres.

### Parameters
The following 13 constants define the shape [S1, book]:

1.  `num_levels`: Total number of levels in the tree.
2.  `a`: Initial radius for the root sphere ($n=0$).
3.  `b`: Radius decrease factor per level ($b < 1$).
4.  `bondfactor`: Multiplier for the distance between parent and child centers.
5.  `g`: Number of generations back for which a common ancestor allows a lenient collision factor.
6.  `adjacent`: Collision factor for spheres with a common ancestor within $g$ generations.
7.  `nonadjacent`: Collision factor for spheres with no close common ancestor.
8.  `min_dist_mult`: Multiple of `bond_r(n)` used to increment the minimum distance from the root node.
9.  `min_z_mult`: Multiple of `bond_r(n)` used to increment the minimum z-value.
10. `z_low`: Truncation level for the unit sphere of directions (removes directions with $z < z_{low}$).
11. `d`: Base constant in the `max_sphere` formula.
12. `tryfac`: Factor for calculating `maxtrials`.
13. `seed`: Initial seed for the random number generator.

Two optional features can be enabled:
1.  `interpolate_extra`: If true, interpolates an extra sphere between parent and child if distance is large.
2.  `continue_branch`: If true, attempts to continue the branch curve to a grandchild if a child has no other children.

### Formulas
*   **Sphere Radius:** The radius of a sphere at generation $n$ is:
    $$r(n) = a \cdot b^n$$
    [S1, book]

*   **Bond Distance:** The distance between the center of a new sphere (generation $n$) and its parent (generation $n-1$) is:
    $$bond\_r(n) = bondfactor \cdot (r(n) + r(n-1))$$
    [S1, book]

*   **Maximum Spheres per Layer:** The maximum number of spheres allowed in layer $n$ is:
    $$max\_sphere(n) = \lfloor 2 \cdot d^n \rfloor$$
    Note: The source text states "e is always 2" and "floor (c • dh)", implying $c=2$ and the exponent is $n$ (referred to as $h$ in the garbled text, but contextually the level index) [S1, book].

*   **Max Trials:** The maximum number of random trials before giving up on a layer is:
    $$maxtrials = tryfac \cdot max\_sphere(n)$$
    [S1, book]

*   **Collision Distance:** The minimum squared distance between a trial sphere (level $n$) and an accepted sphere (level $i$) depends on their genealogical relationship:
    *   If they share a common ancestor within $g$ generations (i.e., $\max(0, n-g, i-g)$ is the level of the common ancestor):
        $$dist\_min = adjacent \cdot (r(n) + r(i))$$
    *   Otherwise (no close common ancestor):
        $$dist\_min = nonadjacent \cdot (r(n) + r(i))$$
    [S1, book]

*   **Shape Constraints:**
    *   Minimum distance from root: $min\_dist(n) = min\_dist\_mult \cdot bond\_r(n)$
    *   Minimum z-value: $min\_z(n) = min\_z\_mult \cdot bond\_r(n)$
    [S1, book]

### Steps
1.  **Initialize:** Create the root sphere (generation 0) at the origin (or specified start position) with radius $r(0) = a$. Set `current_generation = 0`.
2.  **Loop Generations:** While `current_generation < num_levels`:
    a.  Calculate $r(current\_generation)$ and $max\_sphere(current\_generation)$.
    b.  Initialize `accepted_count = 0` and `trial_count = 0`.
    c.  **Generate Spheres:** While `accepted_count < max_sphere(current_generation)` and `trial_count < maxtrials`:
        i.  Select a parent sphere randomly from generation `current_generation - 1`.
        ii. Generate a random direction vector on a unit sphere. Reject if the z-component is $< z_{low}$. Normalize the direction.
        iii. Calculate the trial center: $center_{trial} = center_{parent} + direction \cdot bond\_r(current\_generation)$.
        iv. **Check Shape Constraints:**
            - If distance from root $> min\_dist(current\_generation)$, reject.
            - If $z < min\_z(current\_generation)$, reject.
        v.  **Check Collision:** For every accepted sphere in the entire tree:
            - Calculate squared distance $d^2$ between $center_{trial}$ and accepted sphere center.
            - Determine if they share a common ancestor within $g$ generations.
            - Calculate $dist\_min$ using the appropriate factor (`adjacent` or `nonadjacent`).
            - If $d^2 < dist\_min^2$, reject.
        vi. If all checks pass, add the trial sphere to the accepted list and increment `accepted_count`.
        vii. Increment `trial_count`.
    d.  **Optional Feature 1 (Interpolation):** If `interpolate_extra` is enabled, iterate through all parent-child pairs. If the distance between parent and child is large, insert an intermediate sphere to maintain density and smooth curvature.
    e.  **Optional Feature 2 (Branch Continuation):** If `continue_branch` is enabled, for any child sphere with no children, attempt to generate a grandchild. Generate a random position for the grandchild. If this position violates collision constraints, revise it to follow the smooth continuation of the branch curve from the child, provided the new position does not violate constraints.
    f.  Increment `current_generation`.

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import random

class Sphere:
    def __init__(self, center, radius, parent_index, generation):
        self.center = center
        self.radius = radius
        self.parent_index = parent_index
        self.generation = generation
        self.children = []

def generate_coral(params):
    # Unpack parameters
    num_levels = params['num_levels']
    a = params['a']
    b = params['b']
    bondfactor = params['bondfactor']
    g = params['g']
    adjacent = params['adjacent']
    nonadjacent = params['nonadjacent']
    min_dist_mult = params['min_dist_mult']
    min_z_mult = params['min_z_mult']
    z_low = params['z_low']
    d = params['d']
    tryfac = params['tryfac']
    seed = params['seed']
    interpolate_extra = params['interpolate_extra']
    continue_branch = params['continue_branch']

    random.seed(seed)
    spheres = []
    
    # Initialize root
    root = Sphere(center=[0, 0, 0], radius=a, parent_index=-1, generation=0)
    spheres.append(root)
    
    for n in range(1, num_levels):
        r_n = a * (b ** n)
        r_n_minus_1 = a * (b ** (n - 1))
        bond_r_n = bondfactor * (r_n + r_n_minus_1)
        
        max_sphere_n = math.floor(2 * (d ** n))
        maxtrials = tryfac * max_sphere_n
        
        accepted_count = 0
        trial_count = 0
        
        while accepted_count < max_sphere_n and trial_count < maxtrials:
            # Select random parent from previous generation
            parent_idx = random.randint(0, len(spheres) - 1)
            # Ensure parent is from previous generation
            while spheres[parent_idx].generation != n - 1:
                parent_idx = random.randint(0, len(spheres) - 1)
            
            parent = spheres[parent_idx]
            
            # Generate random direction on truncated unit sphere
            while True:
                # Generate random point on unit sphere
                u = random.random()
                v = random.random()
                theta = 2 * math.pi * u
                phi = math.acos(2 * v - 1)
                x = math.sin(phi) * math.cos(theta)
                y = math.sin(phi) * math.sin(theta)
                z = math.cos(phi)
                
                if z >= z_low:
                    break
            
            direction = [x, y, z]
            
            # Calculate trial center
            trial_center = [
                parent.center[0] + direction[0] * bond_r_n,
                parent.center[1] + direction[1] * bond_r_n,
                parent.center[2] + direction[2] * bond_r_n
            ]
            
            # Check shape constraints
            dist_from_root = math.sqrt(sum(c**2 for c in trial_center))
            min_dist = min_dist_mult * bond_r_n
            min_z = min_z_mult * bond_r_n
            
            if dist_from_root < min_dist or trial_center[2] < min_z:
                trial_count += 1
                continue
            
            # Check collision
            collision = False
            for other in spheres:
                dx = trial_center[0] - other.center[0]
                dy = trial_center[1] - other.center[1]
                dz = trial_center[2] - other.center[2]
                dist_sq = dx*dx + dy*dy + dz*dz
                
                # Determine common ancestor generations back
                # Find common ancestor level
                # This requires traversing up the tree, which is O(N) per check
                # For efficiency, we assume a helper function get_common_ancestor_level
                common_level = get_common_ancestor_level(parent_idx, other.parent_index, spheres)
                
                # Check if common ancestor is within g generations
                # The generation of the common ancestor is common_level
                # The current generation is n, other generation is other.generation
                # If max(0, n-g, other.generation-g) == common_level, they are adjacent
                # Simplified: if the depth difference to common ancestor is <= g for both
                depth_diff_trial = n - common_level
                depth_diff_other = other.generation - common_level
                
                if depth_diff_trial <= g and depth_diff_other <= g:
                    dist_min = adjacent * (r_n + other.radius)
                else:
                    dist_min = nonadjacent * (r_n + other.radius)
                
                if dist_sq < dist_min * dist_min:
                    collision = True
                    break
            
            if not collision:
                new_sphere = Sphere(center=trial_center, radius=r_n, parent_index=parent_idx, generation=n)
                spheres.append(new_sphere)
                spheres[parent_idx].children.append(len(spheres) - 1)
                accepted_count += 1
            
            trial_count += 1
        
        # Optional Feature 1: Interpolate extra spheres
        if interpolate_extra:
            interpolate_spheres(spheres, n)
            
        # Optional Feature 2: Continue branch
        if continue_branch:
            continue_branches(spheres, n)

    return spheres

def get_common_ancestor_level(idx1, idx2, spheres):
    # Traverse up from idx1 and idx2 to find common ancestor
    # Returns the generation level of the common ancestor
    # If no common ancestor (should not happen in a tree), return -1
    path1 = []
    curr = idx1
    while curr != -1:
        path1.append(curr)
        curr = spheres[curr].parent_index
    
    path2 = []
    curr = idx2
    while curr != -1:
        path2.append(curr)
        curr = spheres[curr].parent_index
    
    # Find first common node
    set1 = set(path1)
    for node in path2:
        if node in set1:
            return spheres[node].generation
    return -1

def interpolate_spheres(spheres, current_gen):
    # Iterate through spheres of current_gen
    # If distance to parent is large, insert intermediate sphere
    # Implementation details omitted as per source description "interpolates an extra sphere"
    pass

def continue_branches(spheres, current_gen):
    # Iterate through spheres of current_gen
    # If sphere has no children, try to add grandchild
    # Implementation details omitted as per source description
    pass
```

### Edge Cases and Pitfalls
*   **Precision:** The collision check involves squared distances and floating-point comparisons. Small errors in position accumulation may lead to false positives or negatives in collision detection [S1, book].
*   **Boundaries:** The truncation of the direction sphere ($z < z_{low}$) ensures upward growth but may lead to sparse regions if $z_{low}$ is too high [S1, book].
*   **Degenerate Input:** If `tryfac` is too low, the algorithm may fail to fill the maximum number of spheres for a layer, resulting in a sparse structure [S1, book].
*   **Ancestor Lookup:** The collision check requires finding the common ancestor level, which can be computationally expensive if not optimized. The source implies a direct check based on generation count $g$ [S1, book].

## Operating characteristics
*   **Time Complexity:** The generation of each layer involves up to `maxtrials` trials. Each trial requires checking against all previously accepted spheres for collision. Thus, the complexity is roughly $O(N^2)$ where $N$ is the total number of spheres, due to the pairwise collision checks [S1, book].
*   **Memory Complexity:** $O(N)$ to store the sphere data and tree structure [S1, book].
*   **Performance:** The algorithm is procedural and does not require pre-computation of a grid, making it memory-efficient for high-resolution details. However, the quadratic collision check can become slow for large numbers of spheres [S1, book].
*   **Strengths:** Produces organic, natural-looking coral shapes with smooth contours. The soft object rendering allows for easy shading and bump mapping [S1, book].
*   **Limitations:** The shape is controlled by a fixed set of 13 parameters, which may limit the variety of shapes compared to more complex L-systems or fractal methods. The random nature means reproducibility requires a fixed seed [S1, book].

## Background
The Coral Growth Algorithm was developed by Nelson Max and Geoff Wyvill [S1, book]. It was introduced in the paper "Shapes and Textures for Rendering Coral" [S1, book]. The method is part of the Katachi system, which uses soft objects for modeling [S1, book]. The algorithm is similar to other parametrized growth algorithms used for building trees, such as those by Bloomenthal, Aono & Kunii, De Reffye et al., Viennot et al., and Prusinkiewicz [S1, book]. The resulting shapes were initially examined in hidden line sketches and then rendered using ray tracing with generalized volume textures for shading and bump mapping [S1, book].

## Sources
[S1] Shapes and textures for rendering coral, https://digital.library.unt.edu/ark:/67531/metadc1102376/?q=3d texture based volume rendering, fetched 2026-10-28 (book)
