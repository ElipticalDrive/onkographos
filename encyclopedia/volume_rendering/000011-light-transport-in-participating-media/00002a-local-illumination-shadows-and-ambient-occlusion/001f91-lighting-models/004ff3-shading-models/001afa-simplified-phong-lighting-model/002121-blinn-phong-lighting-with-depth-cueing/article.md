# Blinn-Phong Lighting with Depth Cueing

Also called: Volume Rendering with Depth Cueing and Blinn-Phong Lighting

## Overview
This method computes per-voxel illumination using the Blinn-Phong shading model and applies depth cueing to enhance the perception of volume density and depth [S1, unofficial]. It differs from pure [Blinn-Phong Shading Model](2239-blinn-phong-shading-model.md) by integrating lighting directly into the volume compositing loop with depth-based color modulation [S1, unofficial].

## How it works

### Inputs and Outputs
**Inputs:**
- `rgba`: Current voxel color and opacity as a 4-component vector $(R, G, B, \alpha)$, where $\alpha \in [0, 1]$ [S1, unofficial].
- `convolution.gradient`: The spatial gradient vector $\nabla V$ at the current voxel position [S1, unofficial].
- `camera`: Camera parameters including `near_clip_view` (scalar), `far_clip_view` (scalar), `n` (orthographic forward vector), and projection type (orthographic or perspective) [S1, unofficial].
- `pos_world_init`: The world-space position of the ray origin [S1, unofficial].
- `pos_world`: The world-space position of the current voxel [S1, unofficial].
- `pos_view`: The view-space position of the current voxel [S1, unofficial].
- `context.params_light`: Lighting parameters including `k_ambient`, `k_diffuse`, `k_specular`, `p_shininess`, `depth_color_near`, and `depth_color_far` [S1, unofficial].
- `context.transfer_func.alpha_near_one`: Opacity threshold for early ray termination [S1, unofficial].
- `light`: Light source data containing `num` (number of lights), `rgb` (array of light colors), and `xyz` (array of light direction vectors) [S1, unofficial].

**Outputs:**
- `self.result`: The accumulated RGB color and opacity for the ray up to the current step [S1, unofficial].
- `keepgoing`: Boolean indicating whether ray casting should continue (false if early termination occurs) [S1, unofficial].

### Data Structures
- `self.result`: A 4-component vector $(R_{acc}, G_{acc}, B_{acc}, \alpha_{acc})$ storing the accumulated result [S1, unofficial].
- `self.result_curr`: A 4-component vector $(R_{curr}, G_{curr}, B_{curr}, \alpha_{curr})$ storing the lit and depth-cued color for the current voxel [S1, unofficial].
- `self.transparency`: A scalar storing the accumulated transparency $(1 - \alpha_{acc})$ [S1, unofficial].
- `self.result_cache`: A 4-component vector storing the previous step's `self.result` for compositing [S1, unofficial].

### Numbered Steps
1. **Determine Viewer Direction:** If the corrected opacity `rgba[3]` is 0, skip lighting. Otherwise, if the camera is orthographic, set `viewer_dir` to `camera.n`; if perspective, compute `pos_world_dir = pos_world_init - pos_world`, normalize it, and use it as `viewer_dir` [S1, unofficial].
2. **Compute Lit Color:** Call `blinn_phong(rgba[:3], convolution.gradient, viewer_dir, context)` to compute `rgb_lit` [S1, unofficial].
3. **Compute Depth Cueing Factor:** Calculate $\gamma = \frac{-\text{pos\_view}[2, 0] - \text{camera.near\_clip\_view}}{\text{camera.far\_clip\_view} - \text{camera.near\_clip\_view}}$ [S1, unofficial].
4. **Apply Depth Cueing:** Retrieve `dcn` and `dcf` from `context.params_light.depth_color_near` and `context.params_light.depth_color_far`, then compute `color_lerped = dcn + \gamma (dcf - dcn)` [S1, unofficial]. Set `self.result_curr[:3]` to the component-wise product of `rgb_lit` and `color_lerped`, and set `self.result_curr[3]` to `rgba[3]` [S1, unofficial].
5. **Composite:** Call `blend(context)` to composite the current result with the accumulated result, and return `keepgoing` [S1, unofficial].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blinn_phong_lighting_with_depth_cueing(
    rgba, gradient, camera, pos_world_init, pos_world, pos_view, 
    context, light
):
    # 1. Determine Viewer Direction
    if camera.is_orthographic:
        viewer_dir = camera.n
    else:
        pos_world_dir = pos_world_init - pos_world
        norm_val = sum(x**2 for x in pos_world_dir) ** 0.5
        viewer_dir = [x / norm_val for x in pos_world_dir]
    
    # 2. Compute Lit Color
    rgb_lit = blinn_phong(rgba[:3], gradient, viewer_dir, context, light)
    
    # 3. Compute Depth Cueing Factor
    near = camera.near_clip_view
    far = camera.far_clip_view
    depth_val = -pos_view[2, 0]
    
    if far == near:
        gamma = 0.0
    else:
        gamma = (depth_val - near) / (far - near)
    
    # 4. Apply Depth Cueing
    dcn = context.params_light.depth_color_near
    dcf = context.params_light.depth_color_far
    
    color_lerped = [dcn[i] + gamma * (dcf[i] - dcn[i]) for i in range(3)]
    
    if rgba[3] == 0:
        rgb_lit = rgba[:3]
        
    result_curr = [rgb_lit[i] * color_lerped[i] for i in range(3)] + [rgba[3]]
    
    # 5. Composite
    result, keepgoing = blend(result_curr, context)
    
    return result, keepgoing

def blinn_phong(rgb_in, gradient, viewer_dir, context, light):
    k_ambient = context.params_light.k_ambient
    k_diffuse = context.params_light.k_diffuse
    k_specular = context.params_light.k_specular
    p_shininess = context.params_light.p_shininess
    
    rgb_out = [k_ambient * c for c in rgb_in]
    
    if light.num == 0 or (k_ambient == 0 and k_specular == 0):
        return rgb_out
        
    gradient_len = sum(g**2 for g in gradient) ** 0.5
    
    if gradient_len == 0:
        return rgb_out
        
    normal = [-g / gradient_len for g in gradient]
    
    for i in range(light.num):
        light_col = light.rgb[i]
        light_dir = light.xyz[i]
        
        prod = [rgb_in[j] * light_col[j] for j in range(3)]
        scale_diffuse = max(0.0, sum(n * d for n, d in zip(normal, light_dir)))
        diffuse = [k_diffuse * scale_diffuse * p for p in prod]
        
        halfway = [viewer_dir[j] + light_dir[j] for j in range(3)]
        halfway_len = sum(h**2 for h in halfway) ** 0.5
        if halfway_len == 0:
            scale_specular = 0.0
        else:
            halfway_norm = [h / halfway_len for h in halfway]
            scale_specular = max(0.0, sum(n * h for n, h in zip(normal, halfway_norm)))
            scale_specular = scale_specular ** p_shininess
            
        specular = [k_specular * scale_specular * c for c in light_col]
        
        rgb_out = [rgb_out[j] + diffuse[j] + specular[j] for j in range(3)]
        
    return rgb_out

def blend(result_curr, context):
    keepgoing = True
    transparency_curr = 1.0 - result_curr[3]
    
    if context.self.result_cache is None:
        context.self.result = result_curr[:]
        context.self.transparency = transparency_curr
    else:
        rgb_premultiplied = [context.self.result_cache[3] * c for c in context.self.result_cache[:3]]
        rgb_curr = [context.self.transparency * result_curr[3] * c for c in result_curr[:3]]
        rgb_composite = [rgb_premultiplied[j] + rgb_curr[j] for j in range(3)]
        
        context.self.transparency *= transparency_curr
        opacity = 1.0 - context.self.transparency
        
        if opacity == 0:
            context.self.result = [0.0, 0.0, 0.0, 0.0]
        else:
            inv_opacity = 1.0 / opacity
            context.self.result[:3] = [inv_opacity * c for c in rgb_composite]
            context.self.result[3] = opacity
            
        if opacity > context.transfer_func.alpha_near_one:
            context.self.transparency = 0
            keepgoing = False
            
    context.self.result_cache = context.self.result[:]
    return context.self.result, keepgoing
```

### Parameters
- `context.params_light.k_ambient`: Ambient lighting coefficient [S1, unofficial].
- `context.params_light.k_diffuse`: Diffuse lighting coefficient [S1, unofficial].
- `context.params_light.k_specular`: Specular lighting coefficient [S1, unofficial].
- `context.params_light.p_shininess`: Shininess exponent for specular highlight [S1, unofficial].
- `context.params_light.depth_color_near`: Depth cueing color at near clip plane [S1, unofficial].
- `context.params_light.depth_color_far`: Depth cueing color at far clip plane [S1, unofficial].
- `context.transfer_func.alpha_near_one`: Opacity threshold for early ray termination [S1, unofficial].
- `camera.near_clip_view`: Near clip plane distance in view space [S1, unofficial].
- `camera.far_clip_view`: Far clip plane distance in view space [S1, unofficial].

### Edge Cases and Pitfalls
- **Division by Zero (Opacity):** If the accumulated opacity is 0 during compositing, the division $1/\text{opacity}$ is avoided by setting the result to `[0.0, 0.0, 0.0, 0.0]` [S1, unofficial].
- **Zero Gradient:** If the gradient length is 0, the `blinn_phong` function returns immediately to avoid division by zero in normal computation [S1, unofficial].
- **Zero Opacity Voxel:** If the current voxel's opacity `rgba[3]` is 0, lighting is skipped, and the lit RGB color is set to the input RGB [S1, unofficial].

## Operating characteristics

### Time and Memory Complexity
- **Time Complexity:** $O(L)$ per voxel, where $L$ is the number of lights, due to the loop over lights with constant-time vector operations per light [S1, unofficial].
- **Memory Complexity:** $O(1)$ auxiliary memory per voxel, storing the current result, transparency, and cache [S1, unofficial].

### Measured Performance
Not given in the sources.

### Scaling
Not given in the sources.

### Strengths and Limitations
Not given in the sources.

### When to Use
Not given in the sources.

### When Not to Use
Not given in the sources.

## Background

### Origin
Not given in the sources.

### History and Development
Not given in the sources.

### Where It Is Used
Not given in the sources.

## Sources

[S1] RuolinZheng08/volume-rendering, https://github.com/RuolinZheng08/volume-rendering, fetched 2026-10-26 (unofficial)
