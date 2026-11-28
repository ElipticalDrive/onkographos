# Texture-Based Splat Storage Preprocessing


## Overview
This method preprocesses parsed 3D Gaussian splat data into eight separate RGBA32F textures, one for each attribute channel (Position, Scale, Rotation, Color, and three Spherical Harmonics levels), to enable efficient on-demand loading by the Unreal Engine Niagara rendering system [S1, unofficial]. It sets itself apart from sibling methods by focusing on the creation of uncompressed, high-precision floating-point texture assets rather than byte-level compression or Morton-ordering [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A list of parsed Gaussian splat attributes (Position, Scale, Rotation, Color, SH coefficients) typically derived from PLY or SPZ files [S1, unofficial].
- Output: Eight `UTexture2D` assets, each containing an array of `FLinearColor` values representing the respective attribute channel [S1, unofficial].

**Data Structures:**
The method organizes data using the `FGaussianSplattingTextureData` structure, which contains eight fields, each being a dynamic array of `FLinearColor` (4-component float vector):
1. `PositionTextureData`: Stores 3D coordinates.
2. `ScaleTextureData`: Stores scale factors.
3. `RotationTextureData`: Stores rotation quaternions.
4. `ColorTextureData`: Stores base color.
5. `HarmonicsL1TextureData`: Stores Level 1 SH coefficients.
6. `HarmonicsL2TextureData`: Stores Level 2 SH coefficients.
7. `HarmonicsL31TextureData`: Stores Level 3 SH coefficients (part 1).
8. `HarmonicsL32TextureData`: Stores Level 3 SH coefficients (part 2) [S1, unofficial].

**Steps:**
1. **Parse and Separate:** Extract the attribute values for each of the $N$ splats and separate them into eight distinct lists corresponding to the fields in `FGaussianSplattingTextureData` [S1, unofficial].
2. **Texture Creation:** For each attribute channel, create a new `UTexture2D` asset [S1, unofficial].
3. **Texture Configuration:** Set the following properties on the created texture: `SRGB` to `false` (linear space), `MipGenSettings` to `TMGS_NoMipmaps`, `Compression` to `CompressionNone`, and `Filter` to `TF_Nearest` (point sampling) [S1, unofficial].
4. **Format Initialization:** Initialize the texture source format to `ETextureSourceFormat::TSF_RGBA32F` (32-bit floating point RGBA) [S1, unofficial].
5. **Data Copy:** Lock mip level 0 of the texture, copy the `FLinearColor` array for the specific attribute into the locked memory using a memory copy operation, and unlock the mip level [S1, unofficial].
6. **Resource Update:** Update the texture resources to make them available to the rendering pipeline [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def preprocess_splats_to_textures(splats: list, num_splats: int) -> dict:
    """
    Preprocesses Gaussian splat data into 8 RGBA32F textures.
    Args:
        splats: List of Splat objects with attributes:
                pos (Vec3), scale (Vec3), rot (Quat), color (Vec3),
                sh_l1 (Vec3), sh_l2 (Vec3), sh_l3_1 (Vec3), sh_l3_2 (Vec3).
        num_splats: Total number of splats.
    Returns:
        Dictionary mapping attribute names to texture data structures.
    """
    textures = {}
    channels = [
        "Position", "Scale", "Rotation", "Color",
        "HarmonicsL1", "HarmonicsL2", "HarmonicsL31", "HarmonicsL32"
    ]
    
    for channel in channels:
        # 1. Extract data for this channel
        pixel_data = []
        for i in range(num_splats):
            s = splats[i]
            val = getattr(s, channel)
            # FLinearColor is a 4-component float vector.
            # For 3-component data, pad with 1.0 as per engine convention [G].
            if channel == "Rotation":
                pixel_data.append((val.x, val.y, val.z, val.w))
            else:
                pixel_data.append((val.x, val.y, val.z, 1.0))
                
        # 2. Create Texture2D & 3. Configure Texture & 4. Format Initialization
        tex_data = {
            "srgb": False,
            "mip_gen_settings": "NO_MIPMAPS",
            "compression": "NONE",
            "filter": "NEAREST",
            "source_format": "RGBA32F",
            "mip0_data": pixel_data
        }
        
        # 5. Data Copy (simulated lock/memcpy/unlock)
        # In engine: tex.lock_mip(0); memcpy(ptr, pixel_data); tex.unlock_mip(0)
        
        # 6. Resource Update (simulated)
        # In engine: tex.update_resources()
        
        textures[channel] = tex_data
        
    return textures

class FLinearColor:
    def __init__(self, r, g, b, a):
        self.r = r
        self.g = g
        self.b = b
        self.a = a

class Splat:
    def __init__(self, pos, scale, rot, color, sh_l1, sh_l2, sh_l3_1, sh_l3_2):
        self.pos = pos
        self.scale = scale
        self.rot = rot
        self.color = color
        self.sh_l1 = sh_l1
        self.sh_l2 = sh_l2
        self.sh_l3_1 = sh_l3_1
        self.sh_l3_2 = sh_l3_2

class Vec3:
    def __init__(self, x, y, z):
        self.x = x
        self.y = y
        self.z = z

class Quat:
    def __init__(self, x, y, z, w):
        self.x = x
        self.y = y
        self.z = z
        self.w = w
```

**Parameters:**
- Texture Format: `TSF_RGBA32F` (32-bit floating point per channel) [S1, unofficial].
- Texture Size: $N \times 1$ or $1 \times N$ depending on engine implementation, where $N$ is the number of splats [S1, unofficial].
- Filtering: `TF_Nearest` (Point sampling) [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Precision:** Using `RGBA32F` ensures high precision for SH coefficients and rotations, avoiding quantization errors present in lower-precision formats [S1, unofficial].
- **Memory Usage:** Storing 8 full-precision textures for millions of splats consumes significant VRAM. For example, 2 million splats require $2 \times 10^6 \times 8 \times 16$ bytes $\approx 256$ MB per texture, totaling ~2 GB for all 8 textures [S1, unofficial].
- **Transformations:** Moving, rotating, or scaling the splat actor in the world is not fully supported and may break rendering [S1, unofficial].
- **SH Support:** Support for Spherical Harmonics may be limited or incomplete in early versions [S1, unofficial].

## Subtopics

### [Chunked Bounding Box Computation and Normalization](17031-chunked-bounding-box-computation-and-normalization.md)
Preprocesses splats by dividing them into fixed-size chunks, computing per-chunk bounding boxes, and normalizing data to $[0, 1]$ for efficient storage in 8 separate RGBA32F textures.

### [Gaussian Splat Data Packing and Compression](17062-gaussian-splat-data-packing-and-compression.md)
Converts raw splat data into a compressed, GPU-ready byte buffer layout using Morton ordering and aggressive quantization (Norm11/16/8) for real-time rendering.

## Operating characteristics
- **Time Complexity:** $O(N)$, where $N$ is the number of splats, as each attribute is processed once per splat [S1, unofficial].
- **Memory Complexity:** $O(N)$, with a constant factor of 8 for the 8 separate textures. Each texture stores $N$ `FLinearColor` (16 bytes) values [S1, unofficial].
- **Measured Performance:** Designed for real-time applications, including VR and interactive walkthroughs, supporting models with up to 2 million splats [S1, unofficial].
- **Strengths:** High precision due to 32-bit floating point format; simple integration with Unreal Engine's Niagara system; on-demand loading reduces initial load times [S1, unofficial].
- **Limitations:** High memory footprint compared to compressed formats; lack of support for dynamic transformations of the splat actor [S1, unofficial].
- **When to Use:** When high precision is required for SH coefficients and rotations, and the target platform has sufficient VRAM. Ideal for static scenes or pre-baked splat data in Unreal Engine [S1, unofficial].
- **When Not to Use:** When memory bandwidth or VRAM is constrained, or when dynamic transformations of the splat cloud are required [S1, unofficial].

## Background
This method is implemented in the `unreal-splat` plugin by JI20, which integrates 3D Gaussian Splatting into Unreal Engine's Niagara VFX pipeline [S1, unofficial]. It was developed to address the need for efficient, high-performance rendering of large-scale Gaussian splat models within the Unreal Engine ecosystem, leveraging the engine's texture system for data storage and retrieval [S1, unofficial].

## Sources
[S1] JI20/unreal-splat, https://github.com/JI20/unreal-splat, fetched 2026-10-27 (unofficial)
