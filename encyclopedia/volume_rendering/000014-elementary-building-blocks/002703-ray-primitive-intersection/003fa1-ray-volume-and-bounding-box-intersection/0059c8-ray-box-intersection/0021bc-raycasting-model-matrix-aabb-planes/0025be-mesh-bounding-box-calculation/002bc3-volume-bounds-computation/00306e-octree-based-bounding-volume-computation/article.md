# Octree-based Bounding Volume Computation
Also called: Convex Hull from Straddling Cubes, Octree-based Bounding Volume

## Overview
This method computes a convex bounding volume for a signed distance field (SDF) by adaptively decomposing space into an octree and extracting the convex hull from cubes that straddle the surface [S1, S2]. It serves as an alternative to Sphere Carving, typically requiring significantly more SDF evaluations to obtain a good convex hull, but potentially converging faster for shapes with significant anisotropy in $f$ values [S1, S2]. Unlike Sphere Carving, this approach does not provide bounding guarantees that the resulting volume encloses the entire surface [S1, S2].

## How it works
**Inputs:**
- A signed distance function $f(\mathbf{x})$ [S1, S2].
- An initial large cube $C_{\text{root}}$ enclosing the region of interest [S1, S2].

**Outputs:**
- A convex hull $H$ approximating the surface boundary [S1, S2].

**Data Structures:**
- **Octree Node:** Represents a cube in 3D space [S1, S2].
  - `center`: 3D vector $\mathbf{c}$, the geometric center of the cube [S1, S2].
  - `diagonal`: scalar $d$, the length of the cube's space diagonal [S1, S2].
  - `children`: list of 8 child nodes or `None` if leaf [S1, S2].
  - `vertices`: list of 8 corner points of the cube [S1, S2].

**Algorithm Steps:**
1. **Initialization:** Create the root node with the initial large cube $C_{\text{root}}$ [S1, S2].
2. **Recursive Subdivision:** For each node, evaluate $f(\mathbf{c})$ at the cube center [S1, S2]. If $f(\mathbf{c}) \ge d$, the cube does not straddle the surface; stop subdivision for this branch [S1, S2]. If $f(\mathbf{c}) < d$, the cube may straddle the surface; subdivide it into 8 equal child cubes and process each recursively [S1, S2].
3. **Straddling Detection:** Identify straddling cubes as the octree leaves reached during traversal where $f(\mathbf{c}) < d$ [S1, S2].
4. **Convex Hull Construction:** Collect the eight vertices of all detected straddling cubes [S1, S2]. Compute the convex hull $H$ of this point set [S1, S2].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def compute_octree_bounding_volume(f, initial_cube):
    """
    Computes a convex bounding volume for an SDF f using an octree.
    
    Args:
        f: Signed distance function.
        initial_cube: A cube object with attributes 'center' (vec3), 
                      'diagonal' (float), 'side_length' (float), and 
                      'vertices' (list of 8 vec3).
        
    Returns:
        H: Convex hull of the straddling cubes' vertices.
    """
    straddling_vertices = []
    
    def recurse(cube):
        f_val = f(cube.center)
        
        # If f(c) >= diagonal, the cube does not straddle the surface.
        if f_val >= cube.diagonal:
            return
            
        # [G] Sources do not specify a termination condition other than f(c) >= d.
        # In practice, a minimum cube size or max depth is required.
        if cube.side_length < 1e-6:
            straddling_vertices.extend(cube.vertices)
            return
            
        # Subdivide into 8 children
        for child in cube.subdivide():
            recurse(child)
            
    recurse(initial_cube)
    
    # [G] Sources do not specify the convex hull algorithm.
    H = convex_hull(straddling_vertices)
    return H
```

**Parameters:**
- Initial cube size: Not given in the sources. [S1, S2]
- Max depth / Min cube size: Not given in the sources. [S1, S2]

**Edge Cases and Pitfalls:**
- **Coplanar Points:** The final point set may contain numerous coplanar points, especially for flat surface parts, slowing convex hull computation as all points may contribute to the convex envelope [S1, S2].
- **No Bounding Guarantees:** The octree approach does not provide any bounding guarantees that the resulting volume encloses the entire surface [S1, S2].
- **Infinite Recursion:** If the SDF value at the center remains less than the diagonal indefinitely (e.g., exactly on the surface where $f(\mathbf{c})=0$), the recursion may not terminate without a minimum cube size or maximum depth [G].

## Operating characteristics
- **Time Complexity:** Requires significantly more SDF evaluations than Sphere Carving to accurately obtain a good convex hull [S1, S2].
- **Memory Complexity:** Not given in the sources. [S1, S2]
- **Performance:** For shapes with significant anisotropy in $f$ values, the method may converge faster [S1, S2]. For very conservative functions, the computational advantage of Sphere Carving may be offset by the cost of intersecting many spheres [S1, S2].
- **Strengths:** May converge faster for shapes with significant anisotropy in $f$ values [S1, S2].
- **Limitations:** Does not provide bounding guarantees; requires more SDF evaluations; coplanar points can slow convex hull computation [S1, S2].
- **When to use:** When the shape has significant anisotropy in the SDF values and bounding guarantees are not critical [S1, S2].
- **When not to use:** When strict bounding guarantees are required or when the SDF is very conservative and Sphere Carving is more efficient [S1, S2].

## Background
- **Origin:** The method is described as an alternative strategy for computing convex bounding volumes for implicit surfaces defined by SDFs [S1, S2].
- **History:** The octree decomposition variant is attributed to Hart [Hart 1996] [S1, S2].
- **Usage:** Used in volume rendering and implicit surface processing as an alternative to Sphere Carving [S1, S2].

## Sources
[S1] Sphere Carving: Bounding Volumes for Signed Distance Fields, https://doi.org/10.1145/3730845, fetched 2026-10-27
[S2] Sphere Carving: Bounding Volumes for Signed Distance ..., https://research.adobe.com/publication/sphere-carving-bounding-volumes-for-signed-distance-fields/, fetched 2026-11-05
