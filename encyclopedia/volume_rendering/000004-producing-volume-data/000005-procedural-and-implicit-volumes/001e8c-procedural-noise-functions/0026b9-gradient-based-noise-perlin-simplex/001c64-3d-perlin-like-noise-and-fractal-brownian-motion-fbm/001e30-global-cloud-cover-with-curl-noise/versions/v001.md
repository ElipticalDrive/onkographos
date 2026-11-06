# Global Cloud Cover with Curl Noise

Also called: Curl Noise Cloud Generation, PlaceCloudLayer, Spherical Noise Cloud Pattern

## Overview
Generates global cloud patterns by applying procedural noise to volumetric air blocks within a specified height range, marking the top layer in a cloud map [S1, unofficial]. It creates 2D cloud patterns on a spherical surface by combining octaves of mixed Perlin and Worley noise, subtracting a threshold, and clamping the result [S2, unofficial]. This method differs from local cloud generation by producing a coherent, planet-wide coverage pattern suitable for large-scale atmospheric rendering [S2, unofficial].

## How it works
**Inputs:**
- World dimensions: `worldWidth`, `worldHeight` [S1, unofficial].
- Cloud layer parameters: `height` (top altitude), `thickness` [S1, unofficial].
- Noise parameters: `compression` factor [S1, unofficial], number of octaves, noise type (Perlin/Worley mix) [S2, unofficial].
- Threshold value for cloud formation [S2, unofficial].

**Outputs:**
- `cloudMap`: A 2D array of size `worldWidth x worldWidth` marking cloud presence [S1, unofficial].
- Volumetric density field for ray marching or opacity map generation [S2, unofficial].

**Data Structures:**
- `cloudMap`: 2D integer/boolean array, dimensions `worldWidth` x `worldWidth` [S1, unofficial].
- Cubemap: `6 x 512 x 512` resolution for global cloud cover storage [S2, unofficial].
- Transmittance array: `7 x 512 x 512` for deep opacity maps [S2, unofficial].

**Steps:**
1. Initialize `cloudMap` as a 2D array of size `worldWidth x worldWidth` [S1, unofficial].
2. Iterate through vertical slices of the cloud layer:
   - Loop `y` from `height` down to `height - thickness` (inclusive) [S1, unofficial].
   - Loop `x` from `1` to `worldWidth - 2` [S1, unofficial].
   - Loop `z` from `1` to `worldWidth - 2` [S1, unofficial].
3. Check if the block at `(x, y, z)` is air (`GetBlockData(x, y, z) == 0`) [S1, unofficial].
4. If air, calculate noise value:
   $$ noiseValue = noise.simplex3\left(\frac{x \cdot compression}{100}, \frac{y \cdot compression}{100}, \frac{z \cdot compression}{100}\right) $$
   [S1, unofficial].
5. For global spherical patterns, combine octaves of mixed Perlin and Worley noise [S2, unofficial].
6. Subtract a threshold from the combined noise value [S2, unofficial].
7. Clamp the resulting value to create 2D cloud patterns on the spherical surface [S2, unofficial].
8. Restrict clouds between bottom and top heights to form prism-like objects [S2, unofficial].
9. Store results in the `cloudMap` or cubemap [S1, S2, unofficial].

**Parameters:**
- `compression`: Scaling factor for noise coordinates [S1, unofficial].
- Octaves: 3 octaves of Perlin-Worley noise for horizontal 2D shapes [S2, unofficial].
- High-frequency detail: 5 octaves of 64x64x64 Worley noise [S2, unofficial].
- Vertical profile: 10 values in a texture [S2, unofficial].
- Noise size: 64x64x64 for detail layers [S2, unofficial].

**Edge Cases:**
- Clamping operation for cover sample noise was removed in later implementations to improve quality [S2, unofficial].
- Use cascaded deep opacity maps instead of a single opacity map for performance [S2, unofficial].

## Operating characteristics
- **Memory:** Uses a `6x512x512` cubemap for global cover and `7x512x512` transmittance array [S2, unofficial].
- **Performance:** Renders 3 deep opacity maps with one 512x512 offset layer per frame [S2, unofficial].
- **Strengths:** Efficient global coverage; supports high-frequency detail via Worley noise [S2, unofficial].
- **Limitations:** Requires careful tuning of threshold and compression for desired cloud density [S1, S2, unofficial].

## Background
Originated from techniques for generating global cloud cover using curl noise and mixed Perlin/Worley noise [S2, unofficial]. Used in real-time rendering for atmospheric effects, particularly in games and simulations requiring large-scale cloud systems [S2, unofficial].

## Sources
[S1] mayakeele/bad-minecraft, https://github.com/mayakeele/bad-minecraft, fetched 2026-10-27 (unofficial)
[S2] Procedural Volumetric Clouds · Jan Wedekind, https://www.wedesoft.de/software/2023/05/03/volumetric-clouds/, fetched 2026-10-28 (unofficial)
