# Near/Far Distance Precomputation Pass

Also called: Voxel Brick Distance Pass, tNear-tFar Rendering

## Overview
The Near/Far Distance Precomputation Pass renders voxel bricks to a render target to compute the entry ($t_{Near}$) and exit ($t_{Far}$) distances of viewing rays intersecting the volume for each pixel [S1, unofficial]. This prepares the bounds for the subsequent raymarching pass, allowing the main rendering pass to skip empty space and focus computation only on occupied regions [S1, unofficial]. It differs from [Homogeneous Volume Intersection](3260-homogeneous-volume-intersection.md) by explicitly rasterizing the volume geometry to derive depth bounds rather than analytically solving for intersections with a homogeneous bounding box.

## How it works
**Inputs:**
- A flag volume texture (`m_pFlagVolumeSRV`) indicating which bricks are active (occupied) [S1, unofficial].
- Camera matrices (`cb_mView`, `cb_mViewProj`) and view position (`cb_f4ViewPos`) [S1, unofficial].
- Volume parameters: `VOXEL_NUM_X/Y/Z`, `VOXEL_BRICK_RATIO`, `VOXEL_SIZE`, `g_cf3VoxelReso`, `g_cf3HalfBrickSize` [S1, unofficial].
- A render target view (`m_pFarNearRTV`) for output [S1, unofficial].

**Outputs:**
- A render target where each pixel stores $t_{Near}$ in the RGB channels and $t_{Far}$ in the Alpha channel [S1, unofficial].

**Data Structures:**
- **Render Target (`m_pFarNearRTV`):** 32-bit floating-point format (RGBA32F).
  - RGB channels: Store $t_{Near}$ (entry distance).
  - Alpha channel: Store $t_{Far}$ (exit distance).
- **Vertex Buffer (`m_pPassVB`):** Contains point primitives. Stride is `sizeof(short)` [S1, unofficial]. The content is effectively a list of primitive indices or dummy data, as the geometry is generated in the Geometry Shader [S1, unofficial].
- **Flag Volume (`tex_FlagVolume`):** A 3D texture where active bricks are marked with `1` and inactive with `0` [S1, unofficial].

**Numbered Steps:**
1. **Clear Render Target:** Clear `m_pFarNearRTV` with `{50.0f, 50.0f, 50.0f, -10.0f}` [S1, unofficial]. This sets an initial background value; RGB=50, Alpha=-10 [S1, unofficial].
2. **Set Topology:** Set primitive topology to `D3D11_PRIMITIVE_TOPOLOGY_POINTLIST` [S1, unofficial].
3. **Bind Resources:**
   - Input layout: `m_pOutVL` [S1, unofficial].
   - Vertex Buffer: `m_pPassVB` with stride `sizeof(short)` [S1, unofficial].
   - Shaders: Vertex Shader `m_pPassVS`, Geometry Shader `m_pActiveCellAndSOGS`, Pixel Shader `m_pFarNearPS` [S1, unofficial].
   - Constant Buffer: `m_pCBperFrame` bound to GS [S1, unofficial].
   - Shader Resource: `m_pFlagVolumeSRV` bound to GS [S1, unofficial].
4. **Set States:**
   - Rasterizer State: `m_pBackFaceRS` (back-face culling enabled) [S1, unofficial].
   - Blend State: `m_pBlendState` (configured for min/max operations, see below) [S1, unofficial].
5. **Draw:** Issue a draw call for `VOXEL_NUM_X * VOXEL_NUM_Y * VOXEL_Z / VOXEL_BRICK_RATIO^3` primitives [S1, unofficial].
6. **Restore:** Restore previous blend state [S1, unofficial].

**Blend State Configuration (`m_pBlendState`):**
- `BlendEnable = true`
- **RGB Channels ($t_{Near}$):**
  - `SrcBlend = D3D11_BLEND_ONE`
  - `DestBlend = D3D11_BLEND_ONE`
  - `BlendOp = D3D11_BLEND_OP_MIN`
  - Result: $t_{Near} = \min(t_{Near}, \text{new\_value})$ [S1, unofficial].
- **Alpha Channel ($t_{Far}$):**
  - `SrcBlendAlpha = D3D11_BLEND_ONE`
  - `DestBlendAlpha = D3D11_BLEND_ONE`
  - `BlendOpAlpha = D3D11_BLEND_OP_MAX`
  - Result: $t_{Far} = \max(t_{Far}, \text{new\_value})$ [S1, unofficial].
- `RenderTargetWriteMask = D3D11_COLOR_WRITE_ENABLE_ALL` [S1, unofficial].

**Pseudocode (Geometry Shader Logic):**
The Geometry Shader iterates over active bricks, generates a cube for each, transforms it to clip space, and outputs it. The Pixel Shader calculates the depth.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the Geometry Shader (m_pActiveCellAndSOGS)
# Input: Point list with primitive ID
# Output: Line Strip (cube edges)

struct GS_INPUT {
    uint primID;
};

struct PS_INPUT {
    float4 projPos;
    float4 Pos; // World/View space position for depth calculation
};

cbuffer cbPerFrame {
    matrix cb_mViewProj;
    float2 g_cf3HalfBrickSize; // Half size of a brick in world units
    float4 g_cf3VoxelReso;     // Resolution of the flag volume
    uint g_cuFlagVolSliceStep;
    uint g_cuFlagVolStripStep;
};

Texture3D g_srvFlagVolume;

void main(inout LineStream<PS_INPUT> lineStream, GS_INPUT input) {
    uint primID = input.primID;
    
    // Decode brick coordinates from primitive ID
    uint u3Idx;
    u3Idx.z = primID / (g_cuFlagVolSliceStep * g_cuFlagVolStripStep);
    u3Idx.y = (primID % (g_cuFlagVolSliceStep * g_cuFlagVolStripStep)) / g_cuFlagVolStripStep;
    u3Idx.x = primID % g_cuFlagVolStripStep;

    // Check if the brick is active
    // Note: Source uses g_srvFlagVolume[u3Idx]. Assuming normalized or direct indexing.
    int active = g_srvFlagVolume[int3(u3Idx.x, u3Idx.y, u3Idx.z)];
    
    if (active != 1) return;

    // Adjust Y coordinate (source flips Y)
    u3Idx.y = g_cf3VoxelReso.y - 1 - u3Idx.y;

    // Calculate cell center in world space
    // VOXEL_SIZE * VOXEL_BRICK_RATIO is the brick size
    float3 brickSize = float3(1.0, 1.0, 1.0) * g_cf3VoxelReso.x * 0.5; // Approximation based on g_cf3HalfBrickSize usage
    // More accurately from source:
    // f4CellCenterOffset = (u3Idx - g_cf3VoxelReso * 0.5f / VOXEL_BRICK_RATIO + 0.5f) * VOXEL_SIZE * VOXEL_BRICK_RATIO
    float3 centerOffset = (float3(u3Idx) - g_cf3VoxelReso.xyz * 0.5f / 1.0f + 0.5f) * g_cf3HalfBrickSize.xyz * 2.0f; 
    // Note: Source uses g_cf3HalfBrickSize directly as scale factor for the -1..1 cube
    
    float4 f4HalfCellSize = float4(g_cf3HalfBrickSize, 1.0f);
    
    // Define cube vertices in local space (-1 to 1)
    float3 verts[8] = {
        float3(-1, -1, -1), float3(-1, 1, -1), float3(1, 1, -1), float3(1, -1, -1),
        float3(-1, -1, 1),  float3(-1, 1, 1),  float3(1, 1, 1),  float3(1, -1, 1)
    };
    
    // Edges of the cube (indices into verts)
    int edges[12][2] = {
        {0,1}, {1,2}, {2,3}, {3,0}, // Front face
        {4,5}, {5,6}, {6,7}, {7,4}, // Back face
        {0,4}, {1,5}, {2,6}, {3,7}  // Connecting edges
    };

    for (int i = 0; i < 12; i++) {
        PS_INPUT output;
        
        // Vertex 1
        float3 v1 = verts[edges[i][0]] * g_cf3HalfBrickSize.xyz + centerOffset;
        output.Pos = float4(v1, 1.0f);
        output.projPos = mul(output.Pos, cb_mViewProj);
        lineStream.Append(output);
        
        // Vertex 2
        float3 v2 = verts[edges[i][1]] * g_cf3HalfBrickSize.xyz + centerOffset;
        output.Pos = float4(v2, 1.0f);
        output.projPos = mul(output.Pos, cb_mViewProj);
        lineStream.Append(output);
        
        lineStream.RestartStrip();
    }
}

// Pixel Shader (m_pFarNearPS)
// Calculates tNear and tFar based on the depth of the projected cube faces
float4 main(PS_INPUT input) : SV_Target {
    // Convert clip space position to view space depth
    // z_clip is in [0, 1] or [-1, 1] depending on API. Assuming D3D11 [0,1].
    float z_clip = input.projPos.z / input.projPos.w;
    
    // Convert to view space Z (negative for forward)
    float z_view = -(z_clip * 2.0 - 1.0); // Assuming standard D3D11 projection mapping
    
    // The depth value passed to the pixel shader is the depth of the fragment.
    // Since we are rendering wireframes/cubes, we need to determine the min/max depth 
    // of the brick along the ray.
    
    // However, the source implies a simpler approach: 
    // The blend state does MIN for RGB and MAX for Alpha.
    // This suggests the Pixel Shader outputs the depth of the current face.
    // For a cube, we need the min depth of all faces for tNear and max depth for tFar.
    // But rasterization processes fragments. 
    // A common technique is to render the cube twice or use specific depth logic.
    // Given the blend state MIN(RGB) and MAX(Alpha), it implies:
    // tNear = min(all tNear candidates)
    // tFar = max(all tFar candidates)
    
    // The source code does not show the PS logic explicitly, but the blend state 
    // dictates the accumulation. The PS likely outputs the depth of the current fragment.
    // To get correct tNear/tFar for the *entire* brick, the brick must be rendered 
    // such that all its faces contribute to the min/max.
    
    // Since back-face culling is enabled, we only see front faces.
    // This pass likely relies on the fact that for a convex object like a cube,
    // the front faces define the entry, and if we rendered back faces too (or handled 
    // the exit differently), we would get the exit. 
    // However, with only front faces, we only get the entry depth (tNear).
    // The source clears Alpha to -10 and uses MAX. This suggests tFar might be 
    // derived from the back faces or a separate pass, OR the "cube" rendering 
    // includes both front and back faces in a way that the MAX accumulates the 
    // furthest visible point.
    
    // Given the ambiguity in the PS code not being provided, we rely on the 
    // blend state and clear value.
    // tNear (RGB) = MIN(current_tNear, fragment_depth)
    // tFar (Alpha) = MAX(current_tFar, fragment_depth)
    
    // Note: fragment_depth must be in the same coordinate system as the ray distances.
    // Usually, this is view-space Z distance.
    
    float depth = input.projPos.z / input.projPos.w;
    // Convert to linear view space depth
    float linearDepth = ...; // Implementation detail depends on projection matrix
    
    return float4(linearDepth, linearDepth, linearDepth, linearDepth);
}
```

**Edge Cases and Pitfalls:**
- **Back-Face Culling:** The use of `m_pBackFaceRS` means only front-facing triangles are rendered. This correctly identifies the entry point ($t_{Near}$) but may fail to identify the exit point ($t_{Far}$) if the exit is only visible from the back. The use of `MAX` on Alpha suggests that either back faces are also rendered (contradicting culling) or the geometry is constructed to ensure front faces contribute to both, or a separate pass handles $t_{Far}$ [S1, unofficial].
- **Precision:** Using `float` for depth calculations may introduce precision errors for large scenes. The clear value of 50.0 and -10.0 must be outside the expected range of valid depths to ensure correct min/max accumulation [S1, unofficial].
- **Empty Bricks:** Inactive bricks (flag != 1) are discarded, preventing unnecessary rendering and blending operations [S1, unofficial].

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of active bricks. Each active brick generates a constant number of primitives (cube edges) [S1, unofficial].
- **Memory Complexity:** $O(1)$ additional memory per pixel for the render target. The flag volume requires $O(V)$ memory, where $V$ is the number of voxels [S1, unofficial].
- **Performance:** Depends on the number of active bricks and the resolution of the render target. Rasterizing wireframes can be expensive if the number of active bricks is high.
- **Strengths:** Efficiently culls empty space. Simple to implement using standard rasterization hardware.
- **Limitations:** Requires a second pass for raymarching. Accuracy depends on the rasterization of the cube geometry. Back-face culling may complicate $t_{Far}$ calculation [S1, unofficial].

## Background
This method is part of the Dynamic Volume Raycasting project by pengliu916 [S1, unofficial]. It is used in real-time volume rendering applications where efficient culling of empty space is critical for performance. The technique leverages the GPU's rasterization pipeline to compute depth bounds, avoiding the need for complex CPU-side bounding volume hierarchy traversal or analytical intersection tests for every ray [S1, unofficial].

## Sources
[S1] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
