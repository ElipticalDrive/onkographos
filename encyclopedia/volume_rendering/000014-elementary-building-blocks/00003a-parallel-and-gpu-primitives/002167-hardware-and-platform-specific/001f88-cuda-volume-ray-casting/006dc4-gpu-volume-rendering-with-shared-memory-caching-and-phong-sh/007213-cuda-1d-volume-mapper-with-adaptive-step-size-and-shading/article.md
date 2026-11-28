# CUDA 1D Volume Mapper with Adaptive Step Size and Shading

Also called: Adaptive Ray Casting with Gradient-Based Shading

## Overview
A GPU-based volume rendering method that casts rays through a 3D volume using an adaptive step size to skip empty space and compute photorealistic shading (Phong-Lambert) based on volume gradients [S1, unofficial]. It uses 1D transfer functions for opacity and color, and early ray termination for performance. It builds on [GPU Volume Rendering with Shared Memory Caching and Phong Shading](21421-gpu-volume-rendering-with-shared-memory-caching-and-phong-sh.md) by replacing the shared memory caching with a direct 1D texture lookup approach and introducing adaptive stepping logic that backtracks in opaque regions to improve gradient estimation [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** 3D volume texture (`CUDA_vtkCUDA1DVolumeMapper_input_texture`), 1D transfer function textures for alpha (`alpha_texture_1D`), gradient alpha (`galpha_texture_1D`), and color components (`colorR_texture_1D`, `colorG_texture_1D`, `colorB_texture_1D`).
- **Input Data Structures:**
  - `cuda1DTransferFunctionInformation` (`CUDA_vtkCUDA1DVolumeMapper_trfInfo`): Contains `intensityLow`, `intensityMultiplier`, `gradientLow`, `gradientMultiplier`.
  - `cudaVolumeInformation` (`volInfo`): Contains `Spacing` (float3), `SpacingReciprocal` (float3), `Ambient` (float), `Diffuse` (float), `Specular` (float2).
  - `cudaOutputImageInformation` (`outInfo`): Contains `rayStartX/Y/Z`, `rayIncX/Y/Z`, `numSteps`, `deviceOutputImage`, `resolution` (uint2).
  - `dRandomRayOffsets`: Array indexed by `threadIdx.x + BLOCK_DIM2D * threadIdx.y`.
- **Output:** `uchar4` pixel value in `outInfo.deviceOutputImage`.

**Data Structures:**
- Textures are bound with linear filter and clamp address mode. The 3D input texture uses unnormalized coordinates.
- `BLOCK_DIM2D` is a constant defining the block dimension (typically 16 or 32, exact value not specified in sources, but used in indexing).

**Numbered Steps:**

1.  **Initialization:**
    - Initialize output `outputVal` (float4): `x=0.0f`, `y=0.0f`, `z=0.0f`, `w=1.0f` (where `w` represents remaining opacity).
    - Load transfer function and volume info into registers: `functRangeLow` (`intensityLow`), `functRangeMulti` (`intensityMultiplier`), `gradRangeLow` (`gradientLow`), `gradRangeMulti` (`gradientMultiplier`), `space` (`SpacingReciprocal`), `incSpace` (`Spacing`), `ambient` (`Ambient`), `diffuse` (`Diffuse`), `spec` (`Specular`).

2.  **Ray Initialization:**
    - Apply randomized offset: `retDepth = dRandomRayOffsets[threadIdx.x + BLOCK_DIM2D * threadIdx.y]`.
    - Update `maxSteps = __float2int_rd(numSteps - retDepth)`.
    - Update `rayStart` by adding `retDepth * rayInc` to each component.
    - Calculate `rayLength = sqrtf(rayInc.x*rayInc.x*incSpace.x*incSpace.x + rayInc.y*rayInc.y*incSpace.y*incSpace.y + rayInc.z*rayInc.z*incSpace.z*incSpace.z)`.

3.  **Step Flags:**
    - Initialize `step.x = 0`, `step.y = 0`.

4.  **Ray Marching Loop (`while(maxSteps > 0)`):**
    - Calculate texture index: `tempIndex = functRangeMulti * (tex3D(CUDA_vtkCUDA1DVolumeMapper_input_texture, rayStart.x, rayStart.y, rayStart.z) - functRangeLow)`.
    - Fetch opacity: `alpha = tex1D(alpha_texture_1D, tempIndex)`.

    - **If `alpha > 0.0f`:**
        - Set `step.x = step.y`, `step.y = 0`.
        - Move ray: `rayStart += (step.x ? -rayInc : rayInc)`.
        - Update `maxSteps += (step.x ? 1 : -1)`.
        - If `!step.x`:
            - Compute gradient using central differences:
              ```
              gradient.x = (tex3D(..., rayStart.x+0.5f, ...) - tex3D(..., rayStart.x-0.5f, ...)) * space.x
              gradient.y = (tex3D(..., rayStart.y+0.5f, ...) - tex3D(..., rayStart.y-0.5f, ...)) * space.y
              gradient.z = (tex3D(..., rayStart.z+0.5f, ...) - tex3D(..., rayStart.z-0.5f, ...)) * space.z
              ```
            - Compute `gradMag = sqrtf(dot(gradient, gradient))`.
            - Update `alpha *= isfinite(gradRangeMulti) ? tex1D(galpha_texture_1D, gradRangeMulti*(gradMag-gradRangeLow)) : 1.0f`.
            - Compute `phongLambert = saturate(abs(gradient.x*rayInc.x*incSpace.x + gradient.y*rayInc.y*incSpace.y + gradient.z*rayInc.z*incSpace.z) / (gradMag * rayLength))`.
            - Compute `shadeD = ambient + diffuse * phongLambert`.
            - Compute `shadeS = spec.x * pow(phongLambert, spec.y)`.
            - Accumulate opacity: `multiplier = outputVal.w * alpha`, `outputVal.w *= (1.0f - alpha)`.
            - Accumulate color: `outputVal.x += multiplier * saturate(shadeD * tex1D(colorR_texture_1D, tempIndex) + shadeS)` (similarly for y, z).

    - **Check Early Termination:**
        - If `outputVal.w < 0.015625f`, set `outputVal.w = 0.0f` and `break`.

    - **Else (if `alpha <= 0.0f`):**
        - If `!step.x`:
            - Move ray forward: `rayStart += rayInc`, `maxSteps--`.
        - Set `step.y = !(step.x)`.
        - Move ray forward: `rayStart += rayInc`, `maxSteps--`.
        - Set `step.x = 0`.

5.  **Final Output Adjustment:**
    - `outputVal.w = 1.0f - outputVal.w`.
    - Saturate color components: `outputVal.x = saturate(outputVal.x)`, etc.

6.  **Kernel Launch and Composition:**
    - Calculate `index.x = blockDim.x * blockIdx.x + threadIdx.x`, `index.y = blockDim.y * blockIdx.y + threadIdx.y`.
    - Calculate `outindex = index.x + index.y * outInfo.resolution.x`.
    - Load `rayStart`, `rayInc`, `numSteps` from `outInfo` arrays.
    - Call `CUDA_vtkCUDA1DVolumeMapper_CUDAkernel_CastRays1D(rayStart, numSteps, rayInc, outputVal)`.
    - Convert `outputVal` to `uchar4` by multiplying by 255.0f.
    - Store in `outInfo.deviceOutputImage[outindex]`.

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Constants
BLOCK_DIM2D = 16  # Example value, not explicitly given in sources

# Global/Constant Data
trfInfo = CUDA_vtkCUDA1DVolumeMapper_trfInfo
volInfo = volInfo
outInfo = outInfo
dRandomRayOffsets = dRandomRayOffsets

# Textures (simulated access)
def tex3D_vol(x, y, z):
    return CUDA_vtkCUDA1DVolumeMapper_input_texture.sample(x, y, z)

def tex1D_alpha(idx):
    return alpha_texture_1D.sample(idx)

def tex1D_galpha(idx):
    return galpha_texture_1D.sample(idx)

def tex1D_colorR(idx):
    return colorR_texture_1D.sample(idx)

def tex1D_colorG(idx):
    return colorG_texture_1D.sample(idx)

def tex1D_colorB(idx):
    return colorB_texture_1D.sample(idx)

def saturate(val):
    return max(0.0, min(1.0, val))

def dot(v1, v2):
    return v1.x*v2.x + v1.y*v2.y + v1.z*v2.z

def sqrtf(val):
    return val ** 0.5

def powf(base, exp):
    return base ** exp

def isfinite(val):
    return val != float('inf') and val != float('-inf') and val == val # NaN check

def CastRays1D(rayStart, numSteps, rayInc, outputVal):
    # Initialize output
    outputVal.x = 0.0
    outputVal.y = 0.0
    outputVal.z = 0.0
    outputVal.w = 1.0

    # Load constants
    functRangeLow = trfInfo.intensityLow
    functRangeMulti = trfInfo.intensityMultiplier
    gradRangeLow = trfInfo.gradientLow
    gradRangeMulti = trfInfo.gradientMultiplier
    space = volInfo.SpacingReciprocal
    incSpace = volInfo.Spacing
    ambient = volInfo.Ambient
    diffuse = volInfo.Diffuse
    spec = volInfo.Specular

    # Randomized offset
    thread_idx = threadIdx.x + BLOCK_DIM2D * threadIdx.y
    retDepth = dRandomRayOffsets[thread_idx]
    maxSteps = int(numSteps - retDepth) # __float2int_rd
    rayStart.x += retDepth * rayInc.x
    rayStart.y += retDepth * rayInc.y
    rayStart.z += retDepth * rayInc.z

    # Ray length
    rayLength = sqrtf(
        rayInc.x**2 * incSpace.x**2 +
        rayInc.y**2 * incSpace.y**2 +
        rayInc.z**2 * incSpace.z**2
    )

    # Step flags
    step_x = 0
    step_y = 0

    while maxSteps > 0:
        # Fetch intensity
        intensity = tex3D_vol(rayStart.x, rayStart.y, rayStart.z)
        tempIndex = functRangeMulti * (intensity - functRangeLow)
        
        # Fetch alpha
        alpha = tex1D_alpha(tempIndex)

        if alpha > 0.0:
            step_x = step_y
            step_y = 0
            
            # Move ray
            if step_x:
                rayStart.x -= rayInc.x
                rayStart.y -= rayInc.y
                rayStart.z -= rayInc.z
                maxSteps += 1
            else:
                rayStart.x += rayInc.x
                rayStart.y += rayInc.y
                rayStart.z += rayInc.z
                maxSteps -= 1

            if not step_x:
                # Compute gradient
                grad_x = (tex3D_vol(rayStart.x + 0.5, rayStart.y, rayStart.z) - 
                          tex3D_vol(rayStart.x - 0.5, rayStart.y, rayStart.z)) * space.x
                grad_y = (tex3D_vol(rayStart.x, rayStart.y + 0.5, rayStart.z) - 
                          tex3D_vol(rayStart.x, rayStart.y - 0.5, rayStart.z)) * space.y
                grad_z = (tex3D_vol(rayStart.x, rayStart.y, rayStart.z + 0.5) - 
                          tex3D_vol(rayStart.x, rayStart.y, rayStart.z - 0.5)) * space.z
                
                gradient = (grad_x, grad_y, grad_z)
                gradMag = sqrtf(dot(gradient, gradient))
                
                # Update alpha with gradient info
                if isfinite(gradRangeMulti):
                    galpha_idx = gradRangeMulti * (gradMag - gradRangeLow)
                    alpha *= tex1D_galpha(galpha_idx)
                else:
                    alpha *= 1.0

                # Phong-Lambert shading
                numerator = abs(
                    grad_x * rayInc.x * incSpace.x +
                    grad_y * rayInc.y * incSpace.y +
                    grad_z * rayInc.z * incSpace.z
                )
                phongLambert = saturate(numerator / (gradMag * rayLength))
                
                shadeD = ambient + diffuse * phongLambert
                shadeS = spec.x * powf(phongLambert, spec.y)

                # Accumulate
                multiplier = outputVal.w * alpha
                outputVal.w *= (1.0 - alpha)
                
                colorR = tex1D_colorR(tempIndex)
                colorG = tex1D_colorG(tempIndex)
                colorB = tex1D_colorB(tempIndex)
                
                outputVal.x += multiplier * saturate(shadeD * colorR + shadeS)
                outputVal.y += multiplier * saturate(shadeD * colorG + shadeS)
                outputVal.z += multiplier * saturate(shadeD * colorB + shadeS)

            # Early termination
            if outputVal.w < 0.015625:
                outputVal.w = 0.0
                break

        else:
            # Alpha <= 0
            if not step_x:
                rayStart.x += rayInc.x
                rayStart.y += rayInc.y
                rayStart.z += rayInc.z
                maxSteps -= 1
            
            step_y = not step_x
            
            rayStart.x += rayInc.x
            rayStart.y += rayInc.y
            rayStart.z += rayInc.z
            maxSteps -= 1
            step_x = 0

    # Final adjustment
    outputVal.w = 1.0 - outputVal.w
    outputVal.x = saturate(outputVal.x)
    outputVal.y = saturate(outputVal.y)
    outputVal.z = saturate(outputVal.z)

def Composite():
    index_x = blockDim.x * blockIdx.x + threadIdx.x
    index_y = blockDim.y * blockIdx.y + threadIdx.y
    outindex = index_x + index_y * outInfo.resolution.x
    
    rayStart = (outInfo.rayStartX[outindex], outInfo.rayStartY[outindex], outInfo.rayStartZ[outindex])
    rayInc = (outInfo.rayIncX[outindex], outInfo.rayIncY[outindex], outInfo.rayIncZ[outindex])
    numSteps = outInfo.numSteps[outindex]
    
    outputVal = (0.0, 0.0, 0.0, 1.0)
    CastRays1D(rayStart, numSteps, rayInc, outputVal)
    
    # Convert to uchar4
    out = (
        int(outputVal.x * 255.0),
        int(outputVal.y * 255.0),
        int(outputVal.z * 255.0),
        int(outputVal.w * 255.0)
    )
    outInfo.deviceOutputImage[outindex] = out
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of steps per ray, but adaptive stepping reduces the average number of steps by skipping empty space. Early ray termination further reduces computation for opaque regions.
- **Memory Complexity:** Low per-thread memory usage (registers for constants and flags). Global memory usage includes the 3D volume texture and 1D transfer function textures.
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales with the number of pixels (threads) and the complexity of the volume (density of non-zero alpha regions).
- **Strengths:** Photorealistic shading via gradient-based Phong-Lambert model. Adaptive stepping improves performance in sparse volumes. Early ray termination handles opaque objects efficiently.
- **Limitations:** Gradient calculation at boundaries can be inaccurate due to clamp address mode. Backstepping logic adds complexity and potential for oscillation in highly opaque regions.
- **When to use:** Interactive volume rendering where shading quality is important and volumes have significant empty space.
- **When not to use:** Volumes with uniform density (adaptive stepping provides no benefit) or where boundary artifacts are critical.

## Background
Origin: Developed by John Stuart Haberl Baxter at Dr. Peter's Lab at Robarts Research Institute, first documented on May 11, 2012 [S1, unofficial]. It is part of the CUDAVolumeRendering library [S1, unofficial].

## Sources
[S1] finetjul/CUDAVolumeRendering, https://github.com/finetjul/CUDAVolumeRendering, fetched 2026-10-27 (unofficial)
