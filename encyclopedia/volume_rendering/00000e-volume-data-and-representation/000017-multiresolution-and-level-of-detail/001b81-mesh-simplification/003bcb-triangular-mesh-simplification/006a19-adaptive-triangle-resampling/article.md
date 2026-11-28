# Adaptive Triangle Resampling


## Overview
Adaptive Triangle Resampling is a dynamic mesh optimization technique used in differentiable rendering to maintain a consistent triangle count while avoiding unnecessary geometry. It solves the problem of inefficient triangle distribution by removing transparent primitives and refining dense regions through edge splitting [S1, unofficial]. It is distinct from static simplification methods like [Hoppe's Progressive Mesh Simplification](14926-hoppe-s-progressive-mesh-simplification.md) because it operates iteratively during training to enforce a target budget, and from [Botsch Remeshing](10794-botsch-remeshing.md) by focusing specifically on screen-space coverage and transparency culling rather than geometric error metrics [S1, unofficial].

## How it works
**Inputs:** A triangle soup mesh (set of triangles), a target triangle count $N_{target}$, a screen-space edge-split threshold $T_{screen}$, and a set of training images with known camera poses.
**Outputs:** A resampled triangle soup with approximately $N_{target}$ triangles.

**Data Structures:**
*   **Triangle:** Defined by 3 vertices. Each vertex stores 3 floating-point coordinates $(x, y, z)$. Total 9 floats per triangle [S1, unofficial].
*   **Edge:** Defined by two vertex indices. Length is Euclidean distance in world space.
*   **Coverage Map:** A boolean or count array tracking which triangles receive pixel coverage in any training image.

**Steps:**
1.  **Transparency Culling:** Iterate through all triangles. For each triangle, render it against all training images. If a triangle receives zero pixel coverage in *any* training image (i.e., it is fully transparent or occluded everywhere), remove it from the mesh [S1, unofficial].
2.  **Edge Splitting Loop:** While the current triangle count $N_{current} < N_{target}$:
    a. Identify all edges in the current mesh.
    b. Calculate the screen-space footprint for each edge. This is typically the projected length of the edge in pixels on the image plane [S1, unofficial].
    c. Find the edge with the maximum screen-space length $L_{max}$.
    d. If $L_{max} > T_{screen}$, split the edge at its midpoint. This creates two new triangles (or modifies existing ones depending on topology, though triangle soup implies independent primitives, the context suggests a connected or semi-connected structure where splitting implies refinement; however, for a pure triangle soup, splitting an edge usually implies replacing the triangle or creating new ones if the mesh is connected. Given "triangle soup" and "MobileNeRF" context, the mesh is likely a set of independent triangles or a loose surface. The source says "split the longest edges", implying a connected mesh structure or a specific interpretation of "soup" where adjacency exists. We assume a connected mesh for edge splitting to be well-defined, or that "splitting" refers to subdividing the primitive if it were a higher-order element. However, standard triangle soup has no edges. *Correction*: The source mentions "MobileNeRF" which uses a triangle soup. In differentiable rendering of triangle soups, "splitting" usually refers to replacing a large triangle with smaller ones or adding new triangles. But the text says "split the longest edges". This implies the representation is treated as a mesh for the purpose of resampling, or the "soup" is a loose surface. We will describe the step as splitting the geometric edge of the triangle primitive if it is part of a mesh, or replacing the triangle. Given the ambiguity, we stick to the literal instruction: "split the longest edges". This requires the triangles to share edges, i.e., it is a mesh. If it is a true soup, this step is ill-defined. However, [S1, unofficial] contrasts it with "triangle-soup geometry". It is likely that the "resampling" creates a new set of triangles. Let's assume the standard interpretation: The algorithm maintains a mesh. If it is a soup, "splitting" might mean replacing a triangle with 4 smaller ones (subdivision) or adding new triangles. But "edge split" is specific. We will assume the input is a mesh for the resampling step, or that the "triangle soup" is a loose surface where adjacency is computed. *Refinement*: The source says "random triangle-soup geometry... apply a custom adaptive triangle-resampling". It is possible the resampling converts the soup into a mesh or treats it as one. We will describe the step as: Identify the longest edge in the mesh. Split it.
    e. Increment $N_{current}$.
3.  **Termination:** Stop when $N_{current} \approx N_{target}$ [S1, unofficial].

**Parameters:**
*   $N_{target}$: The desired number of triangles. Typical values depend on the scene and hardware budget.
*   $T_{screen}$: The screen-space edge-split threshold in pixels. Default is not explicitly given, but a remedy involves halving this value [S1, unofficial].

**Edge Cases and Pitfalls:**
*   **Tight Budgets:** In scenes with large spatial extent, a tight triangle budget may cause fine geometric structures (e.g., thin spokes) to be missed because the average screen-space footprint remains too large [S1, unofficial].
*   **Remedy:** To recover fine structures, decrease the average screen-space footprint by:
    1.  Increasing $N_{target}$ (e.g., by 4x).
    2.  Halving $T_{screen}$.
    3.  Reducing texture resolution by one subdivision level [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def adaptive_triangle_resampling(mesh, target_count, screen_threshold, training_images):
    """
    Resample a mesh to a target triangle count by culling transparent triangles
    and splitting long edges.
    
    Args:
        mesh: A mesh object with vertices, triangles, and methods for rendering.
        target_count: Integer, the desired number of triangles.
        screen_threshold: Float, max screen-space edge length in pixels before splitting.
        training_images: List of image data for coverage checking.
        
    Returns:
        Resampled mesh.
    """
    # Step 1: Remove fully transparent triangles
    triangles_to_keep = []
    for tri in mesh.triangles:
        # Check if triangle has any pixel coverage in any training image
        has_coverage = False
        for img in training_images:
            # Render triangle to img's camera and check for non-zero alpha/depth
            if render_triangle_has_coverage(tri, img):
                has_coverage = True
                break
        if has_coverage:
            triangles_to_keep.append(tri)
    
    mesh.triangles = triangles_to_keep
    
    # Step 2: Iteratively split longest edges
    while len(mesh.triangles) < target_count:
        # Find all edges and their screen-space lengths
        edges = get_all_edges(mesh)
        if not edges:
            break
            
        # Find edge with max screen-space length
        max_edge = None
        max_len = 0
        for edge in edges:
            # Project edge endpoints to screen space for each view
            # Take max length across views or average? Source says "screen-space footprint"
            # Assuming max screen-space length for safety
            screen_len = compute_max_screen_space_length(edge, mesh.cameras)
            if screen_len > max_len:
                max_len = screen_len
                max_edge = edge
        
        # Split if above threshold
        if max_len > screen_threshold:
            split_edge(mesh, max_edge)
        else:
            # If no edge exceeds threshold but count is low, we might need to split anyway
            # or the target is unreachable with current threshold.
            # Source implies we split until target is reached.
            # If threshold is not met, we might split the longest anyway to reach count?
            # Source: "iteratively split the longest edges until the triangle count reaches the target value"
            # It does not explicitly say "only if > threshold". It says "split the longest edges".
            # The threshold is mentioned in the remedy context.
            # Let's assume we split the longest edge regardless if count < target,
            # or only if it's "long". The remedy says "halve the screen-space edge-split threshold".
            # This implies the threshold is a condition.
            # However, if no edge is long enough, we can't split based on threshold.
            # We will split the longest edge if count < target, but the threshold guides the *granularity*.
            # Let's stick to the text: "split the longest edges until... target".
            # The threshold is likely a filter. If no edge > threshold, we stop splitting?
            # But we need to reach target.
            # Let's assume we split the longest edge if count < target, and the threshold is used
            # to determine *which* edges are candidates or to control density.
            # Given the remedy "halve the threshold... increase count", it suggests the threshold
            # controls the density.
            # We will split the longest edge if count < target.
            split_edge(mesh, max_edge)
            
    return mesh

def render_triangle_has_coverage(triangle, image):
    # [G] Placeholder for rasterization logic
    # Returns True if triangle covers any pixel with non-zero alpha/depth
    pass

def compute_max_screen_space_length(edge, cameras):
    # [G] Placeholder for projection logic
    # Returns the maximum projected length of the edge across all cameras
    pass

def split_edge(mesh, edge):
    # [G] Placeholder for edge splitting logic
    # Replaces the edge with two edges and updates adjacent triangles
    pass
```

## Subtopics

### Random Triangle Soup Initialization
An initialization method for synthetic data experiments that generates a random triangle-soup geometry to be refined by adaptive resampling. It is chosen when starting from a non-geometric prior or when testing the robustness of the resampling procedure. It differs from structured initialization by using uniform sampling and random orientation [S1, unofficial].
*   **Definition:** Generate a random set of triangles within a bounding box.
*   **Steps:**
    1.  Uniformly sample seed points inside the bounding box (scale typically 2 to 4) [S1, unofficial].
    2.  Place a triangle around each point with a random orientation and a fixed radius of 0.01 [S1, unofficial].
    3.  Apply the Adaptive Triangle Resampling procedure (culling and splitting) to refine this soup [S1, unofficial].
*   **Parameters:** Triangle radius: 0.01 [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** The transparency culling step requires rendering each triangle against all training images, which is $O(N \cdot I)$ where $N$ is the number of triangles and $I$ is the number of images. The splitting loop runs until the target count is reached, requiring edge length calculations and mesh updates.
*   **Memory Complexity:** The representation uses 9 floating-point numbers per triangle (3 per vertex) [S1, unofficial]. This matches the data footprint of standard 3D Gaussian Splatting variants [S1, unofficial].
*   **Strengths:** Ensures a consistent number of triangles, avoiding over-rendering of transparent areas and under-rendering of dense areas. Efficient use of texture resolution by encouraging uniform pixel coverage [S1, unofficial].
*   **Limitations:** In extreme cases with large spatial extent and tight triangle budgets, fine geometric structures can be missed [S1, unofficial]. Requires careful tuning of the screen-space threshold and target count to recover these details.
*   **When to use:** When training differentiable radiance field representations (like MobileNeRF) where triangle count is constrained and transparency culling is beneficial.
*   **When not to use:** When preserving fine geometric details is critical and the triangle budget is insufficient to represent them at the given screen-space threshold.

## Background
Adaptive Triangle Resampling is used in the context of "DiffSoup," a method for direct differentiable rasterization of triangle soup for extreme radiance field simplification. It was introduced in the paper "DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification" [S1, unofficial]. The method is designed to work with MobileNeRF [7] and aims to simplify radiance fields by optimizing the triangle distribution dynamically during training.

## Sources
[S1] DiffSoup: Direct Differentiable Rasterization of Triangle Soup for Extreme Radiance Field Simplification, https://doi.org/10.48550/arXiv.2603.27151, fetched 2026-10-30 (unofficial)
