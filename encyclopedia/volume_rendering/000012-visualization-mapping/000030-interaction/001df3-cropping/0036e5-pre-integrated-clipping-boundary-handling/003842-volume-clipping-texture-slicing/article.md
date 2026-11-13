# Volume Clipping (Texture Slicing)

Also called: Clipping Node, Clipping in Texture Slicing, FSClipper, Volume Clipping Node, Volumetric Clipping

## Overview
Volume clipping is a technique to reveal internal structures of a volume model by setting unwanted parts to transparent, complementing transfer function specifications [S1, S4, unofficial]. It sets apart from sibling methods by allowing arbitrary clip geometry (polygons, voxelized geometry, or iso-surfaces) to define the visible region, often implemented via fragment shaders that modify alpha based on geometric distance [S3][S4, unofficial]. Unlike stencil-based approaches which decouple geometry from volume data, this method integrates clipping logic directly into the rendering pipeline of texture slicing [S1, S4, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** Volume texture (3D), SculptMap texture (3D, stores clipping state), PolygonCoord texture (2D, stores projection coordinates), Clip Plane (4D vector), Projection Matrix (4x4 matrix, implicit in `ProjPlane`).
*   **Output:** Rendered color/alpha buffer with clipped regions.

**Data Structures:**
*   `SculptMap`: A 3D texture where each voxel stores a mask value. Voxels inside the clip geometry are initialized to 0 (transparent), and remaining voxels are set to 1 (opaque) [S1, unofficial].
*   `PolygonCoord`: A 2D texture mapping fragment coordinates to 3D projection space [S3].
*   `ProjPlane`: A uniform `float4` representing the projection matrix components required to transform texture coordinates [S3].
*   `clipPlane`: A uniform `float4` defining the clipping plane equation [S3].

**Steps:**
1.  **Texture Sampling:** Sample the `SculptMap` at the current fragment's texture coordinates to retrieve the current alpha mask value [S3].
2.  **Coordinate Projection:** Sample the `PolygonCoord` texture to obtain 3D texture coordinates. Subtract 0.5 from the xyz components to center the coordinates, then transform them using the `ProjPlane` matrix [S3].
3.  **Perspective Division:** Perform a perspective divide on the transformed plane coordinates by their w-component to normalize them [S3].
4.  **Distance Calculation:** Calculate the 2D Euclidean distance $D$ between the projected point $P$ (derived from the tool or clipper definition) and the current fragment's plane coordinates, normalized by $P.z$ [S3].
5.  **Alpha Update:** If the calculated distance $D$ is less than the current alpha value of the `SculptMap` sample, update the alpha value to $D$ [S3]. This effectively erodes or defines the boundary based on proximity.
6.  **Final Compositing:** Combine the volume color with the updated alpha mask. If the alpha is 0, the voxel is transparent; otherwise, it is rendered [S3].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Input: v2f.Tex0 (float2), SculptMap (sampler3D), PolygonCoord (sampler2D), ProjPlane (float4), clipPlane (float4), P (float4: xyz=point, a=radius)
# Output: Col (float4)

def FSClipper(v2f, SculptMap, PolygonCoord, ProjPlane, clipPlane, P):
    # 1. Sample SculptMap
    Col = tex3D(SculptMap, v2f.Tex0)
    
    # 2. Sample PolygonCoord and Project
    texCoord = tex2D(PolygonCoord, v2f.Tex0)
    
    # Transform centered coordinates
    # Note: ProjPlane is assumed to contain the necessary matrix elements
    PlaneCoordinates = mul(float4(texCoord.xyz - 0.5f, 1.0f), ProjPlane)
    
    # 3. Perspective Divide
    PlaneCoordinates.xyz /= PlaneCoordinates.w
    
    # 4. Calculate 2D Euclidean Distance
    # P.xy is the projected point center, P.z is the depth factor
    D = length(PlaneCoordinates.xy - P.xy) / P.z
    
    # 5. Update Alpha if within distance threshold
    if D < Col.a:
        Col.a = D
        
    return Col
```

**Parameters:**
*   **Clipping Mode 1:** The part inside the clipping geometry is the clipped object (removed) [S1, unofficial].
*   **Clipping Mode 2:** The part outside the clipping geometry is the clipped object (removed) [S1, unofficial].
*   **Clipping Mode 3:** Initializes the clipped volume model and attaches it to the clipping geometry [S1, unofficial].
*   **Tool Radius:** Stored in $P.a$, defines the effective radius of the clipping tool [S3].

**Edge Cases and Pitfalls:**
*   **Multi-volume Clipping:** Issuing the same clipping operation twice does not realize multi-volume clipping directly. A clipping-geometry-list must be created to arrange different clipping geometries [S1, unofficial].
*   **Performance:** Multi-model clipping can seriously affect the interactive rate at runtime if not optimized [S1, unofficial].
*   **Occlusion:** There is unavailability of volume-based depth buffering. The final rendered frame with multiple volume models cannot always show correct occlusions like wireframe ones [S1, unofficial].

## Subtopics

### [Stencil-based Clipping Geometries](14493-stencil-based-clipping-geometries.md)
Uses the OpenGL stencil buffer to restrict rendering to pixels inside or outside an arbitrary closed surface geometry. Sets apart by decoupling clipping boundary definition from volume data, allowing any triangle mesh to act as a clipper without voxelization overhead. Choose when hardware stencil support is available and arbitrary mesh clippers are needed.

## Operating characteristics
*   **Time Complexity:** Depends on the number of slices and the cost of texture sampling and distance calculation per fragment. High-quality shading for clip boundaries adds computational cost [S4, unofficial].
*   **Memory Complexity:** Requires storage for the `SculptMap` texture, which scales with the volume resolution [S3].
*   **Performance:** Multi-step slicing with oversampling (e.g., four-times) yields the best quality-performance tradeoff due to increased cache coherency [S4, unofficial].
*   **Strengths:** Allows arbitrary clip geometry (polygons, voxelized, iso-surface); high-quality shading for clip boundaries [S4, unofficial].
*   **Limitations:** Multi-model clipping impacts interactive rate; lack of volume-based depth buffering leads to incorrect occlusions in multi-volume scenes [S1, unofficial].

## Background
Volume clipping was introduced as part of hybrid rendering frameworks for real-time manipulation of volume and surface models [S1, unofficial]. It complements transfer function specifications by providing geometric control over visibility [S1, S4, unofficial]. Techniques for volume clipping in 2D and 3D texture slicing methods were proposed by Weiskopf et al., who also presented high-quality shading techniques for clip boundaries [S4, unofficial]. The method is used in medical visualization (e.g., anatomical atlases) and engineering visualization (e.g., engine box clipping) [S1, S2, unofficial].

## Sources
[S1] A Hybrid Rendering Framework for the Real-time Manipulation of Volume and Surface Models, https://core.ac.uk/download/55863.pdf, fetched 2026-11-03 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S3] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
[S4] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
