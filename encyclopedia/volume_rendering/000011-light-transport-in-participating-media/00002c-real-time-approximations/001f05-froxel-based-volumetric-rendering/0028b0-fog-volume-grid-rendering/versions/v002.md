# Fog Volume Grid Rendering

Also called: FogMaterial volumetric fog integration, FogVolume Volumetric Rendering

## Overview
Fog Volume Grid Rendering integrates `FogVolume` and `FogMaterial` nodes into a froxel-based volumetric fog system by adding their density, albedo, and emission to the froxels that the volume's shape covers [S1, unofficial]. It solves the problem of missing local fog contributions and negative density carving in real-time approximations, matching Godot 4.6.3's behavior [S1, unofficial]. Unlike sibling methods that focus on temporal stability or depth distribution, this method handles the geometric mapping of arbitrary volume shapes and texture-based density modulation [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Inputs:** `FogVolume` nodes (with shape, transform, and `FogMaterial`), `FogMaterial` properties (density, albedo, emission, `density_texture`), froxel grid configuration, and noise/texture resources (`NoiseTexture3D`, `ImageTexture3D`, imported 3D textures) [S1, unofficial].
*   **Outputs:** Updated froxel data containing accumulated density, albedo, and emission values; a stacked 3D density atlas texture [S1, unofficial].

**Data Structures:**
1.  **Froxel Range:** Determined by intersecting the volume's bounding shape with the froxel grid [S1, unofficial].
2.  **Row Texture:** Each visible, unculled volume is packed into a row texture (`fogVolumeRows.ts`) [S1, unofficial].
3.  **Density Atlas:** A single 3D texture (`fogDensityAtlas.ts`) where every density texture is stacked in depth [S1, unofficial]. This is necessary because WebGL2 shaders cannot index a sampler array in a loop [S1, unofficial]. The atlas holds floats once a density texture does [S1, unofficial].
4.  **Image Decoding:** Supports uncompressed formats: 8-bit, 16-bit, half, float, RGBA4444, RGB565, and RGBE9995 [S1, unofficial]. Decoding follows `Image::_get_color_at_ofs` [S1, unofficial].

**Steps:**
1.  **Froxel Range Calculation:** For each `FogVolume`, calculate the set of froxels it covers using the ported range calculation logic [S1, unofficial].
2.  **Volume Packing:** Pack each visible, unculled volume into a row texture [S1, unofficial].
3.  **Density Texture Preparation:**
    *   If the density is defined by a `NoiseTexture3D`, build the density texture as a worker job [S1, unofficial]. Ensure texels and seamless blend match Godot byte-for-byte [S1, unofficial].
    *   If the density is an `ImageTexture3D` (inline or saved as `.tres`) or an imported 3D texture, decode the pixels [S1, unofficial]. For imported 3D textures, cut them into slices matching Godot's layered importer (`resource_importer_layered_texture.cpp:370-386`) [S1, unofficial].
    *   Decode `Image` sub-resources in every uncompressed format (8-bit, 16-bit, half, float, RGBA4444, RGB565, RGBE9995) using the unpacking logic from `Image::_get_color_at_ofs` [S1, unofficial].
    *   Float texels upload raw as half-float because Godot provides no sRGB view for float formats and WebGL2 always filters half-floats [S1, unofficial]. An atlas crop of a half-float sheet remains half-float; a canvas read converts it to bytes [S1, unofficial].
4.  **Density Atlas Stacking:** Stack every prepared density texture in depth into the single 3D `fogDensityAtlas` [S1, unofficial].
5.  **Light Pass Integration:** In the light pass shader (`fogVolumeShader.ts`), add the volumes' fixed-point sums to the froxels [S1, unofficial]. Handle Godot's packed-color overflow [S1, unofficial].
6.  **Shape and Material Application:** Apply the volume's shape (box, ellipsoid, cone, cylinder, world) to determine which froxels are affected [S1, unofficial]. Apply height falloff, edge fade, and negative density [S1, unofficial]. If no `FogMaterial` is present, use the default fog shader [S1, unofficial].
7.  **Resource Slot Resolution:** Use `useTextResourceSlot` to read a slot naming a `.tres` file as that file's `[resource]` in the file's own tables [S1, unofficial]. This allows `AtlasTexture` to use the inline route [S1, unofficial].

**Formulas:**
*   **Extinction Coefficient:** $\sigma_t = \sigma_s + \sigma_a$, where $\sigma_s$ is scattering and $\sigma_a$ is absorption [S2, unofficial].
*   **Henyey-Greenstein Phase Function:** $p(\theta) = \frac{1}{4\pi} \frac{1 - g^2}{(1 + g^2 - 2g \cos\theta)^{3/2}}$, where $g \in (-1, 1)$ is the asymmetry parameter [S2, unofficial].

**Parameters:**
*   `density_texture`: A `FogMaterial` property that shapes density [S1, unofficial]. It can be a `NoiseTexture3D`, an `ImageTexture3D` (inline or saved as `.tres`), or an imported 3D texture [S1, unofficial].

**Edge Cases and Pitfalls:**
*   Float texels upload raw as half-float [S1, unofficial]. An atlas crop of a half-float sheet stays half-float; a canvas read converts it to bytes [S1, unofficial].
*   A fog `ShaderMaterial` runs no fog shader [S1, unofficial]. This requires a shading-language runtime (Issue #734) [S1, unofficial].
*   An `Image` in a block-compressed format draws nothing [S1, unofficial]. An `Image` in an integer format draws nothing because a float sampler cannot read it (Issue #736) [S1, unofficial].
*   A `NoiseTexture3D` shows even density until its build lands [S1, unofficial].
*   Past 2048 stacked density slices, further textures leave the density even [S1, unofficial].
*   A 3D colour slot still decodes every texture as sRGB [S1, unofficial]. Godot reads a format without an sRGB view raw, so a Lum8 or grey-noise albedo draws too dark (Issue #737) [S1, unofficial].

## Subtopics

### Density and Phase Function Evaluation
This variant executes a compute shader pass (`glDispatchCompute`) to evaluate fog density and material properties for every cell in the 3D texture [S2, unofficial]. For dynamic fog, it samples an auxiliary 3D noise texture (e.g., Perlin-Worley noise) using the reconstructed world-space position of the froxel [S2, unofficial]. It calculates the extinction coefficient $\sigma_t = \sigma_s + \sigma_a$ (where $\sigma_s$ is scattering and $\sigma_a$ is absorption) and uses the Henyey-Greenstein phase function $p(\theta) = \frac{1}{4\pi} \frac{1 - g^2}{(1 + g^2 - 2g \cos\theta)^{3/2}}$ (where $g \in (-1, 1)$ is the asymmetry parameter) to model angular scattering [S2, unofficial]. This method is chosen when dynamic, patchy fog with physically-based scattering is required, as opposed to the static density accumulation of the primary method [S2, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   Time complexity depends on the number of froxels covered by volumes and the cost of building noise textures [S1, unofficial]. NoiseTexture3D building is done as a worker job [S1, unofficial].
*   Memory usage requires a stacked 3D density atlas and scales with the number of density textures, up to a limit of 2048 stacked slices [S1, unofficial].

**Measured Performance:**
*   Golden test `fog-volume` (3x3x3 box at 0, 1, -5): Mean error against Godot 4.6.3 drops from 3.476/255 (no volume) to 0.205/255 [S1, unofficial].
*   Value check mean error over pixels the volume changes in Godot: box 13.05→0.40, ellipsoid 10.66→0.22, cone 8.90→0.40, cylinder 15.67→0.41, world shape 14.07→0.25, negative density 3.65→0.26, height falloff 2 9.58→0.24, edge fade 0 14.21→0.40, edge fade 4 2.55→0.08, no material 13.21→0.35, albedo/emission 32.91→1.13, density 8 23.92→0.45, rotated/scaled 13.73→0.30, spot/sun lights 24.70→0.60 [S1, unofficial].
*   Value check mean error over whole frame: imported 3D texture 3.554→0.209, ImageTexture3D .tres Lum8 4.459→0.206, NoiseTexture3D .tres 5.418→0.196, ImageTexture3D .tres RHalf 2.227→0.215, RFloat 2.229→0.216, R16 1.153→0.184, RGBE9995 2.229→0.215, NoiseTexture3D warp simplex 0.714→0.194, reduced simplex progressive 0.683→0.199, basic grid independent 0.496→0.196 [S1, unofficial].

**Strengths and Limitations:**
*   Strengths: Matches Godot 4.6.3 behavior byte-for-byte for supported formats; supports arbitrary volume shapes; handles negative density and height falloff [S1, unofficial].
*   Limitations: WebGL2 sampler array indexing limitation requires texture stacking; 2048 slice limit; `ShaderMaterial` support requires external runtime; block-compressed and integer formats do not draw; sRGB decoding issues for non-sRGB formats [S1, unofficial].

**When to Use:**
*   Use when implementing static or pre-baked volumetric fog volumes in a WebGL2 environment that needs to match Godot 4.6.3's visual output [S1, unofficial].
*   Do not use when dynamic, physically-based scattering with phase functions is required (use Density and Phase Function Evaluation instead) or when block-compressed textures are necessary [S1, unofficial][S2, unofficial].

## Background
This method implements fog volume rendering for a WebGL2-based renderer, matching Godot 4.6.3's behavior [S1, unofficial]. It addresses specific issues where local fog, negative density carving, and texture-based density were missing [S1, unofficial]. The implementation closely follows Godot's internal C++ logic (`fog.cpp`, `fog_material.cpp`, `image.cpp`, etc.) to ensure visual parity [S1, unofficial].

## Sources
[S1] feat(renderer): draw FogVolume and FogMaterial into the volumetric fog, https://github.com/Vortiago/textscene-inspector/pull/732, fetched 2026-10-26 (unofficial)
[S2] Volumetric Fog Using 3D Textures in OpenGL - opengl, https://salivity.github.io/opengl/article/volumetric-fog-using-3-d-textures-in-open-gl, fetched 2026-10-27 (unofficial)
