# Volume Clipping (Texture Slicing)

Also called: Clipping Node, Clipping in Texture Slicing, FSClipper, Volume Clipping Node, Volumetric Clipping

## Overview
Volume clipping reveals internal structures of a volume model by setting unwanted parts to transparent, serving as a complement to transfer function specifications [S1, unofficial]. It allows arbitrary clip geometry defined by polygons, voxelized geometry, or the iso-surface of another volume [S4, unofficial]. Unlike [Pre-integrated Clipping Boundary Handling](7867-pre-integrated-clipping-boundary-handling.md), which focuses on artifact reduction at clip boundaries for pre-integrated rendering, this method defines the clipping logic itself via texture slicing or voxel masking [S1, S4, unofficial].

## How it works
**Inputs and Outputs**
*   **Input Volume:** 3D texture containing scalar values and associated colors.
*   **Clip Geometry:** Defined by polygons, voxelized geometry, or an iso-surface [S4, unofficial].
*   **SculptMap:** A 3D texture storing the clipping state/mask [S3].
*   **PolygonCoord:** A 2D texture storing projection coordinates [S3].
*   **Output:** Rendered volume with clipped regions rendered as transparent (alpha = 0) [S1, unofficial].

**Data Structures**
*   **Vertex Shader Input Struct:**
    *   `Tex0`: `float2`, semantic `TEXCOORD0` [S3].
*   **Fragment Shader Uniforms:**
    *   `SculptMap`: `sampler3D`, unit `TEXUNIT0` [S3].
    *   `PolygonCoord`: `sampler2D`, unit `TEXUNIT1` [S3].
    *   `ProjPlane`: `float4` (matrix row or vector, used in multiplication) [S3].
    *   `clipPlane`: `float4` [S3].
    *   `P`: `float4` (3D point coordinates + tool radius/depth parameter) [S3].

**Algorithm Steps**
The method can be implemented via two primary approaches described in the sources: a voxel-based initialization approach [S1, unofficial] and a fragment shader-based projection approach [S3].

### Approach 1: Voxel Mask Initialization [S1, unofficial]
1.  **Initialize Mask:** Create a 3D mask volume. Voxels inside the clipping geometry are initialized to 0. Remaining voxels are set to 1 [S1, unofficial].
2.  **Apply Mask:** During rendering, multiply the volume's alpha channel by the mask value. Voxels with mask value 0 become transparent (alpha = 0), effectively removing the "clipping object" [S1, unofficial].
3.  **Multi-Volume Handling:** To clip multiple volumes or apply multiple geometries, create a `clipping-geometry-list` to arrange different clipping geometries. Issuing the same clipping operation twice does not realize multi-volume clipping directly [S1, unofficial].

### Approach 2: FSClipper Fragment Shader [S3]
This approach implements the clipper tool for volume sculpting, modifying the alpha channel of a `SculptMap` texture based on the 2D Euclidean distance from a projected point.

1.  **Sample Textures:**
    *   Sample `SculptMap` at input texture coordinates:
        $$Col = \text{tex3D}(\text{SculptMap}, v2f.Tex0)$$
    *   Sample `PolygonCoord` at input texture coordinates to get `texCoord`:
        $$texCoord = \text{tex2D}(\text{PolygonCoord}, v2f.Tex0)$$
2.  **Project to Plane:**
    *   Offset texture coordinates by -0.5 and project using `ProjPlane`:
        $$PlaneCoordinates = \text{mul}(\text{float4}(texCoord.xyz - 0.5f, 1.0f), ProjPlane)$$
    *   Perform perspective divide:
        $$PlaneCoordinates.xyz /= PlaneCoordinates.w$$
3.  **Calculate Distance:**
    *   Calculate the 2D Euclidean distance $D$ between the projected point $P.xy$ and the current fragment's plane coordinates $PlaneCoordinates.xy$, divided by $P.z$:
        $$D = \frac{\text{length}(PlaneCoordinates.xy - P.xy)}{P.z}$$
4.  **Update Alpha:**
    *   If $D$ is less than the current alpha value $Col.a$, update $Col.a$ to $D$:
        $$\text{if } (D < Col.a) \ Col.a = D$$

**Pseudocode (FSClipper Implementation)**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs
v2f_Tex0: float2  # From vertex shader
SculptMap: sampler3D
PolygonCoord: sampler2D
ProjPlane: float4x4 # Or float4 representing matrix row/transform
clipPlane: float4
P: float4 # P.xyz = 3D Point, P.a = tool radius/depth parameter

# Output
Col: float4

# Step 1: Sample Textures
texCoord = sample2D(PolygonCoord, v2f_Tex0)
Col = sample3D(SculptMap, texCoord.xyz)

# Step 2: Project Texture Coordinates to Plane
# Offset by 0.5 as per source specification
offset_coord = texCoord.xyz - 0.5f
# Multiply by projection matrix/transform
PlaneCoordinates = mul(float4(offset_coord, 1.0f), ProjPlane)

# Step 3: Perspective Divide
PlaneCoordinates.xyz = PlaneCoordinates.xyz / PlaneCoordinates.w

# Step 4: Calculate 2D Euclidean Distance
# P.xy is the reference point in plane coordinates
# P.z is the divisor (depth/radius parameter)
distance_xy = length(PlaneCoordinates.xy - P.xy)
D = distance_xy / P.z

# Step 5: Update Alpha based on Distance
if D < Col.a:
    Col.a = D
```

**Parameters**
*   **Clipping Mode 1:** The part inside the clipping geometry is the clipped object (removed) [S1, unofficial].
*   **Clipping Mode 2:** The part outside the clipping geometry is the clipped object (removed) [S1, unofficial].
*   **Clipping Mode 3:** Initializes the clipped volume model and attaches it to the clipping geometry [S1, unofficial].
*   **Tool Radius/Depth ($P.a$):** Used in distance calculations for sculpting tools [S3].

**Edge Cases and Pitfalls**
*   **Multi-Model Clipping Performance:** Multi-model clipping seriously affects the interactive rate at runtime if not used properly [S1, unofficial].
*   **Occlusion Artifacts:** There is unavailability of volume-based depth buffering. The final rendered frame with multiple volume models cannot always show correct occlusions like wireframe ones [S1, unofficial].
*   **Precision:** The distance calculation relies on $P.z$ as a divisor; if $P.z$ is near zero, precision issues may arise [S3, G].

## Operating characteristics
*   **Time Complexity:** Depends on the resolution of the volume and the complexity of the clip geometry. The fragment shader approach [S3] performs constant-time operations per fragment but requires sampling multiple textures.
*   **Memory Complexity:** Requires storage for the volume texture, the `SculptMap` (3D texture), and `PolygonCoord` (2D texture) [S3].
*   **Performance:** The method is designed for real-time manipulation [S1, unofficial]. However, multi-volume clipping can degrade interactive rates [S1, unofficial].
*   **Strengths:** Allows arbitrary clip geometry (polygons, voxelized, iso-surface) [S4, unofficial]. Complements transfer functions [S1, unofficial].
*   **Limitations:** Lack of volume-based depth buffering leads to incorrect occlusions in multi-model scenes [S1, unofficial].

## Background
Volume clipping was introduced as part of hybrid rendering frameworks for real-time manipulation of volume and surface models [S1, unofficial]. It is considered a complement to transfer function specifications [S1, unofficial]. The technique has been applied in various domains, including anatomical atlases and virtual resection techniques [S2, unofficial]. The FSClipper shader implementation is part of a volume sculpting system using geometric tools [S3].

## Sources
[S1] A Hybrid Rendering Framework for the Real-time Manipulation of Volume and Surface Models, https://core.ac.uk/download/55863.pdf, fetched 2026-11-03 (unofficial)
[S2] Enhanced perception in volume visualization, https://doi.org/10.5821/dissertation-2117-94929, fetched 2026-10-26 (unofficial)
[S3] Volume sculpting based on geometric tools, https://doi.org/10.1007/bf03194498, fetched 2026-10-26
[S4] Hardware-Accelerated Volume Rendering, https://vcg.seas.harvard.edu/publications/20040101-hardware-accelerated-volume-rendering/paper, fetched 2026-10-27 (unofficial)
