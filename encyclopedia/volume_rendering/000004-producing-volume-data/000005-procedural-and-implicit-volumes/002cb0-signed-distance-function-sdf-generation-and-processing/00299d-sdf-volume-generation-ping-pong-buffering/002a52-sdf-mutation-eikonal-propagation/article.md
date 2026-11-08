# SDF Mutation & Eikonal Propagation

Also called: SDFMutate, brush carving, distance field update

## Overview
SDF Mutation & Eikonal Propagation is a GPU-based method for interactively modifying Signed Distance Fields (SDFs) using volumetric brushes. It solves the problem of maintaining geometrically correct distance values after local modifications (addition or subtraction) by combining immediate ray-casting-based brush application with iterative Eikonal equation propagation. It differs from static SDF generation by supporting real-time user input and from simple voxel carving by ensuring the resulting field remains a valid distance function through distance propagation.

## How it works
**Inputs and Outputs:**
- Input: Current SDF texture (`_sdfTexIn`), camera ray (origin `_mouseOrigin`, direction `_mouseDir`), brush parameters (`_brushSize`, `_brushHeight`), and interaction mode (`_mouseAdd`, `_mouseSubtract`).
- Output: Updated SDF texture (`_sdfTexOut`).

**Data Structures:**
- `_sdfTexIn`, `_sdfTexOut`: `RWTexture3D<float>` representing the 3D volume.
- `_mouseOrigin`, `_mouseDir`: `float4` (x,y,z,w) where xyz is the vector and w is unused or padding.
- `_brushSize`, `_brushHeight`: `float` defining the cylindrical brush dimensions.
- `_doEikonel`: `bool` flag indicating if Eikonal propagation is required.
- `_eikonelPasses`: `int` counter for remaining propagation passes.

**Steps:**
1. **Ray Casting:** Cast a ray from the camera through the mouse position to determine the interaction point in world space.
2. **Flag Setting:** Set `_mouseAdd` or `_mouseSubtract` based on the current interaction mode.
3. **Dispatch:** Dispatch the `SDFMutate` compute kernel with thread groups of size `[8,8,8]`.
4. **Kernel Execution:**
   - For each voxel in the output texture, determine if it lies within the brush volume intersected by the ray.
   - If inside the brush:
     - If `_mouseSubtract` is true: Set the SDF value to the distance from the voxel center to the brush boundary (carving).
     - If `_mouseAdd` is true: Set the SDF value to the negative distance from the voxel center to the brush boundary (adding volume).
   - If outside the brush: Copy the value from `_sdfTexIn`.
5. **Eikonal Propagation (Optional):** If `_doEikonel` is true, run additional passes to correct distance values.
   - In each pass, for each voxel, compute the new SDF value using the Eikonal solver based on its neighbors.
   - Decrement `_eikonelPasses`. Repeat until `_eikonelPasses` reaches zero.

**Formulas:**
The Eikonal solver uses 1D, 2D, and 3D cases to compute the new distance value $e$ for a voxel with gradient magnitude $g$ and neighbor distances $u_x, u_y, u_z$.

1D case:
$$ e = \min(h, v) + g $$
where $h, v$ are the distances of the two neighbors along the axis.

2D case:
If $|h - v| \ge g$:
$$ e = \min(h, v) + g $$
Else:
$$ e = \frac{1}{2} \left( (h+v) + \sqrt{(h+v)^2 - 2(h^2 + v^2 - g^2)} \right) $$

3D case:
Let $xyz = u_x + u_y + u_z$ and $n=3$.
If $d < 0$, the 3D solution is invalid, fall back to 2D cases.
Else:
$$ e = \frac{1}{n} \left( xyz + \sqrt{xyz^2 - n(u_x^2 + u_y^2 + u_z^2 - g^2)} \right) $$

The final Eikonal value for a voxel is the minimum of the 3D solution (if valid) and the minimum of the three 2D solutions (xy, xz, yz).

**Parameters:**
- `_brushSize`: Radius of the cylindrical brush. Typical value: 0.5 units.
- `_brushHeight`: Height of the cylindrical brush. Typical value: 1.0 units.
- `_eikonelPasses`: Number of propagation passes. Typical value: 1-3.
- `MAX_RAYMARCH_STEPS`: Maximum steps for raymarching in the renderer. Passed to shaders.

**Edge Cases and Pitfalls:**
- **Precision:** Floating-point precision errors in the Eikonal solver can cause artifacts. The 3D case check $d < 0$ handles degenerate cases.
- **Boundaries:** Voxel indices must be clamped to the texture bounds during neighbor access.
- **Degenerate Input:** If the ray does not intersect the volume, no mutation occurs.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Compute Shader Kernel SDFMutate
[numthreads(8, 8, 8)]
void SDFMutate(uint3 id : SV_DispatchThreadID)
{
    uint3 kernel = id;
    float3 uv = kernelToUV(kernel);
    
    // Read input value
    float inputVal = sdfUV(uv);
    float newVal = inputVal;
    
    // Check if voxel is within brush volume
    // This requires transforming the voxel position to world space and checking against the ray/brush
    float3 worldPos = uvToWorld(uv);
    float distToRay = distancePointToRay(worldPos, _mouseOrigin, _mouseDir);
    
    if (distToRay < _brushSize)
    {
        // Calculate distance to brush boundary
        float distToBoundary = _brushSize - distToRay;
        
        if (_mouseSubtract)
        {
            // Carving: set positive distance to boundary
            newVal = distToBoundary;
        }
        else if (_mouseAdd)
        {
            // Adding: set negative distance to boundary
            newVal = -distToBoundary;
        }
    }
    
    // Write to output
    _sdfTexOut[id] = newVal;
}

# Eikonal Propagation Kernel (run in separate dispatch if _doEikonel is true)
[numthreads(8, 8, 8)]
void EikonalPropagate(uint3 id : SV_DispatchThreadID)
{
    uint3 kernel = id;
    float3 uv = kernelToUV(kernel);
    
    // Get neighbor distances
    float3 u = float3(
        sdfUV(kernelToUV(kernel + uint3(1, 0, 0))),
        sdfUV(kernelToUV(kernel + uint3(0, 1, 0))),
        sdfUV(kernelToUV(kernel + uint3(0, 0, 1)))
    );
    
    // Gradient magnitude g is typically 1.0 for unit grid spacing
    float g = 1.0;
    
    // Compute Eikonal solution
    float e = 0.0;
    bool valid3D = eikonal3d(u, e, g);
    
    if (valid3D)
    {
        _sdfTexOut[id] = e;
    }
    else
    {
        float e1 = eikonal2d(u.x, u.y, g);
        float e2 = eikonal2d(u.x, u.z, g);
        float e3 = eikonal2d(u.y, u.z, g);
        _sdfTexOut[id] = min(min(e1, e2), e3);
    }
}

# Helper Functions
float eikonal1d(float h, float v, float g)
{
    return min(h, v) + g;
}

float eikonal2d(float h, float v, float g)
{
    if (abs(h - v) >= g)
    {
        return eikonal1d(h, v, g);
    }
    float hv = h + v;
    float d = hv * hv - 2.0 * (h * h + v * v - g * g);
    return 0.5 * (hv + sqrt(d));
}

bool eikonal3d(float3 u, out float e, float g)
{
    float xyz = u.x + u.y + u.z;
    float n = 3.0;
    float d = (xyz * xyz) - n * ((u.x * u.x + u.y * u.y + u.z * u.z) - (g * g));
    if (d < 0.0)
    {
        return false;
    }
    e = (1.0 / n) * (xyz + sqrt(d));
    return true;
}

float eikonal(float3 u, float g)
{
    float e = 0.0;
    if (eikonal3d(u, e, g))
    {
        return e;
    }
    float e1 = eikonal2d(u.x, u.y, g);
    float e2 = eikonal2d(u.x, u.z, g);
    float e3 = eikonal2d(u.y, u.z, g);
    return min(min(e1, e2), e3);
}
```

## Subtopics

## Operating characteristics
- **Time Complexity:** $O(N)$ per voxel per dispatch, where $N$ is the number of voxels in the volume. Eikonal propagation adds $P$ passes, where $P$ is `_eikonelPasses`.
- **Memory Complexity:** $O(N)$ for the two SDF textures (ping-pong buffering).
- **Measured Performance:** Performance depends on the resolution of the SDF volume and the number of Eikonal passes. The use of compute shaders allows for parallel processing of voxels.
- **Strengths:** Real-time interactive modification, maintains valid distance field properties.
- **Limitations:** Limited to GPU execution, requires careful tuning of Eikonal passes for accuracy vs. performance.
- **When to use:** Interactive 3D modeling, real-time sculpting.
- **When not to use:** Offline rendering where pre-computed SDFs are sufficient, CPU-only environments.

## Background
Originated in the context of real-time 3D modeling in Unity, as demonstrated by the `samuelbigos/signed_distance_fields_in_unity` repository. It leverages GPU compute shaders for efficient voxel manipulation and distance field correction.

## Sources
[S1] samuelbigos/signed_distance_fields_in_unity, https://github.com/samuelbigos/signed_distance_fields_in_unity, fetched 2026-10-27 (unofficial)
