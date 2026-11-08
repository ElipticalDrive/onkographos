# Dynamic Volume Raycasting with Accumulated Shading

Also called: Dynamic Volume Raycasting (Accumulated Shading), MarchPS_OUT RaymarchPS, Méthode de Scharsach, RayCast.fx accumulatedShading, Scharsach Ray-Casting, accumulatedShading

## Overview
Dynamic Volume Raycasting with Accumulated Shading is a GPU-based volume rendering technique that performs front-to-back alpha blending and shading accumulation in a single fragment shader pass using dynamic branching [S1, S2, unofficial]. It extends Krüger’s method by leveraging Shader Model 3.0 features to iterate along rays within the fragment shader, enabling early ray termination based on accumulated opacity [S1, unofficial]. Unlike static volume rendering, this method supports dynamic volumes composed of overlapping spherical sources, updating the density field via a compute shader before rendering [S2, unofficial]. It is distinguished from its parent method by its specific implementation of single-pass accumulation with dynamic branching and its support for procedurally generated dynamic density fields [S1, S2, unofficial].

## How it works
The method consists of two main phases: a compute shader phase to update the dynamic volume data, and a rendering phase that executes in four passes to allow navigation and compositing with surface geometry [S1, S2, unofficial]. The core rendering logic is contained in the `accumulatedShading` function.

### Inputs and Outputs
**Inputs:**
*   `eyeray`: A ray structure containing origin `o` (float4) and direction `d` (float4).
*   `f2NearFar`: A float2 containing the near (`x`) and far (`y`) t-values of the ray intersection with the volume bounding box in world space [S2, unofficial].
*   `f2MinMaxDen`: A float2 containing the minimum and maximum density values to consider for rendering [S2, unofficial].
*   `cb_f2MinMaxDensity`: A float2 containing the transfer function’s density range [S2, unofficial].
*   `VOXEL_SIZE`: A constant defining the physical size of a voxel [S2, unofficial].
*   `g_srvDataVolume`: A 3D texture (`RWTexture3D<float4>` or `ShaderResourceView`) storing the volume data [S2, unofficial].

**Outputs:**
*   `outColor`: The accumulated color (float4) [S2, unofficial].
*   `outDepth`: The depth of the first surface entry (float) [S2, unofficial].

### Data Structures
*   **Volume Data Texture (`tex_DataVolume`)**: `RWTexture3D<float4>` [S2, unofficial].
    *   `x`: Accumulated density.
    *   `yzw`: Normalized color vector [S2, unofficial].
*   **Ball Info Buffer**: A constant buffer (`cbuffer`) containing `float4 balls[MAX_BALLS]` [S2, unofficial].
    *   `xyz`: Center of the sphere.
    *   `w`: Radius (used as squared radius in the `Ball` function logic, see Pitfalls) [S2, unofficial].
*   **Ball Color Buffer**: An array `ballsCol[MAX_BALLS]` of `float3` [S2, unofficial].

### Compute Shader Update (Dynamic Volume Generation)
Before raycasting, the volume density field is updated based on a list of dynamic spherical sources [S2, unofficial].

1.  Initialize `field` to `float4(0, 1, 1, 1)` for each voxel [S2, unofficial].
2.  Iterate through all balls `i` from `0` to `numOfBalls - 1`:
    *   Calculate density contribution using the `Ball` function: `density = Ball(currentPos, balls[i].xyz, balls[i].w)` [S2, unofficial].
    *   The `Ball` function calculates distance squared `DistSq` between `currentPos` and `balls[i].xyz`. It returns `balls[i].w / DistSq` [S2, unofficial]. Note: `balls[i].w` is treated as a squared radius parameter in the division, despite being named `w` [S2, unofficial].
    *   Add `density` to `field.x` [S2, unofficial].
    *   Add color contribution: `ballsCol[i].xyz * pow(density, 3) * 1000` to `field.yzw` [S2, unofficial].
3.  Normalize `field.yzw` [S2, unofficial].
4.  Store `field` in `tex_DataVolume` at the current thread ID [S2, unofficial].

### Ray Initialization
1.  Set ray origin `eyeray.o` to `cb_f4ViewPos` [S2, unofficial].
2.  Calculate ray direction `eyeray.d` from the camera origin to the pixel position `input.Pos` [S2, unofficial].
3.  Normalize `eyeray.d.xyz` [S2, unofficial].
4.  **Pitfall Handling**: If any component of the normalized direction is exactly `0.0`, replace it with `1e-15` to prevent division by zero in subsequent intersection calculations [S2, unofficial].
5.  Load `f2NearFar` from the `g_srvFarNear` resource at index `input.Tex` [S2, unofficial].
6.  Multiply `f2NearFar` by the ray length `len` to convert from texture space to world space t-values [S2, unofficial].

### Ray Marching Loop (Fragment Shader)
The following pseudocode implements the core accumulation loop [S2, unofficial].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def accumulatedShading(eyeray, f2NearFar, f2MinMaxDen):
    bFirstEnter = True
    # Calculate near and far points in world space
    Pnear = eyeray.o.xyz + eyeray.d.xyz * f2NearFar.x
    Pfar = eyeray.o.xyz + eyeray.d.xyz * f2NearFar.y
    
    P = Pnear
    t = f2NearFar.x
    
    # Step size: 0.8 * VOXEL_SIZE ensures sub-voxel stepping
    tSmallStep = 0.8 * VOXEL_SIZE
    PsmallStep = eyeray.d.xyz * tSmallStep
    
    # Initial sample
    field_now = SampleLevel(g_srvDataVolume, local2tex(P), 0).x
    
    src = float4(0, 0, 0, 0)
    dst = float4(0, 0, 0, 0)
    outDepth = 0.0
    
    while t <= f2NearFar.y:
        texCoord = local2tex(P)
        Field = SampleLevel(g_srvDataVolume, texCoord, 0)
        
        # Check if density is within valid range
        if Field.x >= f2MinMaxDen.x and Field.x <= f2MinMaxDen.y:
            if bFirstEnter:
                # Record depth of first entry
                # Note: The source code divides by 10.f, likely a specific scaling factor
                outDepth = mul(float4(P, 1.0), cb_mView).z / 10.0
                bFirstEnter = False
            
            # Compute transfer function
            opacity = transferFunction(Field.x)
            
            # Prepare source color
            # Scale opacity by 0.25f as per source implementation
            src.a = opacity * 0.25f
            src.rgb = Field.yzw * src.a
            
            # Front-to-back blending
            dst = (1.0f - dst.a) * src + dst
            
            # Early termination
            if dst.a >= 0.95f:
                break
        
        # Advance ray
        P += PsmallStep
        t += tSmallStep
        
    outColor = dst * dst.a
    return outColor, outDepth

def local2tex(P):
    # Convert world position P to texture coordinates
    uv = P * inverseXYZsize.xyz + 0.5
    uv.y = 1 - uv.y
    return uv

def transferFunction(density):
    # Normalize density to [0, 1] range based on global min/max
    opacity = (density - cb_f2MinMaxDensity.x) / (cb_f2MinMaxDensity.y - cb_f2MinMaxDensity.x)
    
    # Polynomial transfer function
    p2 = opacity * opacity + 0.02
    p4 = p2 * p2
    
    return p4 * 0.3 + p2 * 0.1 + opacity * 0.15
```

### Parameters
*   `tSmallStep`: `0.8 * VOXEL_SIZE` [S2, unofficial].
*   Early Termination Threshold: `0.95f` for accumulated opacity `dst.a` [S2, unofficial].
*   Transfer Function Scaling: Output is multiplied by `0.25f` before blending [S2, unofficial].

### Edge Cases and Pitfalls
*   **Zero Division**: Ray direction components set to `0.0` are replaced with `1e-15` [S2, unofficial].
*   **Radius Interpretation**: The `Ball` function uses `balls[i].w` in a division `RadiusSq * InvDistSq`. The source code passes `balls[i].w` as the radius parameter but the function logic implies it acts as a squared radius or scaling factor for the falloff [S2, unofficial].
*   **Depth Scaling**: The depth output is divided by `10.f`, which is a specific implementation detail that may require adjustment for different scene scales [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Linear with respect to the number of steps along the ray, which depends on the volume thickness and step size. Early termination reduces average steps [S2, unofficial].
*   **Memory Complexity**: O(N) for the volume texture, where N is the number of voxels [S2, unofficial].
*   **Performance**: Performance is significantly impacted by the volume data size due to the rasterization of support primitives required for the 4-pass structure [S1, unofficial].
*   **Scalability**: The method does not scale well with large volume data sizes [S1, unofficial].
*   **Strengths**: Supports dynamic volumes, single-pass accumulation, and correct compositing with surface geometry [S1, S2, unofficial].
*   **Limitations**: Poor scalability with data volume, reliance on Shader Model 3.0 dynamic branching [S1, unofficial].

## Background
The method was introduced by Scharsach in 2005 as an extension of Krüger’s raycasting method [S1, unofficial]. It leverages Shader Model 3.0 features, specifically dynamic branching, to perform accumulation in a single pass within the fragment shader [S1, unofficial]. The technique is designed for interactive exploration of procedural volumes and allows navigation within the volume data [S1, unofficial].

## Sources
[S1] Représentation et algorithmes pour l'exploration interactive de volumes procéduraux étendus et détaillés, https://inria.hal.science/inria-00598406v1, fetched 2026-10-26 (unofficial)
[S2] pengliu916/DynamicVolumeRayCasting, https://github.com/pengliu916/DynamicVolumeRayCasting, fetched 2026-10-26 (unofficial)
