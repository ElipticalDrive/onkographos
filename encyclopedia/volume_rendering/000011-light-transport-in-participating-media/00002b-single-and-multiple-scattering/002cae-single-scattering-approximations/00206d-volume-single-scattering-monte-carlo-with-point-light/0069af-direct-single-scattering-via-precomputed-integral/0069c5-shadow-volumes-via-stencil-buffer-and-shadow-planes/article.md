# Shadow Volumes via Stencil Buffer and Shadow Planes

Also called: Hard shadows on view rays, Shadow planes rendering

## Overview
This method computes hard shadows on view rays for direct single scattering (DSS) in participating media by splitting the integration ray into lit and shadowed segments using shadow volumes generated from object silhouettes [S1, unofficial]. It differs from sibling methods by explicitly handling occlusion along the view ray through hardware-accelerated stencil buffer operations on infinite shadow planes, rather than relying solely on depth maps or ray marching for shadow determination [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** A scene with closed triangular meshes (2-manifold) for shadow casters, a light source position (potentially moving), and the precomputed 2D integral function $\Gamma$ from [Direct Single Scattering via Precomputed Integral](20414-direct-single-scattering-via-precomputed-integral.md).
*   **Outputs:** The direct single scattering radiance $L_{dss}$ for points on view rays, correctly accounting for partial occlusion.

**Data Structures:**
*   **Shadow Planes:** Infinite quads defined by a silhouette edge and the light position. Each plane is constituted by two edge vertices ($V_1, V_2$) and two points projected to infinity in the direction $(L - V)$, where $L$ is the light position [S1, unofficial].
*   **Silhouette Edges:** Edges common to one front-facing triangle and one back-facing triangle relative to the light position [S1, unofficial].

**Parameters:**
*   $\gamma_P, \gamma_B, \gamma_A, \gamma_O$: Angles corresponding to the endpoints of the lit/shadowed segments on the view ray, used as arguments to the $\Gamma$ function [S1, unofficial].
*   $k_t$: Extinction coefficient.
*   $t$: Distance along the view ray.
*   $h$: Height parameter in $\Gamma$.
*   $\Omega_{IS}$: Solid angle of the light source.

**Numbered Steps:**

1.  **(CPU) Compute Silhouettes:** For each moving shadow caster, identify silhouette edges. An edge is a silhouette edge if it is shared by one front-facing triangle and one back-facing triangle relative to the light position [S1, unofficial]. Link these edges into loop lists. This requires well-designed closed triangular meshes with connectivity information [S1, unofficial].
2.  **(GPU) Render Scene for Lit Areas:** Render the scene using conventional polygonal rendering to obtain direct illumination and indirect single scattering. Use shadow planes algorithms to determine surface shadows. The stencil buffer is initialized to contain lit areas of the scene [S1, unofficial].
3.  **(CPU) Generate Shadow Planes:** For each silhouette edge, generate a shadow plane. The plane is formed by the two edge vertices and two points projected to infinity in the direction from the vertex to the light [S1, unofficial].
4.  **(CPU) Sort Shadow Planes:** Sort the generated shadow planes in back-to-front order relative to the camera [S1, unofficial].
5.  **(GPU) Render Shadow Planes:** Render the sorted shadow planes.
    *   **Stencil Logic:** Use the stencil buffer to manage "shadow-in-shadow" artifacts (shadow planes that are themselves in shadow) [S1, unofficial].
    *   **Stencil Function:** Allow fragments if the stencil value is equal to 1 for front-facing planes and 0 for back-facing planes [S1, unofficial].
    *   **Stencil Operation:** Front-facing planes decrement the stencil buffer; back-facing planes increment it [S1, unofficial].
    *   **Blending:** Front-facing planes add their contribution to the radiance; back-facing planes subtract their contribution [S1, unofficial].
    *   **Depth Test:** Set to equality or accept only planes closer to the camera to ensure correct ordering [S1, unofficial].

**Formula:**
The direct single scattering radiance $L_{dss}$ for a view ray split into lit and shadowed parts is given by [S1, unofficial]:

$$
L_{dss}(d, \phi) = \frac{k_t^2 \Omega_{IS}}{4\pi} e^{-k_t t} \left[ \Gamma(k_t h, \gamma_P) - \Gamma(k_t h, \gamma_B) + \Gamma(k_t h, \gamma_A) - \Gamma(k_t h, \gamma_O) \right]
$$

Where $\gamma_P, \gamma_B, \gamma_A, \gamma_O$ are the angles corresponding to the endpoints of the segments on the view ray [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Step 1: CPU - Compute Silhouettes
def compute_silhouettes(mesh, light_pos):
    """
    Identifies silhouette edges for a 2-manifold mesh.
    Returns a list of edge loops.
    """
    silhouette_edges = []
    for edge in mesh.edges:
        tri1, tri2 = edge.neighbors
        # Check face orientation relative to light
        normal1 = tri1.normal
        normal2 = tri2.normal
        # Front-facing if dot product with light direction is positive
        # Back-facing if dot product is negative
        is_front1 = dot(normal1, light_pos - tri1.centroid) > 0
        is_front2 = dot(normal2, light_pos - tri2.centroid) > 0
        
        # Silhouette edge: one front, one back
        if is_front1 != is_front2:
            silhouette_edges.append(edge)
    return link_edges_into_loops(silhouette_edges)

# Step 4: CPU - Generate and Sort Shadow Planes
def generate_and_sort_shadow_planes(silhouette_loops, light_pos, camera_pos):
    """
    Generates infinite shadow planes from silhouette edges and sorts them back-to-front.
    """
    shadow_planes = []
    for loop in silhouette_loops:
        for edge in loop:
            v1, v2 = edge.vertices
            # Project vertices to infinity in direction (light - vertex)
            dir1 = normalize(light_pos - v1)
            dir2 = normalize(light_pos - v2)
            p1_inf = v1 + dir1 * INFINITY
            p2_inf = v2 + dir2 * INFINITY
            
            # Create plane from v1, v2, p1_inf, p2_inf
            plane = Plane(v1, v2, p1_inf, p2_inf)
            shadow_planes.append(plane)
    
    # Sort back-to-front relative to camera
    shadow_planes.sort(key=lambda p: distance(p.centroid, camera_pos), reverse=True)
    return shadow_planes

# Step 5: GPU - Render Shadow Planes
def render_shadow_planes(shadow_planes, stencil_buffer, radiance_buffer):
    """
    Renders shadow planes to update radiance and stencil buffer.
    """
    for plane in shadow_planes:
        # Determine if plane is front-facing or back-facing relative to camera
        is_front = is_front_facing(plane, camera_pos)
        
        # Set stencil function
        if is_front:
            stencil_func = EQUAL
            stencil_ref = 1
            stencil_op = DECREMENT
            blend_op = ADD
        else:
            stencil_func = EQUAL
            stencil_ref = 0
            stencil_op = INCREMENT
            blend_op = SUBTRACT
            
        # Set depth test to accept only closer planes
        depth_func = LESS_EQUAL
        
        # Render plane
        render(plane, stencil_func, stencil_ref, stencil_op, blend_op, depth_func)

# Formula Application
def compute_l_dss(k_t, Omega_IS, t, h, gamma_P, gamma_B, gamma_A, gamma_O, Gamma_func):
    """
    Computes L_dss using the precomputed integral function Gamma.
    """
    term1 = Gamma_func(k_t * h, gamma_P)
    term2 = Gamma_func(k_t * h, gamma_B)
    term3 = Gamma_func(k_t * h, gamma_A)
    term4 = Gamma_func(k_t * h, gamma_O)
    
    bracket = term1 - term2 + term3 - term4
    return (k_t**2 * Omega_IS / (4 * PI)) * exp(-k_t * t) * bracket
```

**Edge Cases and Pitfalls:**
*   **Shadow-in-Shadow Artifacts:** Shadow planes themselves can be in shadow, creating artifacts. This is handled by the stencil buffer logic in Step 5 [S1, unofficial].
*   **Mesh Requirements:** Requires well-designed closed triangular meshes (2-manifold) with connectivity information for silhouette computation [S1, unofficial].
*   **Moving Lights:** If the light source is moving, silhouettes and shadow planes must be recomputed every frame [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Dominated by silhouette computation on CPU ($O(E)$ where $E$ is edges) and sorting/rendering shadow planes on GPU ($O(N)$ where $N$ is number of silhouette edges).
*   **Memory Complexity:** Storage for shadow planes and silhouette loops.
*   **Strengths:** Hardware-accelerated via stencil buffer; handles complex occlusions along view rays; avoids ray marching for shadows.
*   **Limitations:** Requires 2-manifold meshes; artifacts if stencil logic is incorrect; performance degrades with many silhouette edges.
*   **When to use:** Real-time rendering of participating media with hard shadows from complex geometry.
*   **When not to use:** Soft shadows, non-manifold geometry, or static scenes with simple occluders where depth maps suffice.

## Background
Originated in the context of real-time single scattering effects in participating media [S1, unofficial]. The method builds on shadow plane algorithms for surface shadows [S1, unofficial]. It is used in graphics applications requiring realistic volumetric lighting with hard shadows.

## Sources
[S1] Real Time Single Scattering Effects, https://hal.science/hal-00622215v1, fetched 2026-10-26 (unofficial)
