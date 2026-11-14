# Gradient-Density Raycasting with Transfer Function Blending

Also called: Gradient-Density Raycasting

## Overview
A fragment-based volume rendering technique that performs raycasting in the fragment shader, sampling density and gradient simultaneously, blending between two transfer functions based on an index, and applying silhouette and gradient magnitude modulation for shading [S1, unofficial]. It differs from standard raycasting by explicitly computing the gradient vector per sample to drive both shading (via `Illuminate`) and opacity modulation (via `SilhouetteModulation` and `GradientMagnitudeModulation`), while using a fixed step count derived from ray length rather than adaptive traversal [S1, unofficial].

## How it works
**Inputs:**
- `volume`: `sampler3D` containing normalized volume data [S1, unofficial].
- `frontfaces`: `sampler2D` containing 3D coordinates of ray entry points in world space [S1, unofficial].
- `backfaces`: `sampler2D` containing 3D coordinates of ray exit points in world space [S1, unofficial].
- `transfer`: `sampler2D` containing transfer functions, with one function stored per row [S1, unofficial].
- `samplepos`: 2D texture coordinate for the current fragment [S1, unofficial].
- `invertedmodel`: 4x4 matrix transforming world space to model space [S1, unofficial].
- `camerapos`: 3D camera position in world space [S1, unofficial].
- `transferindex`: Float index for interpolating between two transfer functions stored in the `transfer` texture [S1, unofficial].

**Outputs:**
- `out_color`: `vec4` output color [S1, unofficial].

**Data Structures:**
- `volume`: 3D texture, normalized data [S1, unofficial].
- `frontfaces`, `backfaces`: 2D textures, RGBA32F, storing 3D world coordinates [S1, unofficial].
- `transfer`: 2D texture, storing transfer functions [S1, unofficial].

**Parameters:**
- `kambient`: 0.3 [S1, unofficial].
- `kdiffuse`: 0.6 [S1, unofficial].
- `kspecular`: 0.5 [S1, unofficial].
- `shininess`: 50.0 [S1, unofficial].

**Steps:**
1. Initialize `out_color` to zero: `out_color = vec4(0.0);` [S1, unofficial].
2. Fetch ray entry and exit points in world space: `world_pos = texture(frontfaces, samplepos).xyz`, `world_exit = texture(backfaces, samplepos).xyz` [S1, unofficial].
3. If `world_pos == world_exit`, return immediately (empty ray) [S1, unofficial].
4. Transform entry and exit points to model space: `model_pos = (invertedmodel * vec4(world_pos, 1.0)).xyz`, `model_exit = (invertedmodel * vec4(world_exit, 1.0)).xyz` [S1, unofficial].
5. Calculate ray directions in world and model space: `world_dir = world_exit - world_pos`, `model_dir = model_exit - model_pos` [S1, unofficial].
6. Calculate number of steps: `nstep = length(model_dir) * 2.0` [S1, unofficial].
7. Calculate per-step increments in world and model space: `world_step = world_dir / nstep`, `model_step = model_dir / nstep` [S1, unofficial].
8. Convert model space position and step to normalized texture coordinates: `volumesize = vec3(textureSize(volume, 0))`, `model_pos = (model_pos + vec3(0.5)) / volumesize`, `model_step /= volumesize` [S1, unofficial].
9. Get the height of the transfer function texture: `maxy = float(textureSize(transfer, 0).y)` [S1, unofficial].
10. Loop `nstep` times:
    a. Sample density and gradient at current model position: `tmp = GradientDensity(model_pos)`, `density = tmp.w`, `gradient = tmp.xyz` [S1, unofficial].
    b. Calculate transfer function coordinates for bottom and top rows: `cordbottom = vec2(density, transferindex / maxy)`, `cordtop = vec2(density, (transferindex + 1.0) / maxy)` [S1, unofficial].
    c. Sample transfer function at bottom and top, then interpolate based on fractional part of transfer index: `texbottom = texture(transfer, cordbottom)`, `textop = texture(transfer, cordtop)`, `texel = mix(texbottom, textop, fract(transferindex))` [S1, unofficial].
    d. Calculate view direction: `viewdir = world_pos - camerapos` [S1, unofficial].
    e. Apply illumination to RGB channel: `texel.rgb *= Illuminate(world_pos, viewdir, gradient)` [S1, unofficial].
    f. Modulate alpha by gradient magnitude: `texel.a *= GradientMagnitudeModulation(gradient)` [S1, unofficial].
    g. Modulate alpha by silhouette effect: `texel.a *= SilhouetteModulation(gradient, viewdir)` [S1, unofficial].
    h. Composite the sample using front-to-back blending with a fixed step weight of 0.5: `out_color += (1.0 - out_color.a) * texel * 0.5` [S1, unofficial].
    i. Break loop if accumulated alpha reaches 0.9: `if (out_color.a >= 0.9) { break; }` [S1, unofficial].
    j. Advance ray position in world and model space: `world_pos += world_step`, `model_pos += model_step` [S1, unofficial].

**Formulas:**
- **Gradient Calculation:**
  - `value.x = a - b` (x-component), `value.w = a + b` (initial density sum) [S1, unofficial].
  - Add `a + b` for y and z axes to `value.w` [S1, unofficial].
  - `value.w /= 6.0` [S1, unofficial].
  - Where `a` and `b` are texture samples at `position + (1,0,0)` and `position - (1,0,0)` respectively for x, and similarly for y and z [S1, unofficial].
- **Silhouette Modulation:**
  - `opacity = 0.9f + 15.0f * pow(1.0 - abs(dot(gradient, viewdir)), 0.25)` [S1, unofficial].
  - Clamped to 1.0 [S1, unofficial].
- **Gradient Magnitude Modulation:**
  - `opacity = 0.8 + 5.0 * magnitude` [S1, unofficial].
  - Where `magnitude = length(gradient)` [S1, unofficial].
  - Clamped to 1.0 [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Inputs: volume, frontfaces, backfaces, transfer, samplepos, invertedmodel, camerapos, transferindex
# Outputs: out_color

out_color = vec4(0.0)

# Fetch ray entry and exit points in world space
world_pos = texture(frontfaces, samplepos).xyz
world_exit = texture(backfaces, samplepos).xyz

# If entry equals exit, return immediately (empty ray)
if world_pos == world_exit:
    return

# Transform entry and exit points to model space
model_pos = (invertedmodel * vec4(world_pos, 1.0)).xyz
model_exit = (invertedmodel * vec4(world_exit, 1.0)).xyz

# Calculate ray directions in world and model space
world_dir = world_exit - world_pos
model_dir = model_exit - model_pos

# Calculate number of steps as twice the length of the model space ray direction
nstep = length(model_dir) * 2.0

# Calculate per-step increments in world and model space
world_step = world_dir / nstep
model_step = model_dir / nstep

# Convert model space position and step to normalized texture coordinates
volumesize = vec3(textureSize(volume, 0))
model_pos = (model_pos + vec3(0.5)) / volumesize
model_step /= volumesize

# Get the height of the transfer function texture
maxy = float(textureSize(transfer, 0).y)

for i in range(int(nstep)):
    # Sample density and gradient at current model position
    tmp = GradientDensity(model_pos)
    density = tmp.w
    gradient = tmp.xyz

    # Calculate transfer function coordinates for bottom and top rows
    cordbottom = vec2(density, transferindex / maxy)
    cordtop = vec2(density, (transferindex + 1.0) / maxy)

    # Sample transfer function at bottom and top, then interpolate
    texbottom = texture(transfer, cordbottom)
    textop = texture(transfer, cordtop)
    texel = mix(texbottom, textop, fract(transferindex))

    # Calculate view direction
    viewdir = world_pos - camerapos

    # Apply illumination to RGB channel
    texel.rgb *= Illuminate(world_pos, viewdir, gradient)

    # Modulate alpha by gradient magnitude
    texel.a *= GradientMagnitudeModulation(gradient)

    # Modulate alpha by silhouette effect
    texel.a *= SilhouetteModulation(gradient, viewdir)

    # Composite the sample using front-to-back blending with a fixed step weight of 0.5
    out_color += (1.0 - out_color.a) * texel * 0.5

    # Break loop if accumulated alpha reaches 0.9
    if out_color.a >= 0.9:
        break

    # Advance ray position in world and model space
    world_pos += world_step
    model_pos += model_step

def GradientDensity(position):
    # Sample neighbors for gradient and density
    # x-axis
    a_x_pos = texture(volume, position + vec3(1.0, 0.0, 0.0))
    b_x_pos = texture(volume, position - vec3(1.0, 0.0, 0.0))
    value_x = a_x_pos - b_x_pos
    sum_density = a_x_pos + b_x_pos

    # y-axis
    a_y_pos = texture(volume, position + vec3(0.0, 1.0, 0.0))
    b_y_pos = texture(volume, position - vec3(0.0, 1.0, 0.0))
    value_y = a_y_pos - b_y_pos
    sum_density += a_y_pos + b_y_pos

    # z-axis
    a_z_pos = texture(volume, position + vec3(0.0, 0.0, 1.0))
    b_z_pos = texture(volume, position - vec3(0.0, 0.0, 1.0))
    value_z = a_z_pos - b_z_pos
    sum_density += a_z_pos + b_z_pos

    # Normalize density sum
    sum_density /= 6.0

    return vec4(value_x, value_y, value_z, sum_density)

def Illuminate(world_pos, viewdir, gradient):
    # Standard Phong illumination with given coefficients
    # kambient = 0.3, kdiffuse = 0.6, kspecular = 0.5, shininess = 50.0
    # Assuming a standard light direction, e.g., (0.0, 0.0, 1.0) or normalized gradient
    # Note: The exact light direction is not specified in the source, so we assume a standard setup
    # or that the gradient is used as the normal for diffuse/specular calculations
    normal = normalize(gradient)
    view_dir_normalized = normalize(viewdir)
    
    # Assuming a light direction L, e.g., from the camera or a fixed direction
    # Since the source does not specify L, we assume a standard diffuse/specular model
    # where L is normalized and dot(N, L) is used
    # For simplicity, let's assume L is (0, 0, 1) in world space, normalized
    light_dir = normalize(vec3(0.0, 0.0, 1.0))
    
    diff = max(dot(normal, light_dir), 0.0)
    spec = pow(max(dot(reflect(-light_dir, normal), view_dir_normalized), 0.0), shininess)
    
    color = vec3(kambient) + vec3(kdiffuse) * diff + vec3(kspecular) * spec
    return color

def GradientMagnitudeModulation(gradient):
    magnitude = length(gradient)
    opacity = 0.8 + 5.0 * magnitude
    return clamp(opacity, 0.0, 1.0)

def SilhouetteModulation(gradient, viewdir):
    opacity = 0.9 + 15.0 * pow(1.0 - abs(dot(normalize(gradient), normalize(viewdir))), 0.25)
    return clamp(opacity, 0.0, 1.0)
```

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of steps per ray, which is proportional to the length of the ray in model space [S1, unofficial].
- **Memory Complexity:** Constant per fragment, dependent on the size of the textures and the number of steps [S1, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Not given in the sources.
- **Strengths:** Simultaneous sampling of density and gradient allows for detailed shading and opacity modulation [S1, unofficial]. Fixed step count simplifies implementation [S1, unofficial].
- **Limitations:** Fixed step count may lead to aliasing or inefficiency if not properly tuned [S1, unofficial]. The method relies on pre-computed front and back face textures, which may require additional rendering passes [S1, unofficial].
- **When to use:** When detailed shading and opacity modulation based on gradient is required [S1, unofficial].
- **When not to use:** When adaptive ray marching or more efficient traversal schemes are needed [S1, unofficial].

## Background
Origin: The method is described in an unofficial source from a GitHub repository [S1, unofficial]. History and development: Not given in the sources. Where it is used: Not given in the sources.

## Sources
[S1] mversiotech/volume-rendering-demo, https://github.com/mversiotech/volume-rendering-demo, fetched 2026-10-26 (unofficial)
