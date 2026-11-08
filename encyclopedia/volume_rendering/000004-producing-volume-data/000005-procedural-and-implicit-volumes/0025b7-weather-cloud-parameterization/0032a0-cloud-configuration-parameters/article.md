# Cloud Configuration Parameters

Also called: CloudConfig

## Overview
Cloud Configuration Parameters define the physical and visual properties for volumetric cloud rendering, controlling coverage, density, type, geometry, lighting, and noise interaction [S1, S2, unofficial]. Unlike [Procedural Volume Cloud Modeling](2256-procedural-volume-cloud-modeling.md) which focuses on the noise generation algorithm, or [Volumetric Cloud Raymarching](1248-volumetric-cloud-raymarching.md) which handles the integration step, this method provides the structured data interface that drives those processes [S1, S2, unofficial]. It serves as the bridge between high-level scene design (e.g., "stratocumulus at 2000m") and the low-level volumetric representation [S1, unofficial].

## How it works
The method consists of two distinct data structures found in the source literature: the high-level `CloudConfig` for runtime rendering parameters and the `VolumeParameters` for procedural generation validation.

### Inputs and Outputs
*   **Input**: Scene requirements, cloud type selection, resolution constraints, and noise strength values.
*   **Output**: A validated configuration object used to initialize cloud shaders, generate noise masks, or bake volume textures.

### Data Structures

#### 1. CloudConfig (Runtime Parameters)
Defined in [S1, unofficial], this structure maps directly to RenderEngine settings. All fields are 32-bit floats unless noted.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```rust
struct CloudConfig {
    // Cloud Type: 0.0 = stratus, 0.5 = stratocumulus, 1.0 = cumulus
    cloud_type: f32,
    
    // Global density multiplier
    coverage_multiplier: f32,
    
    // Wind vector [x, y, z]
    wind_direction: [f32; 3],
    
    // Wind speed scalar
    wind_speed: f32,
    
    // Albedo color [r, g, b]
    cloud_color: [f32; 3],
    
    // Sphere geometry bounds
    inner_sphere_radius: f32,
    outer_sphere_radius: f32,
    sphere_y_offset: f32,
    
    // Rendering limits
    max_render_distance: f32,
    cloud_top_offset: f32,
    
    // Noise scaling
    weather_texture_scale: f32,
    base_noise_scale: f32,
    high_frequency_noise_scale: f32,
    high_frequency_noise_uv_scale: f32,
    high_frequency_noise_h_scale: f32,
    
    // Quality preset
    quality: CloudQuality, // Enum: Low, Medium, High
}
```

#### 2. VolumeParameters (Generation Validation)
Defined in [S2, unofficial], this is a frozen dataclass used to validate inputs before generating volume masks.

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
@dataclass(frozen=True)
class VolumeParameters:
    cloud_type_id: int = 9
    resolution: int = 128
    seed: int = 42
    shape_strength: float = 1.0
    detail_strength: float = 0.55
    disturbance_strength: float = 0.65
    cartoon_style: bool = False
    allow_boundary_overlap: bool | None = None
```

### Validation Steps
When initializing `VolumeParameters`, the following checks must be performed [S2, unofficial]:

1.  **Type Existence**: Verify `cloud_type_id` exists in the `CLOUD_TYPE_BY_ID` catalog.
2.  **Resolution Check**: `resolution` must be exactly 64, 128, or 256.
3.  **Strength Bounds**: `shape_strength`, `detail_strength`, and `disturbance_strength` must be within the range $[0.0, 2.0]$.
4.  **Boundary Overlap**: If `allow_boundary_overlap` is `None`, use the default value defined by the specific cloud type; otherwise, use the user-provided boolean value.

### Edge Cases and Pitfalls
*   **Rect Info Validation**: When using region parameter maps, the following constraints apply to the occupied regions [S2, unofficial]:
    *   The "clear sky" area (channel 2) must be entirely 0.
    *   The aspect ratio $B/A$ (width/height, channels 3/0 and 4/1) must be $> 0$.
    *   $U_{start} + width \le 1.0001$.
    *   $V_{start} + height \le 1.0001$.
*   **Height Multiplier**: The volume box height is determined by a multiplier relative to the weather height layer. The standard multiplier is $4.0$ [S2, unofficial].
*   **Precision**: Floating point comparisons for bounds should allow for small epsilon errors (e.g., $1.0001$) to avoid false negatives due to precision loss [S2, unofficial].

## Subtopics

### Volume Parameters Validation
A strict validation subset of Cloud Configuration Parameters focused on ensuring procedural noise inputs are within valid ranges before texture generation. It enforces resolution constraints (64/128/256) and strength bounds (0.0–2.0) to prevent artifacts in the generated volume masks [S2, unofficial].

## Operating characteristics
*   **Time Complexity**: Validation is $O(1)$ relative to resolution, as it checks scalar bounds and enum existence. Texture generation validation is $O(N)$ where $N$ is the number of pixels in the rect info map [S2, unofficial].
*   **Memory Complexity**: $O(1)$ for the config structs themselves. The associated volume textures scale with $Resolution^2$ or $Resolution^3$ depending on implementation [S1, S2, unofficial].
*   **Performance**: High-quality settings (`CloudQuality::High`) increase shader complexity in the raymarching pass but do not change the size of the configuration struct [S1, unofficial].
*   **Strengths**: Decouples physical parameters from noise algorithms, allowing easy swapping of cloud types without changing the renderer core [S1, unofficial].
*   **Limitations**: Rigid resolution constraints (64/128/256) may limit flexibility for very high-res or very low-res targets [S2, unofficial].

## Background
The `CloudConfig` structure originates from the Aether Engine's attempt to 1:1 replicate the volumetric cloud system from the RenderEngine project, specifically targeting Rust/wgpu implementation [S1, unofficial]. The `VolumeParameters` validation logic comes from the Noise-Cloud project, which focuses on deterministic region mask generation for volumetric clouds [S2, unofficial].

## Sources
[S1] [volumetric-clouds-v2][RenderEngine] 1:1 复刻 NadirRoGue/RenderEngine 体积云, https://github.com/ruochenhua/AetherEngine/issues/139, fetched 2026-10-28 (unofficial)
[S2] mingtu149411/Noise-Cloud, https://github.com/mingtu149411/Noise-Cloud, fetched 2026-10-27 (unofficial)
