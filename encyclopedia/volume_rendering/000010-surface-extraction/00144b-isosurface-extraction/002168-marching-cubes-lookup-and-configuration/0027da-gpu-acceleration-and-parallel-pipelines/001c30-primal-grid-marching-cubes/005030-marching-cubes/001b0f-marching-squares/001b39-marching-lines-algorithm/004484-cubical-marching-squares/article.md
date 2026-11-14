# Cubical Marching Squares

Also called: CMS, CMSA, Marching Squares (for Voxels)

## Overview
Cubical Marching Squares (CMS) is an adaptive surface extraction method for volume data that partitions space in a binary fashion using X-Y, Y-Z, and X-Z planes to extract surfaces while preserving sharp features and topology [S4]. It extends the 2D marching squares algorithm to 3D by unfolding a cube into six 2D faces, applying marching squares to generate isocurves on each face, and reconnecting these segments in 3D to form components and triangles [S2]. This approach eliminates inter-cell dependency and crack patching by sampling sharp features on faces, making it suitable for adaptive multiresolution representations [S2]. It is significantly more complicated to implement than Dual Contouring but offers advantages in sharp feature preservation and crack-free guarantees [S3, unofficial].

## How it works

### Inputs and Outputs
**Input:** Hermite data $H$, defined as a signed grid where each edge is tagged with exact intersection points (sample points) and their normals (sample normals) [S2]. This data is typically acquired at a very fine resolution, e.g., a uniform $n_k \times n_k \times n_k$ grid [S2].
**Output:** A polygonal triangle mesh approximating the isosurface [S2].

### Data Structures
*   **Base Grid $B$:** A coarse uniform grid of size $n_0 \times n_0 \times n_0$ [S2].
*   **Adaptive Signed Octree:** The result of recursive subdivision. Cells are either "leaf cells" (no subcells) or internal nodes. Faces are "leaf faces" (no subfaces) or internal nodes. A "transition face" is shared by two cells subdivided at different levels [S2].
*   **Segment:** A line segment on a face connecting two sample points, tagged with potential sharp feature positions [S2].
*   **Component:** A closed loop (circle) of segments within a leaf cell [S2].

### Parameters
*   $n_0$: Base grid resolution exponent. Typical value: 8 [S2, S4].
*   $k$: Maximal level of subdivision. The finest resolution is $n_k = 2^k \cdot n_0$ [S2].

### Algorithm Steps

#### Stage 1: Building the Adaptive Signed Octree
1.  Initialize the base grid $B$ of size $n_0 \times n_0 \times n_0$ [S2].
2.  For each cell $c$ in $B$, call `SUBDIVIDECELL(H, c)` [S2].
    *   **Subdivision Condition 1 (Edge Ambiguity):** Subdivide if an edge has more than one sample point [S2]. This prevents incorrect surface generation [S2].
    *   **Subdivision Condition 2 (Complexity Heuristic):** Subdivide if the maximal spanning angle of all pairs of sample normals inside the cell exceeds a predefined angle threshold [S2, S4]. This indicates the surface is not flat enough [S2].
    *   **Subdivision Process:** When subdividing, subdivide faces first. Each face is split into four subfaces. Record relationships between subcells and subfaces. The subdivision sequence cycles through X-Y, Y-Z, and X-Z planes [S2, S4]. Stop if the maximal level $k$ is exceeded [S2].

#### Stage 2: Segment Generation for Faces
For each leaf face $f$, call `GENERATESEGMENT(f)` [S2]. For non-leaf faces, segments are the union of segments of their subfaces [S2].

1.  **Lookup:** Check sign patterns of face vertices against the marching squares lookup table [S2].
    *   Case 0: No segments.
    *   Cases 1, 2: One segment.
    *   Case 3: Two segments (face ambiguity) [S2].
2.  **Face Ambiguity Resolution (Case 3):** If two segments exist, call `RESOLVEFACEAMBIGUITY(f)` [S2].
    *   Detect sharp features on the potential segments. A 2D sharp feature is the intersection point of tangent lines defined by sample points and normals [S2].
    *   Choose the assignment where sharp features do NOT overlap. Overlapping features imply self-intersection, which is invalid for volume data [S2].
3.  **Sharp Feature Detection:** For each segment, test if the angle between the two normals at its endpoints is large enough. If so, tag the segment and store the sharp feature position [S2]. This removes inter-cell dependency [S2].

#### Stage 3: Surface Extraction for Cells
For each leaf cell $c$, call `EXTRACTSURFACE(c)` [S2].

1.  **Collect Segments:** Collect all segments from the cell's six faces using `GETSEGMENT(fi)` for each face $f_i$ [S2].
2.  **Component Tracing:** Call `TRACECOMPONENTS(l)` to group segments into circles (components). Start from an edge and sequentially find the next edge sharing an endpoint [S2].
3.  **Internal Ambiguity Check:** Check if the set of components $O$ has internal ambiguity using `HasInternalAmbiguity(O)` [S2].
    *   Internal ambiguity occurs when it is unclear if two components are joined or separated [S2].
    *   If yes, call `RESOLVEINTERNALAMBIGUITY(O)`. This is resolved by checking if 3D sharp features of two components overlap (forming cone-like volumes) [S2].
    *   If no, proceed to triangulation [S2].
4.  **3D Sharp Feature Sampling:** For each component, sample sharp features using Singular Value Decomposition (SVD) to solve $[n_i]^T p = [n_i s_i]$ for $p$, where $s$ is sample point location, $n$ is sample normal, and $p$ is sharp feature location [S2].
5.  **Triangulation:**
    *   **With Sharp Feature:** If a component has a sharp feature $p$, create a triangle fan centered at $p$ using component vertices $v_1 \dots v_n$: triangles $pv_1v_2, pv_2v_3, \dots, pvnv_1$ [S2].
    *   **Without Sharp Feature:** Calculate the average point of all sample points on the component and use it as the center for the triangle fan [S2].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def CubicalMarchingSquares(HermiteData H):
    # Stage 1: Build Adaptive Signed Octree
    B = InitializeBaseGrid(n0=8)
    for c in B:
        SubdivideCell(H, c)

    # Stage 2: Generate Segments for Faces
    for f in LeafFaces:
        GenerateSegment(f)

    # Stage 3: Extract Surfaces for Cells
    for c in LeafCells:
        ExtractSurface(c)

def SubdivideCell(H, c):
    # Check Edge Ambiguity
    for edge in c.edges:
        if len(H.samples_on_edge(edge)) > 1:
            SubdivideCellRecursively(c)
            return

    # Check Complexity Heuristic
    max_angle = 0
    normals = H.sample_normals_inside(c)
    for n1 in normals:
        for n2 in normals:
            angle = AngleBetween(n1, n2)
            if angle > max_angle:
                max_angle = angle
    
    if max_angle > ANGLE_THRESHOLD:
        SubdivideCellRecursively(c)

def SubdivideCellRecursively(c):
    if c.level >= k:
        return
    
    # Subdivide faces first
    for face in c.faces:
        face.subdivide()
    
    # Create subcells
    c.create_subcells()
    
    # Recurse
    for subc in c.subcells:
        SubdivideCell(H, subc)

def GenerateSegment(Face f):
    sign_pattern = GetSignPattern(f.vertices)
    case = LookupMarchingSquareCase(sign_pattern)
    
    if case == 0:
        f.segments = []
    elif case == 1 or case == 2:
        l = CreateSegmentFromCase(case, f)
        f.segments = [l]
        DetectFaceSharpFeature(f.segments)
    elif case == 3:
        # Face Ambiguity
        seg1, seg2 = CreateTwoSegmentsFromCase3(f)
        sharp1 = DetectSharpFeature(seg1)
        sharp2 = DetectSharpFeature(seg2)
        
        if SharpFeaturesOverlap(sharp1, sharp2):
            # Choose the other assignment (not overlapping)
            # Note: In practice, this involves swapping connections
            f.segments = ResolveAmbiguity(f)
        else:
            f.segments = [seg1, seg2]
        
        DetectFaceSharpFeature(f.segments)

def ExtractSurface(Cell c):
    segments = []
    for fi in c.faces:
        segments.extend(GetSegment(fi))
    
    components = TraceComponents(segments)
    
    for o in components:
        DetectSharpFeature(o)
    
    if HasInternalAmbiguity(components):
        ResolveInternalAmbiguity(components)
    
    for o in components:
        Triangulate(o)

def DetectSharpFeature(segment_or_component):
    # For segments: check angle between endpoint normals
    # For components: Solve [ni]^T p = [ni si] via SVD
    pass

def Triangulate(component):
    vertices = component.vertices
    if component.has_sharp_feature:
        p = component.sharp_feature
    else:
        p = AveragePoint(component.sample_points)
    
    for i in range(len(vertices)):
        v1 = vertices[i]
        v2 = vertices[(i + 1) % len(vertices)]
        CreateTriangle(p, v1, v2)
```

### Edge Cases and Pitfalls
*   **Cracks:** Can occur on transition faces (faces shared by cells at different subdivision levels). The algorithm guarantees crack-free meshes because every segment on a transition face is shared by exactly two components from neighboring cells [S2, S4].
*   **Precision:** Hermite data must be acquired at high resolution to ensure accurate sample points and normals [S2].
*   **Ambiguity Resolution:** The sharp feature overlap test may produce different results than asymptotic deciders, but is effective for volume data [S2].

## Operating characteristics

*   **Time Complexity:** Generally slightly slower than standard marching cubes due to the overhead of adaptive subdivision and segment tracing [S2].
*   **Memory Complexity:** Depends on the adaptive octree depth. Higher detail requires more memory for the octree structure and segment lists.
*   **Accuracy:** Produces about one-third the distance error of other algorithms like marching cubes [S4].
*   **Strengths:**
    *   Adaptive refinement based on surface complexity and ambiguity.
    *   Preserves sharp features.
    *   Crack-free meshes without post-processing patching.
    *   Eliminates inter-cell dependency, allowing for potential parallelization [S4].
*   **Limitations:**
    *   Significantly more complicated to implement than Dual Contouring or standard Marching Cubes [S3, unofficial].
    *   Requires Hermite data (sample points and normals), which may need to be pre-computed or derived [S2].
*   **When to Use:** When sharp feature preservation and crack-free adaptive meshes are critical, and implementation complexity is acceptable.
*   **When Not to Use:** When simplicity of implementation is prioritized, or when Hermite data is not available and cannot be efficiently computed.

## Background

Cubical Marching Squares was introduced as an adaptive surface extraction method for volume data. It builds on the 2D marching squares algorithm, extending it to 3D by treating each cube as six 2D faces [S2]. The method was developed to address issues with inter-cell dependency and crack formation in adaptive multiresolution representations, which are common in algorithms like Edge Matching Cubes (EMC) and Dual Contouring (DC) [S2, S4]. It uses Hermite data, a format previously used in dual contouring, to store exact intersection points and normals [S2]. The algorithm has been applied in volume visualization and 3D sculpting tools, offering improved visual quality and interactive performance [S4].

## Sources

[S1] Presentation on the noa voxel game engine, made and presented in-engine (2015), http://andyhall.github.io/noa-lt/, fetched 2026-10-26 (unofficial)
[S2] Cubical Marching Squares: Adaptive Feature Preserving Surface Extraction from Volume Data, https://doi.org/10.1111/j.1467-8659.2005.00879.x, fetched 2026-10-26
[S3] Improve surface rendering, https://github.com/CauldronDevelopmentLLC/CAMotics/issues/1, fetched 2026-10-27 (unofficial)
[S4] Detail sculpting using cubical marching squares, https://doi.org/10.1145/1152399.1152402, fetched 2026-11-06
