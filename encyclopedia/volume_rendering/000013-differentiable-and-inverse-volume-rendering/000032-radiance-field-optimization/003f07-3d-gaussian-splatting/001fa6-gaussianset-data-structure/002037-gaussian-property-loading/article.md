# Gaussian Property Loading

Also called: PLY Property Loading with Fallbacks, PLY Gaussian Loading

## Overview
Gaussian Property Loading parses 3D Gaussian Splatting scene data from a Polygon File Format (PLY) file, decoding raw vertex properties into a structured Gaussian representation suitable for rendering [S1, S2, S3, S4, unofficial]. It solves the problem of converting the standardized, often compressed or logarithmically encoded PLY storage format into the explicit mathematical parameters (position, scale, rotation, color, opacity) required by the rasterization pipeline [S1, S2, S3, S4, unofficial]. It differs from its siblings by focusing on the initial ingestion and decoding of the PLY file into the `GaussianData` structure, whereas [Coordinate System Conversion for Gaussian Splats](3935-coordinate-system-conversion-for-gaussian-splats.md) handles spatial alignment and [INRIA V1 PLY Splat Parsing](6822-inria-v1-ply-splat-parsing.md) addresses specific legacy format variations.

## How it works
**Inputs and Outputs:**
- Input: A `.ply` file containing vertex elements with properties for position, rotation, scale, opacity, and spherical harmonics (SH) or RGB color.
- Output: A dictionary or struct containing decoded Gaussian properties: `xyz` (N,3), `rot` (N,4), `scale` (N,3), `sh` (N, total SH coefficients), `opacity` (N,1), and `dc_colours` (N,3) [S2, S4, unofficial].

**Data Structures:**
The output is typically a dictionary or struct with the following fields [S2, S4, unofficial]:
- `xyz`: Float32 array of shape (N, 3) representing mean positions.
- `rot`: Float32 array of shape (N, 4) representing quaternions (w, x, y, z).
- `scale`: Float32 array of shape (N, 3) representing log-scales (after decoding).
- `sh`: Float32 array of shape (N, K) where K is the total number of SH coefficients.
- `opacity`: Float32 array of shape (N, 1) representing opacity values.
- `dc_colours`: Float32 array of shape (N, 3) representing the DC term of the SH converted to RGB.

**Numbered Steps:**

1. **Open and Parse Header:** Open the PLY file in binary or ASCII mode. Parse the header to identify the file type (ASCII, binary little-endian, or binary big-endian) and property types [S4, unofficial]. Identify the vertex element and request the following properties:
   - Position: `x`, `y`, `z` [S1, S2, S3, S4, unofficial].
   - Scale: Try `scale_0`, `scale_1`, `scale_2`. If not found, fallback to `scaling_0`, `scaling_1`, `scaling_2` [S1, S2, S4, unofficial].
   - Rotation: Try `rot_0`, `rot_1`, `rot_2`, `rot_3`. If not found, fallback to `rotation_0`, `rotation_1`, `rotation_2`, `rotation_3` [S1, S2, S4, unofficial].
   - Color/SH: Try `f_dc_0`, `f_dc_1`, `f_dc_2` (indicating SH presence). If not found, fallback to `red`, `green`, `blue` (RGB). If neither, default to white [S1, S2, S4, unofficial].
   - Opacity: Try `opacity`. If not found, default to full opacity (1.0) [S1, S2, S4, unofficial].
   - Extra SH: Identify extra feature names starting with `f_rest_` [S2, S3, S4, unofficial].

2. **Extract Raw Data:** Read the vertex data. Extract `xyz` coordinates and stack them into a shape (N, 3) [S2, S3, S4, unofficial]. Extract scale, rotation, and opacity values. Cast all extracted values to float32 [S2, S4, unofficial].

3. **Decode Scale:** The stored scale values are typically log-scales. Apply the exponential function to decode them:
   $$ \text{scale}_i = \exp(\text{stored\_scale}_i) $$
   for $i \in \{0, 1, 2\}$ [S1, S2, S4, unofficial].

4. **Decode Opacity:** The stored opacity values are typically pre-activation logits. Apply the sigmoid function to decode them:
   $$ \text{opacity} = \frac{1}{1 + \exp(-\text{stored\_opacity})} $$
   [S1, S2, S4, unofficial].

5. **Decode Rotation:** The stored rotation values are quaternions. Normalize them to ensure they represent valid rotations:
   $$ \text{rot} = \frac{\text{stored\_rot}}{\|\text{stored\_rot}\|} $$
   [S2, S4, unofficial].

6. **Decode Spherical Harmonics (SH):**
   - Identify the maximum SH degree (`max_sh_degree`). This is often determined by the number of `f_rest_` fields. The number of extra features should equal $3 \times ((\text{max\_sh\_degree} + 1)^2 - 1)$ [S2, S3, S4, unofficial].
   - Extract `f_dc_0`, `f_dc_1`, `f_dc_2` into the first 3 SH coefficients (DC term) [S1, S2, S3, S4, unofficial].
   - Extract `f_rest_N` coefficients starting at index 3 [S1, S2, S3, S4, unofficial].
   - Reshape and transpose the SH coefficients into a shape of (N, (max_sh_degree + 1)^2, 3) or a flattened version (N, total SH coefficients) [S2, S3, S4, unofficial].

7. **Compute DC Colours:** Extract the first 3 channels of the SH coefficients (`dc_vals`). Apply the following formula to convert to RGB:
   $$ \text{dc\_colours} = \max(\text{dc\_vals} \times C_0 + 0.5, 0) $$
   where $C_0$ is the SH constant for degree 0, typically $0.28209479177387814$ [S2, S4, unofficial].

8. **Final Assembly:** Concatenate `features_dc` and `features_extra` if necessary to form the final `sh` array. Ensure all arrays are contiguous and cast to float32 [S2, S4, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import numpy as np
import torch

def load_gaussians_from_ply(ply_path):
    """
    Loads Gaussian properties from a PLY file.
    Returns a dictionary with keys: 'xyz', 'rot', 'scale', 'sh', 'opacity', 'dc_colours'.
    """
    # Step 1: Open and Parse Header
    # Use a PLY parser (e.g., tinyply, ply_rs, miniply) to read the file.
    # Identify file type and property types.
    # Request properties: x, y, z, scale_*/scaling_*, rot_*/rotation_*, f_dc_*/red/green/blue, opacity, f_rest_*
    
    # Step 2: Extract Raw Data
    xyz = np.array([data['x'], data['y'], data['z']]).T.astype(np.float32) # Shape (N, 3)
    
    # Handle scale fallback
    if 'scale_0' in data:
        scale_raw = np.array([data['scale_0'], data['scale_1'], data['scale_2']]).T.astype(np.float32)
    else:
        scale_raw = np.array([data['scaling_0'], data['scaling_1'], data['scaling_2']]).T.astype(np.float32)
        
    # Handle rotation fallback
    if 'rot_0' in data:
        rot_raw = np.array([data['rot_0'], data['rot_1'], data['rot_2'], data['rot_3']]).T.astype(np.float32)
    else:
        rot_raw = np.array([data['rotation_0'], data['rotation_1'], data['rotation_2'], data['rotation_3']]).T.astype(np.float32)
        
    # Handle opacity
    if 'opacity' in data:
        opacity_raw = np.array(data['opacity']).astype(np.float32).reshape(-1, 1)
    else:
        opacity_raw = np.ones((xyz.shape[0], 1), dtype=np.float32)
        
    # Handle color/SH
    has_sh = 'f_dc_0' in data
    has_rgb = 'red' in data
    
    if has_sh:
        f_dc = np.array([data['f_dc_0'], data['f_dc_1'], data['f_dc_2']]).T.astype(np.float32) # Shape (N, 3)
        # Extract f_rest_*
        f_rest_names = sorted([k for k in data.keys() if k.startswith('f_rest_')], key=lambda x: int(x.split('_')[-1]))
        f_rest = np.array([data[name] for name in f_rest_names]).T.astype(np.float32)
        
        # Determine max_sh_degree
        # Number of extra features = 3 * ((max_sh_degree + 1)^2 - 1)
        num_extra_features = len(f_rest_names)
        # Solve for max_sh_degree: (max_sh_degree + 1)^2 = num_extra_features / 3 + 1
        max_sh_degree = int(np.sqrt(num_extra_features / 3 + 1) - 1)
        
        # Reshape f_rest to (N, 3, (max_sh_degree + 1)^2 - 1)
        f_rest = f_rest.reshape(-1, 3, (max_sh_degree + 1)**2 - 1)
        # Transpose to (N, (max_sh_degree + 1)^2 - 1, 3)
        f_rest = np.transpose(f_rest, (0, 2, 1))
        
        # Concatenate f_dc and f_rest
        # f_dc needs to be reshaped to (N, 1, 3) to match the SH dimension structure
        f_dc_reshaped = f_dc.reshape(-1, 1, 3)
        sh = np.concatenate([f_dc_reshaped, f_rest], axis=1) # Shape (N, (max_sh_degree + 1)^2, 3)
        
        # Compute dc_colours
        C0 = 0.28209479177387814
        dc_vals = f_dc # Shape (N, 3)
        dc_colours = np.maximum(dc_vals * C0 + 0.5, 0.0)
        
    elif has_rgb:
        # Fallback to RGB
        sh = np.zeros((xyz.shape[0], 1, 3), dtype=np.float32) # Placeholder for SH structure
        dc_colours = np.array([data['red'], data['green'], data['blue']]).T.astype(np.float32)
        # Note: RGB values in PLY are often in [0, 1] or [0, 255]. Adjust if necessary.
        # Assuming [0, 1] based on typical PLY float formats.
        
    else:
        # Default to white
        sh = np.zeros((xyz.shape[0], 1, 3), dtype=np.float32)
        dc_colours = np.ones((xyz.shape[0], 3), dtype=np.float32)
        
    # Step 3: Decode Scale
    scale = np.exp(scale_raw)
    
    # Step 4: Decode Opacity
    opacity = 1.0 / (1.0 + np.exp(-opacity_raw))
    
    # Step 5: Decode Rotation
    rot = rot_raw / np.linalg.norm(rot_raw, axis=1, keepdims=True)
    
    # Step 6: Final Assembly
    # Ensure sh is flattened if required by the downstream pipeline
    # For example, if the output expects (N, total_SH_coeffs)
    total_sh_coeffs = sh.shape[1] * sh.shape[2]
    sh_flat = sh.reshape(xyz.shape[0], -1)
    
    return {
        'xyz': xyz,
        'rot': rot,
        'scale': scale,
        'sh': sh_flat,
        'opacity': opacity,
        'dc_colours': dc_colours
    }
```

**Parameters:**
- `max_sh_degree`: Maximum spherical harmonic degree, typically set to 3 for high-quality rendering [S2, S4, unofficial].
- `C0`: SH constant for degree 0, value $0.28209479177387814$ [S4, unofficial].

**Edge Cases and Pitfalls:**
- **Coordinate System:** PLY files may use a different coordinate system (e.g., Right-handed Down-Forward) than the rendering engine (e.g., Right-handed Up-Back). Coordinate system conversion must be applied after loading [S1, S3, S4, unofficial].
- **Endianness:** Binary PLY files can be little-endian or big-endian. The parser must handle both [S4, unofficial].
- **Missing Properties:** Fallbacks are necessary for scale (`scaling_*`), rotation (`rotation_*`), and color (`red/green/blue`) [S1, S2, S4, unofficial].
- **Precision:** Ensure all values are cast to float32 to maintain consistency with the rendering pipeline [S2, S4, unofficial].

## Subtopics

### [Gaussian Splat PLY Loader](3254-gaussian-splat-ply-loader.md)
The Gaussian Splat PLY Loader parses 3D Gaussian Splatting scene data from a Polygon File Format (PLY) file, decoding raw vertex properties into a structured Gaussian representation suitable for rendering. It solves the problem of converting the standardized, often compressed or logarithmically encoded PLY storage format into the explicit mathematical parameters (position, scale, rotation, color, opacity) required by the rasterization pipeline. It differs from its siblings by focusing on the initial ingestion and decoding of the PLY file into the `GaussianData` structure, whereas [Coordinate System Conversion for Gaussian Splats](3935-coordinate-system-conversion-for-gaussian-splats.md) handles spatial alignment and [INRIA V1 PLY Splat Parsing](6822-inria-v1-ply-splat-parsing.md) addresses specific legacy format variations.

### [Coordinate System Conversion for Gaussian Splats](3935-coordinate-system-conversion-for-gaussian-splats.md)
This method transforms Gaussian splat attributes (position, rotation, spherical harmonics) between different coordinate systems, such as converting from the Right-handed Up-Back (RUB) format used internally to the Right-handed Down-Forward (RDF) format used in PLY files. It sets apart from sibling loaders by handling the specific axis flips and rotations required for quaternion and spherical harmonic consistency, ensuring that the rendered output remains invariant under coordinate system changes.

### [INRIA V1 PLY Splat Parsing](6822-inria-v1-ply-splat-parsing.md)
The INRIA V1 PLY Splat Parsing method decodes the text-based `.ply` files generated by the original INRIA 3D Gaussian Splatting implementation into an uncompressed, contiguous array of Gaussian properties suitable for rendering or further processing. It solves the problem of interpreting variable-length spherical harmonic (SH) fields and applying specific activation functions (sigmoid for opacity, exponential for scale) and coordinate system transformations required by downstream engines. Unlike the standard `.splat` binary format which uses fixed-width records, this method handles the PLY header's variable field counts to dynamically determine the SH degree (0, 1, or 2) and extract the corresponding coefficients.

### [3D Gaussian Splatting Model Preprocessing and Grid Partitioning](6908-3d-gaussian-splatting-model-preprocessing-and-grid-partition.md)
This method preprocesses a raw 3D Gaussian Splatting model stored in PLY format for use in a specific rendering engine (Unreal Engine). It parses the PLY file, converts coordinate systems and property encodings to match the engine's internal representation, and partitions the Gaussians into a spatial grid for efficient culling and rendering. It differs from generic loaders by applying specific axis swaps, exponential scale decoding, and sigmoid opacity activation required by the target engine's shader pipeline.

## Operating characteristics
- **Time Complexity:** Linear with respect to the number of Gaussians $N$, $O(N)$, as each Gaussian's properties are read and decoded independently [S1, S2, S3, S4, unofficial].
- **Memory Complexity:** Linear with respect to the number of Gaussians $N$, $O(N)$, storing position, rotation, scale, opacity, and SH coefficients [S2, S4, unofficial].
- **Measured Performance:** Not given in the sources.
- **Scaling:** Scales linearly with the number of Gaussians. The parsing time is dominated by I/O and the decoding of SH coefficients, which increases with the SH degree [S2, S4, unofficial].
- **Strengths:** Handles various PLY formats and property naming conventions through fallbacks. Decodes log-scale and sigmoid-activated opacity correctly. Supports variable SH degrees [S1, S2, S4, unofficial].
- **Limitations:** Requires a PLY parser library. Coordinate system conversion is a separate step. Does not handle compression or other non-standard PLY extensions [S1, S2, S4, unofficial].
- **When to use:** When loading pre-trained 3D Gaussian Splatting models from PLY files for rendering or further processing [S1, S2, S3, S4, unofficial].
- **When not:** When working with other data formats (e.g., `.splat`, `.ply` with different structures) or when real-time loading is critical and a more optimized binary format is available [S3, S4, unofficial].

## Background
3D Gaussian Splatting (3DGS) was introduced by Kerbl et al. in 2023 as a novel method for real-time radiance field rendering using 3D Gaussians [S1, S2, S3, S4, unofficial]. The Gaussian Property Loading method is a critical component of the 3DGS pipeline, enabling the ingestion of pre-trained models stored in the PLY format. The PLY format was chosen for its simplicity and widespread support in graphics applications. The method has been implemented in various open-source projects, including those by Thomas Antony, Fhz2346, liaozhangsheng, and JI20 [S1, S2, S3, S4, unofficial].

## Sources
[S1] thomasantony/splat, https://github.com/thomasantony/splat, fetched 2026-10-27 (unofficial)
[S2] Fhz2346/3D-Gaussian-Splatting, https://github.com/Fhz2346/3D-Gaussian-Splatting, fetched 2026-10-27 (unofficial)
[S3] liaozhangsheng/gsplat-opengl, https://github.com/liaozhangsheng/gsplat-opengl, fetched 2026-10-28 (unofficial)
[S4] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
