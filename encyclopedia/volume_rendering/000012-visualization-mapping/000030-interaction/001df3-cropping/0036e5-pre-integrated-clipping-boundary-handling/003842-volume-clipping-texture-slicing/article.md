# Volume Clipping (Texture Slicing)

Also called: Clipping in Texture Slicing, FSClipper

## Overview
Volume Clipping (Texture Slicing) is a fragment shader-based technique for interactive volume sculpting that modifies the alpha channel of a volume representation based on geometric proximity to a clipping plane or tool. It solves the problem of defining arbitrary clip boundaries (polygons, voxelized geometry, or iso-surfaces) within 2D and 3D texture slicing pipelines by combining surface-based and volumetric shading at the clip boundary [S1][S2, unofficial]. Unlike parent methods that handle pre-integrated artifacts, this method actively sculpts the volume data during the rendering pass using a distance-based alpha modification [S1].

## How it works
**Inputs and Outputs:**
*   **Input:** A fragment shader receives vertex attributes via `vertex2fragment` struct containing `Tex0` (float2, semantic `TEXCOORD0`) [S1].
*   **Uniforms:**
    *   `SculptMap`: `sampler3D` at `TEXUNIT0` [S1]. Stores the current volume alpha state.
    *   `PolygonCoord`: `sampler2D` at `TEXUNIT1` [S1]. Stores coordinate mapping for the clip geometry.
    *   `ProjPlane`: `float4` at `C0` [S1]. Projection matrix component.
    *   `clipPlane`: `float4` at `C0` [S1]. Defines the clipping plane equation.
    *   `P`: `float4` [S1]. Represents the 3D point 'P' and tool radius 'r' (P.a = r).
*   **Output:** `Col` (float4, semantic `COLOR0`) [S1]. The final color and modified alpha.

**Data Structures:**
The `SculptMap` texture is a 3D texture where the alpha channel is modified by the clipper. The `PolygonCoord` texture is a 2D texture providing coordinate mapping [S1].

**Steps:**
1.  **Texture Mapping:** Sample the `SculptMap` texture using the input texture coordinates to retrieve the current volume state [S1].
    $$ Col = \text{tex3D}(SculptMap, v2f.Tex0) $$
2.  **Coordinate Retrieval:** Sample the `PolygonCoord` texture to obtain the texture coordinates for the clip geometry [S1].
    $$ texCoord = \text{tex2D}(PolygonCoord, v2f.Tex0) $$
3.  **Projection:** Transform the texture coordinates (offset by -0.5) into plane coordinates using the `ProjPlane` uniform [S1].
    $$ PlaneCoordinates = \text{mul}(\text{float4}(texCoord.xyz - 0.5f, 1.0f), ProjPlane) $$
4.  **Perspective Divide:** Perform perspective division on the plane coordinates [S1].
    $$ PlaneCoordinates.xyz /= PlaneCoordinates.w $$
5.  **Distance Calculation:** Calculate the 2D Euclidean distance $D$ between the projected point $P.xy$ and the current fragment's plane coordinates $PlaneCoordinates.xy$, normalized by $P.z$ [S1].
    $$ D = \frac{\text{length}(PlaneCoordinates.xy - P.xy)}{P.z} $$
    *Note: The source text also mentions a 3D Euclidean distance calculation involving $P.a$ (tool radius) in the context of the sculpting tool, but the specific FSClipper logic uses the 2D distance formula above [S1].*
6.  **Alpha Modification:** If the calculated distance $D$ is less than the current alpha value $Col.a$, update the alpha channel to $D$ [S1].
    $$ \text{if } (D < Col.a) \ Col.a = D $$

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
v2f_Tex0: float2  # From vertex shader
SculptMap: sampler3D
PolygonCoord: sampler2D
ProjPlane: float4
clipPlane: float4
P: float4         # P.xyz is point, P.a is radius

# Output
Col: float4

# Step 1: Sample SculptMap
Col = tex3D(SculptMap, v2f_Tex0)

# Step 2: Sample PolygonCoord
texCoord = tex2D(PolygonCoord, v2f_Tex0)

# Step 3: Project to plane
# Offset by 0.5 as per source
offset_coord = float4(texCoord.xyz - 0.5, 1.0)
PlaneCoordinates = mul(offset_coord, ProjPlane)

# Step 4: Perspective divide
PlaneCoordinates.xyz /= PlaneCoordinates.w

# Step 5: Calculate 2D Euclidean distance
# P.xy is the projected point center, P.z is used for normalization
D = length(PlaneCoordinates.xy - P.xy) / P.z

# Step 6: Update alpha if inside clip boundary
if D < Col.a:
    Col.a = D
```

**Parameters and Edge Cases:**
*   **Tool Radius:** The parameter $P.a$ represents the tool radius, used in the broader sculpting context (e.g., in `FSEraser` or `FSCombiner` logic described in the source), but the specific `FSClipper` distance formula relies on $P.z$ for normalization [S1].
*   **Precision:** The calculation involves perspective division and distance metrics, requiring careful handling of $P.z$ to avoid division by zero [S1].
*   **Boundary Quality:** The method employs a high-quality shading technique for clip boundaries, combining surface-based and volumetric shading to reduce artifacts [S2, unofficial].

## Operating characteristics
*   **Performance:** The method operates in the fragment shader, adding minimal overhead to the texture slicing pipeline. It benefits from cache coherency, with oversampling (e.g., 4x) providing a good quality-performance tradeoff [S2, unofficial].
*   **Flexibility:** Supports arbitrary clip geometry defined by polygons, voxelized geometry, or iso-surfaces of another volume [S2, unofficial].
*   **Limitations:** Relies on the accuracy of the `PolygonCoord` texture mapping and the `ProjPlane` transformation. The alpha modification is a direct distance-based cut, which may require additional blending or pre-integration steps to avoid hard edges if not handled by the parent boundary handling method [S1][S2, unofficial].

## Background
*   **Origin:** Introduced in the context of volume sculpting tools, specifically as the `FSClipper` fragment shader [S1].
*   **Development:** Extended techniques for volume clipping in 2D and 3D texture slicing methods proposed by Weiskopf et al. [S2, unofficial]. The approach integrates with hardware-accelerated volume rendering pipelines, such as those on ATI Radeon hardware [S2, unofficial].
*   **Usage:** Used for interactive volume sculpting, allowing artists or users to modify volume data (alpha channels) based on geometric tools [S1].

## Sources
[S1] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
[S2] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
