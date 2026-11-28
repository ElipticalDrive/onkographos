# Probabilistic Silhouette Search

Also called: Markosian et al. Silhouette Search

## Overview
Probabilistic Silhouette Search is a spatial coherence-based heuristic for detecting and tracing silhouette edges on polygonal meshes [S1]. It solves the performance bottleneck of brute-force silhouette detection by limiting the initial search space and leveraging adjacency information to trace outlines rapidly [S1]. It differs from brute-force methods by achieving significant speedups for detailed models, though it may miss silhouettes in regions disconnected from the initial search seed [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A polygonal mesh (vertices, edges, faces), camera position, and an initial seed edge known to be a silhouette.
*   **Output:** A set of silhouette edges forming continuous outlines.

**Data Structures:**
*   **Adjacency Graph:** The mesh must support $O(1)$ access to adjacent edges for any given edge.
*   **Visibility Buffer:** An enhanced version of Appel’s hidden line algorithm data structure is used to determine if an edge is visible from the current viewpoint [S1].

**Algorithm Steps:**
1.  **Initial Search:** Identify a single silhouette edge. The search examines a constant $O(\sqrt{n})$ number of edges, where $n$ is the number of edges in the mesh [S1].
2.  **Probability Weighting:** The likelihood that an edge is a silhouette is inversely proportional to its dihedral angle $\theta$ with the adjacent face [S1]. Edges with smaller dihedral angles (sharper features) are prioritized.
3.  **Tracing:** Using the initial seed, trace the silhouette outline by traversing adjacent edges. Spatial coherence ensures that if an edge is a silhouette, its neighbors are likely candidates [S1].
4.  **Visibility Check:** For each candidate edge in the trace, use the enhanced Appel’s hidden line algorithm to verify visibility [S1].
5.  **Inter-frame Optimization (Optional):** Retain a fraction of detected silhouette edges from the previous frame to serve as seeds for the current frame, accelerating detection [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def probabilistic_silhouette_search(mesh, camera, prev_frame_seeds=None):
    """
    Detects silhouette edges using probabilistic search and spatial coherence.
    """
    silhouette_edges = []
    
    # Step 1: Determine initial seeds
    if prev_frame_seeds and len(prev_frame_seeds) > 0:
        seeds = prev_frame_seeds
    else:
        # Brute force constant O(sqrt(n)) search for initial silhouette
        seeds = brute_force_initial_search(mesh, camera, limit=int(math.sqrt(len(mesh.edges))))
    
    if not seeds:
        return silhouette_edges

    # Step 2: Trace silhouettes from seeds
    for seed_edge in seeds:
        if is_edge_in_silhouette_list(seed_edge, silhouette_edges):
            continue
            
        current_edge = seed_edge
        trace = [current_edge]
        
        while current_edge is not None:
            # Get adjacent edges for tracing
            adjacent_edges = get_adjacent_edges(current_edge, mesh)
            
            # Find next silhouette edge in trace
            next_edge = None
            for adj in adjacent_edges:
                if is_candidate_silhouette(adj, mesh, camera):
                    if is_visible(adj, camera, mesh):
                        next_edge = adj
                        break
            
            if next_edge:
                trace.append(next_edge)
                silhouette_edges.append(next_edge)
                current_edge = next_edge
            else:
                current_edge = None
                
    return silhouette_edges

def is_candidate_silhouette(edge, mesh, camera):
    """
    Probability of silhouette is inversely proportional to dihedral angle.
    """
    face1, face2 = edge.faces
    dihedral_angle = compute_dihedral_angle(face1, face2)
    
    # Higher probability for smaller angles (sharper edges)
    # Implementation detail: [G] Use a threshold or weighted random selection
    # based on 1/dihedral_angle to select the next edge in the trace.
    return True # Simplified for structure; actual implementation uses probability weighting

def is_visible(edge, camera, mesh):
    """
    Uses enhanced Appel's hidden line algorithm for visibility.
    """
    # [G] Standard hidden line test: project edge to screen space,
    # check if any other geometry occludes it.
    return True # Placeholder for specific Appel's algorithm implementation
```

**Parameters:**
*   **Dihedral Angle Threshold:** Not given in the sources.
*   **Initial Search Limit:** $O(\sqrt{n})$ edges [S1].

**Edge Cases and Pitfalls:**
*   **Disconnected Silhouettes:** Silhouettes in regions unconnected to the initial search seed may not be detected [S1].
*   **Precision:** The method relies on accurate dihedral angle calculation and visibility tests [S1].

## Subtopics

### Probabilistic Hybrid Silhouette Rendering
A hybrid approach by Northrup and Markosian for rendering artistic silhouettes using probabilistic detection, image processing, and stylization [S1]. It performs detection and rendering using the probabilistic method of Markosian et al., then detects edges using an image, interpolates detected edges to form line segments, and stylizes the line segments using various methods [S1].

## Operating characteristics
*   **Time Complexity:** Initial search is $O(\sqrt{n})$, where $n$ is the number of edges. Tracing is linear with respect to the number of silhouette edges due to spatial coherence [S1].
*   **Memory Complexity:** Requires storage for adjacency information and visibility buffers [S1].
*   **Performance:** Offers significant performance improvement over brute force search for detailed models [S1].
*   **Strengths:** Fast for low-detail models and scalable for detailed models compared to brute force [S1].
*   **Limitations:** May miss silhouettes in regions unconnected to the initial search [S1].
*   **Inter-frame Acceleration:** Retaining silhouette edges from previous frames yields a detection rate of almost 80% for subsequent frames [S1].

## Background
Probabilistic Silhouette Search was introduced by Markosian et al. in 1997 [S1]. It was developed to address the challenges of accurate and fast detection and visibility determination of edges in detailed models, where brute force methods do not scale well [S1]. The method uses an enhanced version of Appel's hidden line algorithm from 1967 for visibility calculations [S1]. DeCarlo et al. later observed the effectiveness of retaining silhouette edges for accelerated inter-frame calculation [S1].

## Sources
[S1] State of the Art Non-Photorealistic Rendering (NPR) Techniques, https://doi.org/10.2312/localchapterevents/tpcg/tpcg06/089-098, fetched 2026-10-26
