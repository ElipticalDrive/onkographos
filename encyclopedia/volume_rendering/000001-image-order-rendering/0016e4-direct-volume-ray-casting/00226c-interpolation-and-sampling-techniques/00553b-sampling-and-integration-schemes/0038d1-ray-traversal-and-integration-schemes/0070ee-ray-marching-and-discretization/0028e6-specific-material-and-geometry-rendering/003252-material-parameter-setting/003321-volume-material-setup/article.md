# Volume Material Setup

Also called: ApplyToMaterial, Material Initialization, Shader Parameter Binding, Volume Material Setup (Affine, Weights, Label Map), initMaterial

## Overview
Volume Material Setup initializes the rendering state for direct volume ray casting by creating and caching a material object configured for transparent volumetric rendering [S1, S2, unofficial]. It binds necessary textures (noise, fire profiles, transfer functions, labels, weights) and uniform parameters (affine transforms, intensity ranges) to the GPU, enabling the shader to correctly sample and composite volume data [S1, S2, unofficial]. It differs from sibling methods by focusing on the binding and configuration of material resources rather than the traversal logic or the transfer function definition itself [S1, S2, unofficial].

## How it works
The method constructs a material object that encapsulates all shader inputs required for volume rendering [S1, S2, unofficial]. It handles the selection of textures based on availability, computes inverse affine matrices for coordinate transformation, and sets flags to inform the shader about the data type (e.g., label map vs. continuous) [S1, unofficial].

### Inputs and Outputs
*   **Inputs**:
    *   `d`: Data object containing metadata (`d.meta`), transfer function (`d.tf`), and optional texture references (`d.labelTexture`, `d.weightTexture`) [S1, unofficial].
    *   `volumeTexLabels`: Optional pre-loaded 3D texture for label data [S1, unofficial].
    *   `volumeTexWeights`: Optional pre-loaded 3D texture for weight data [S1, unofficial].
    *   `tfTexCurrent`: The current transfer function texture [S1, unofficial].
    *   `vs`, `fs`: Vertex and fragment shader source code strings [S1, S2, unofficial].
*   **Outputs**:
    *   `material`: An initialized material object ready for assignment to a mesh renderer [S1, S2, unofficial].

### Data Structures and Layouts

**Uniforms Object Layout**
The uniforms passed to the shader are structured as follows [S1, S2, unofficial]:

| Uniform Name | Type | Value/Source | Description |
| :--- | :--- | :--- | :--- |
| `nzw` | `t` (Texture) | `nzw.png` | Noise texture for turbulence. |
| `fireProfile` | `t` (Texture) | `firetex.png` | Fire profile texture. |
| `time` | `f` (Float) | `1.0` | Time parameter for animation. |
| `_TFTex` | `t` (Texture) | `tfTexCurrent` | Transfer function texture. |
| `_VolumeTexLabels` | `t` (Texture) | Selected label texture | Label map texture. |
| `_VolumeTexWeights` | `t` (Texture) | Selected weight texture | Weight texture (or black if null). |
| `_Affine` | `m4` (Matrix4) | `affine` | 4x4 affine transformation matrix. |
| `_InvAffine` | `m4` (Matrix4) | `affine.inverse` | Inverse affine transformation matrix. |
| `_Dim` | `v4` (Vector4) | `(dimX, dimY, dimZ, 1.0)` | Volume dimensions. |
| `_P1` | `f` (Float) | `p1` | Lower bound of intensity range. |
| `_P99` | `f` (Float) | `p99` | Upper bound of intensity range. |
| `_IsLabelMap` | `i` (Int) | `0` or `1` | Flag indicating label map usage. |
| `_HasWeights` | `i` (Int) | `0` or `1` | Flag indicating weight texture presence. |

**Texture Configuration**
*   `nzw`: `RepeatWrapping` on S and T axes; `LinearFilter` for mag/min [S2, unofficial].
*   `fireProfile`: `ClampToEdgeWrapping` on S and T axes; `LinearFilter` for mag/min [S2, unofficial].

### Numbered Steps

1.  **Extract Dimensions**: Read `dimX`, `dimY`, `dimZ` from `d.meta.dim` [S1, unofficial].
2.  **Initialize Intensity Range**: Set `p1 = 0.0`, `p99 = 1.0`. If `d.tf.type == "continuous"` and `d.meta.intensity_range` exists with length 2, set `p1 = d.meta.intensity_range[0]` and `p99 = d.meta.intensity_range[1]` [S1, unofficial].
3.  **Load Noise Texture**: Load `nzw.png`. Set wrapping to `RepeatWrapping` and filtering to `LinearFilter` [S2, unofficial].
4.  **Load Fire Profile Texture**: Load `firetex.png`. Set wrapping to `ClampToEdgeWrapping` and filtering to `LinearFilter` [S2, unofficial].
5.  **Compute Affine Matrices**: Construct `affine` matrix from `d.meta.affine` (4x4). Compute `invAffine = affine.inverse()` [S1, unofficial].
6.  **Select Label Texture**: If `volumeTexLabels` is provided, use it. Else if `d.labelTexture` is provided, use it. Else, use a default black 3D texture (`BlackTex3D`) [S1, unofficial]. Bind to `_VolumeTexLabels`.
7.  **Select Weight Texture**: If `volumeTexWeights` is provided, use it. Else if `d.weightTexture` is provided, use it. Else, set `texW = null`. Bind `texW` to `_VolumeTexWeights` or `BlackTex3D`. Set `_HasWeights = 1` if `texW != null`, else `0` [S1, unofficial].
8.  **Bind Transfer Function**: Bind `tfTexCurrent` to `_TFTex`. Set `_IsLabelMap = 1` if `d.tf.type == "labelmap"`, else `0` [S1, unofficial].
9.  **Set Uniforms**: Set `_P1`, `_P99`, `_Dim` (Vector4(dimX, dimY, dimZ, 1.0)), `_Affine`, `_InvAffine` [S1, unofficial].
10. **Create Material**: Instantiate `RawShaderMaterial` with `vs`, `fs`, `uniforms`, `side: DoubleSide`, `blending: AdditiveBlending`, `transparent: true` [S1, S2, unofficial].
11. **Assign Material**: Assign `material` to `MeshRenderer.sharedMaterial` [S1, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Helper definitions to ensure self-contained pseudocode
def load_texture(path):
    """Returns a texture object with wrapS, wrapT, magFilter, minFilter attributes."""
    pass

class Matrix4:
    def __init__(self, data): self.data = data
    def inverse(self): return Matrix4(self.data)

class Vector4:
    def __init__(self, x, y, z, w): self.data = (x, y, z, w)

class RawShaderMaterial:
    def __init__(self, vertexShader, fragmentShader, uniforms, side, blending, transparent):
        self.vertexShader = vertexShader
        self.fragmentShader = fragmentShader
        self.uniforms = uniforms
        self.side = side
        self.blending = blending
        self.transparent = transparent

BlackTex3D = "default_black_texture"

def init_material(d, volumeTexLabels=None, volumeTexWeights=None, tfTexCurrent=None, vs=None, fs=None):
    # Step 1: Extract dimensions
    dimX, dimY, dimZ = d.meta.dim[0], d.meta.dim[1], d.meta.dim[2]
    
    # Step 2: Initialize intensity range
    p1 = 0.0
    p99 = 1.0
    if d.tf.type == "continuous" and hasattr(d.meta, 'intensity_range') and len(d.meta.intensity_range) == 2:
        p1 = d.meta.intensity_range[0]
        p99 = d.meta.intensity_range[1]
        
    # Step 3: Load noise texture
    nzw = load_texture('nzw.png')
    nzw.wrapS = 'REPEAT'
    nzw.wrapT = 'REPEAT'
    nzw.magFilter = 'LINEAR'
    nzw.minFilter = 'LINEAR'
    
    # Step 4: Load fire profile texture
    fireProfile = load_texture('firetex.png')
    fireProfile.wrapS = 'CLAMP_TO_EDGE'
    fireProfile.wrapT = 'CLAMP_TO_EDGE'
    fireProfile.magFilter = 'LINEAR'
    fireProfile.minFilter = 'LINEAR'
    
    # Step 5: Compute affine matrices
    affine = Matrix4(d.meta.affine)
    invAffine = affine.inverse()
    
    # Step 6: Select label texture
    if volumeTexLabels is not None:
        texLbl = volumeTexLabels
    elif hasattr(d, 'labelTexture') and d.labelTexture is not None:
        texLbl = d.labelTexture
    else:
        texLbl = BlackTex3D
        
    # Step 7: Select weight texture
    if volumeTexWeights is not None:
        texW = volumeTexWeights
    elif hasattr(d, 'weightTexture') and d.weightTexture is not None:
        texW = d.weightTexture
    else:
        texW = None
        
    hasWeights = 1 if texW is not None else 0
    texW_for_uniform = texW if texW is not None else BlackTex3D
        
    # Step 8: Bind Transfer Function and Flags
    isLabelMap = 1 if d.tf.type == "labelmap" else 0
        
    # Step 9: Create Uniforms Object
    uniforms = {
        'nzw': {'type': 't', 'value': nzw},
        'fireProfile': {'type': 't', 'value': fireProfile},
        'time': {'type': 'f', 'value': 1.0},
        '_TFTex': {'type': 't', 'value': tfTexCurrent},
        '_VolumeTexLabels': {'type': 't', 'value': texLbl},
        '_VolumeTexWeights': {'type': 't', 'value': texW_for_uniform},
        '_Affine': {'type': 'm4', 'value': affine},
        '_InvAffine': {'type': 'm4', 'value': invAffine},
        '_Dim': {'type': 'v4', 'value': Vector4(dimX, dimY, dimZ, 1.0)},
        '_P1': {'type': 'f', 'value': p1},
        '_P99': {'type': 'f', 'value': p99},
        '_IsLabelMap': {'type': 'i', 'value': isLabelMap},
        '_HasWeights': {'type': 'i', 'value': hasWeights}
    }
    
    # Step 10: Instantiate Material
    material = RawShaderMaterial(
        vertexShader=vs,
        fragmentShader=fs,
        uniforms=uniforms,
        side='DOUBLE',
        blending='ADDITIVE',
        transparent=True
    )
    
    # Step 11: Assign Material
    # MeshRenderer.sharedMaterial = material
    return material
```

### Parameters and Typical Values
*   `_P1`, `_P99`: Floats representing the intensity range for continuous transfer functions. Default: `0.0`, `1.0` [S1, unofficial].
*   `_IsLabelMap`: Integer flag (`0` or `1`). `1` if `d.tf.type == "labelmap"` [S1, unofficial].
*   `_HasWeights`: Integer flag (`0` or `1`). `1` if a weight texture is provided [S1, unofficial].
*   `time`: Float, default `1.0` [S2, unofficial].

### Edge Cases and Pitfalls
*   **Missing Intensity Range**: If `d.meta.intensity_range` is missing for a continuous TF, the range defaults to `[0.0, 1.0]`, which may cause incorrect color mapping if the data is not normalized [S1, unofficial].
*   **Null Textures**: If no label or weight texture is provided, a black 3D texture is used. The shader must handle this gracefully (e.g., treating black as zero intensity/weight) [S1, unofficial].
*   **Affine Inverse**: Computing the inverse affine matrix is critical for correct coordinate transformation. Singular matrices will cause errors [S1, unofficial].
*   **Texture Wrapping**: Incorrect wrapping modes (e.g., `RepeatWrapping` for fire profile) may cause artifacts at texture boundaries [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: O(1) for material setup, assuming texture loading is asynchronous or cached. The actual rendering cost is determined by the ray marching scheme [G].
*   **Memory Complexity**: O(1) additional memory for the material object itself, plus the memory for the textures (which are typically loaded once and reused) [G].
*   **Strengths**: Centralizes material configuration, making it easy to switch between different rendering modes (e.g., label map vs. continuous) by changing flags [S1, S2, unofficial].
*   **Limitations**: Depends on the availability of specific textures (`nzw.png`, `firetex.png`) which may not be relevant for all volume rendering scenarios (e.g., non-fire volumes) [S2, unofficial].

## Background
The method reflects patterns from WebGL-based volume rendering libraries. `VolumetricFire` [S2, unofficial] demonstrates loading noise and fire textures with additive blending for transparent volumetric rendering. `vrdf-sdk` [S1, unofficial] describes a generic volume rendering setup involving affine matrix computation, dimension extraction, and explicit uniform bindings for textures and matrices.

## Sources
[S1] guillaume-schneider/vrdf-sdk, https://github.com/guillaume-schneider/vrdf-sdk, fetched 2026-10-26 (unofficial)
[S2] yomotsu/VolumetricFire, https://github.com/yomotsu/VolumetricFire, fetched 2026-10-27 (unofficial)
