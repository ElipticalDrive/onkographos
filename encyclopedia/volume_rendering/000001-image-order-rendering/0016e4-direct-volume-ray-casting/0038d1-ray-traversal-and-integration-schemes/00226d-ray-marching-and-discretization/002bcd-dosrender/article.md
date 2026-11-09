# DOSRender

Also called: DOS Renderer, Depth of Field Scattering, Depth-Ordered Slice Renderer, Direct Volume Rendering with Occlusion and Instance Tracking, Occlusion-Aware Slice Accumulator [S1, unofficial]

## Overview
DOSRender is a depth-ordered, slice-based volume rendering technique that accumulates color and occlusion along a ray while simultaneously tracking semantic instance and group IDs for segmentation [S1, unofficial]. It distinguishes itself from standard front-to-back ray casting by integrating precomputed occlusion samples and hierarchical group membership buffers directly into the accumulation pass, allowing for efficient occlusion-aware rendering and semantic data extraction in a single pass [S1, unofficial].

## How it works
**Inputs and Outputs**
*   **Inputs:**
    *   `uMvpInverseMatrix`: Inverse Model-View-Projection matrix for unprojecting screen coordinates [S1, unofficial].
    *   `uDepth`: Current depth value for the ray [S1, unofficial].
    *   `uInstanceID`, `uGroupID`: Textures storing instance and group IDs at 2D screen positions [S1, unofficial].
    *   `uIDVolume`: 3D texture mapping 3D positions to instance IDs [S1, unofficial].
    *   `sGroupMembership`: `std430` buffer mapping instance IDs to group IDs [S1, unofficial].
    *   `uAccumulator`: Texture storing the accumulated color, occlusion, instance ID, and group ID from previous slices [S1, unofficial].
    *   `uOcclusion`: Texture containing precomputed occlusion samples [S1, unofficial].
    *   `uOcclusionScale`: Scale factor for occlusion sample offsets [S1, unofficial].
    *   `uOcclusionSamplesCount`: Number of occlusion samples to average [S1, unofficial].
    *   `uExtinction`: Uniform float controlling the extinction coefficient multiplier [S1, unofficial].
    *   `uSliceDistance`: Uniform float representing the distance between slices [S1, unofficial].
    *   `uColorBias`, `uAlphaBias`, `uAlphaTransfer`: Parameters for transfer function biasing [S1, unofficial].
    *   Volume Color Data: Mask/data volumes and transfer functions used to sample `transferSample` [S1, unofficial].
*   **Outputs (Fragment Shader):**
    *   `oColor`: `vec4` containing the accumulated color and alpha [S1, unofficial].
    *   `oOcclusion`: `float` representing the remaining occlusion/transmittance [S1, unofficial].
    *   `oInstanceID`: `uint` for the current instance ID [S1, unofficial].
    *   `oGroupID`: `uint` for the current group ID [S1, unofficial].
    *   `vPosition`: `vec2` vertex shader output representing screen position [S1, unofficial].

**Data Structures**
*   `bGroupMembership`: A `std430` storage buffer (binding 0) containing `uint sGroupMembership[]`, which maps instance IDs to group IDs [S1, unofficial].
*   `oAccColorVolume`: An `rgba8` image3D (binding 1), write-only, used for accumulating color across slices [S1, unofficial].

**Algorithm Steps**
1.  **Initialization (Reset Pass):** Reset outputs to initial values: `oColor = vec4(0)`, `oOcclusion = 1.0`, `oInstanceID = 0u`, `oGroupID = 0u` [S1, unofficial].
2.  **Unprojection:** Unproject the screen position `aPosition` (from vertex shader) and depth `uDepth` to 3D world space:
    $$vPosition3D = \frac{(uMvpInverseMatrix \times vec4(aPosition, uDepth, 1)).xyz}{w}$$
    where $w$ is the $w$-component of the resulting vector [S1, unofficial].
3.  **Boundary Check:** If any component of $vPosition3D$ is outside the $[0, 1]$ range, output the previous color, occlusion, instance ID, and group ID (from the accumulator or previous step) and terminate the shader execution [S1, unofficial].
4.  **Semantic ID Resolution:** Retrieve the current instance and group IDs from 2D position textures using the screen position `vPosition`:
    ```glsl
    uint instanceID = texture(uInstanceID, vPosition).r;
    uint groupID = texture(uGroupID, vPosition).r;
    ```
    If `groupID` is 0 (indicating an unassigned instance):
    1.  Fetch a new instance ID from `uIDVolume` at $vPosition3D$.
    2.  Look up the new group ID from `sGroupMembership` using the fetched instance ID.
    3.  If the new group ID is non-zero, update `oInstanceID` and `oGroupID` with these new values; otherwise, retain the old IDs [S1, unofficial].
5.  **Volume Sampling:** Sample the volume color at $vPosition3D$ using mask/data volumes and transfer functions, applying biases:
    ```glsl
    vec4 transferSample = sampleVolumeColor(vPosition3D);
    ```
    The sampling incorporates `uColorBias`, `uAlphaBias`, and `uAlphaTransfer` [S1, unofficial].
6.  **Accumulator Retrieval:** Sample the accumulator texture at the current screen position to get previous state:
    ```glsl
    vec4 color = texture(uAccumulator, vPosition);
    ```
    Let `prevColor` be this value [S1, unofficial].
7.  **Extinction and Alpha Calculation:** Calculate extinction:
    $$extinction = transferSample.a \times uExtinction$$
    Calculate alpha (opacity) for the current slice:
    $$\alpha = 1.0 - \exp(-extinction \times uSliceDistance)$$
    Here, $uExtinction$ acts as the total extinction coefficient $\sigma_t$ [S1, unofficial].
8.  **Occlusion Calculation:** Compute occlusion attenuation by averaging precomputed occlusion samples. The samples are taken at offsets from `vPosition`:
    $$O = \frac{1}{N} \sum_{i=0}^{N-1} C_i \cdot e^{-\sigma_t \cdot d}$$
    where $C_i$ are the occlusion samples, $\sigma_t$ is the extinction, $d$ is the slice distance ($uSliceDistance$), and $N$ is the number of samples (`uOcclusionSamplesCount`) [S1, unofficial].
    The samples are located at `vPosition + occlusionSampleOffset * uOcclusionScale` [S1, unofficial].
    Update `oOcclusion` using this calculated value [S1, unofficial].
9.  **Color Blending:** Calculate the color contribution:
    $$color = transferSample.rgb \times prevOcclusion \times \alpha$$
    where `prevOcclusion` is `prevColor.a` (or the stored occlusion value) [S1, unofficial].
    Update the output color:
    $$oColor = prevColor + vec4(color \times (1.0 - prevColor.a), \alpha)$$
    Clamp the alpha component:
    $$oColor.a = \min(oColor.a, 1.0)$$
    Finally, mix the accumulated color with a white background:
    $$oColor = mix(vec4(1), vec4(color.rgb, 1), color.a)$$
    [S1, unofficial].
10. **Output:** Write `oColor`, `oOcclusion`, `oInstanceID`, and `oGroupID` to the frame buffer or accumulator texture [S1, unofficial].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def dos_render_fragment(aPosition, uDepth, uMvpInverseMatrix, vPosition, 
                        uInstanceID, uGroupID, uIDVolume, sGroupMembership,
                        uAccumulator, uOcclusion, uOcclusionScale, uOcclusionSamplesCount,
                        uExtinction, uSliceDistance, volume_data, transfer_function,
                        occlusionSampleOffsets):
    # 1. Unproject
    clip_pos = vec4(aPosition.x, aPosition.y, uDepth, 1.0)
    world_pos_vec = mat4_mul_vec4(uMvpInverseMatrix, clip_pos)
    w = world_pos_vec.w
    if w == 0.0:
        return vec4(0.0), 1.0, 0, 0 # Fallback
    
    vPosition3D = vec3(world_pos_vec.x / w, world_pos_vec.y / w, world_pos_vec.z / w)
    
    # 2. Boundary Check
    if not (0.0 <= vPosition3D.x <= 1.0 and 0.0 <= vPosition3D.y <= 1.0 and 0.0 <= vPosition3D.z <= 1.0):
        prev_acc = texture_2d(uAccumulator, vPosition)
        return prev_acc.rgb, prev_acc.a, prev_acc.b, prev_acc.c # Assuming accumulator layout matches outputs
    
    # 3. Semantic ID Resolution
    instanceID = texture_2d(uInstanceID, vPosition).r
    groupID = texture_2d(uGroupID, vPosition).r
    
    if groupID == 0:
        new_instanceID = texture_3d(uIDVolume, vPosition3D).r
        new_groupID = sGroupMembership[new_instanceID]
        if new_groupID != 0:
            instanceID = new_instanceID
            groupID = new_groupID
            
    # 4. Volume Sampling
    # Standard volume texture lookup with transfer function application
    transferSample = sample_volume_color(vPosition3D, volume_data, transfer_function)
    
    # 5. Retrieve Accumulator
    prevColor = texture_2d(uAccumulator, vPosition)
    prevOcclusion = prevColor.a # Assuming alpha channel stores previous occlusion/transmittance
    
    # 6. Extinction and Alpha
    extinction = transferSample.a * uExtinction
    alpha = 1.0 - exp(-extinction * uSliceDistance)
    
    # 7. Occlusion Calculation
    occlusion_sum = 0.0
    for i in range(uOcclusionSamplesCount):
        # Standard offset calculation for precomputed samples
        offset = occlusionSampleOffsets[i] * uOcclusionScale
        sample_pos = vPosition + offset
        C_i = texture_2d(uOcclusion, sample_pos).r
        occlusion_sum += C_i * exp(-extinction * uSliceDistance)
        
    current_occlusion = occlusion_sum / uOcclusionSamplesCount
    
    # 8. Color Blending
    color_contribution = transferSample.rgb * prevOcclusion * alpha
    
    # Update color
    new_color = prevColor + vec4(color_contribution * (1.0 - prevColor.a), alpha)
    new_color.a = min(new_color.a, 1.0)
    
    # Mix with white background
    final_color = mix(vec4(1.0), vec4(new_color.rgb, 1.0), new_color.a)
    
    # 9. Output
    return final_color, current_occlusion, instanceID, groupID
```

**Parameters**
*   `steps`: Number of scattering steps or depth levels [S1, unofficial].
*   `slices`: Number of slices used in the rendering process [S1, unofficial].
*   `extinction`: Extinction coefficient controlling light absorption/scattering intensity [S1, unofficial].
*   `aperture`: Aperture size, likely controlling depth of field blur or scattering spread [S1, unofficial].
*   `samples`: Number of samples used for occlusion or scattering calculations [S1, unofficial].
*   `cutDepth`: Depth cutoff value, likely for clipping or limiting scattering depth [S1, unofficial].
*   `showBoundingBox`: Boolean flag to render a bounding box [S1, unofficial].
*   `showAxes`: Boolean flag to render coordinate axes [S1, unofficial].
*   `boundingBoxColor`: RGB color array for the bounding box [S1, unofficial].

**Edge Cases and Pitfalls**
*   **Group ID 0:** A group ID of 0 indicates an unassigned instance. The shader explicitly checks for this and remaps it using the `sGroupMembership` buffer to prevent rendering artifacts from uninitialized group data [S1, unofficial].
*   **Out of Bounds:** If the 3D position is outside the $[0,1]$ range, the shader must correctly output the previous accumulated state to avoid discarding valid data or introducing black artifacts [S1, unofficial].

## Operating characteristics
*   **Time Complexity:** Not given in the sources.
*   **Memory Complexity:** Not given in the sources.
*   **Strengths:** Integrates semantic segmentation (instance/group IDs) directly into the volume rendering pass, reducing the need for separate post-processing passes [S1, unofficial].
*   **Limitations:** Relies on precomputed occlusion samples, which may limit dynamic occlusion scenarios. The requirement for `std430` buffers and specific texture layouts may increase implementation complexity on some hardware [S1, unofficial].
*   **When to use:** When semantic segmentation is required alongside volume visualization, or when occlusion-aware depth ordering is critical and can be approximated by precomputed samples [S1, unofficial].
*   **When not to use:** For scenes with highly dynamic occlusion that cannot be precomputed, or when hardware does not support `std430` buffers efficiently [S1, unofficial].

## Background
DOSRender is described in the context of the `vpt-conductor` project, an unofficial implementation or toolset related to volume rendering and visualization [S1, unofficial]. The technique appears to be a specialized variant of direct volume ray casting, adapted for semantic segmentation tasks by tracking instance and group IDs [S1, unofficial].

## Sources
[S1] UL-FRI-LGM/vpt-conductor, https://github.com/UL-FRI-LGM/vpt-conductor, fetched 2026-10-26 (unofficial)
