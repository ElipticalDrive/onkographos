# Blue Noise Jitter Generator
Also called: Void-and-Cluster Blue Noise

## Overview
The Blue Noise Jitter Generator produces temporally coherent, spatially blue-noise-distributed offsets for ray origins or sampling positions. It combines a low-discrepancy spatial lookup table with a golden-ratio-based temporal shift and a Halton sequence for subpixel jitter, ensuring that jitter values change smoothly across frames while maintaining blue-noise spectral properties. It differs from [Hammersley Sampling](8569-hammersley-sampling.md) by explicitly decoupling spatial structure (via a precomputed table) from temporal evolution (via modular golden-ratio shifts), which reduces temporal aliasing artifacts in animated scenes.

## How it works
**Inputs:**
- Pixel coordinates $(x, y)$ (floats, typically subpixel-accurate).
- Frame index $f$ (integer, $\ge 0$).
- Table size $N$ (default 64).

**Outputs:**
- A single jitter value $j \in [0, 1)$.

**Data Structures:**
- `table`: A 1D array of size $N \times N$ (default $64 \times 64 = 4096$).
- Each entry `table[i]` stores a float in $[0, 1)$.

**Initialization:**
The table is initialized using a low-discrepancy sequence based on the golden ratio.
$$ \phi = 0.618033988749895 $$
$$ \text{table}[i] = (i \times \phi) \pmod{1.0}, \quad \text{for } i = 0, \dots, N^2-1 $$
[G] The modulo operation ensures values remain in $[0, 1)$. In floating-point arithmetic, this is typically implemented as `i * phi - floor(i * phi)`.

**Step 1: Spatial Base Offset**
Determine the spatial index within the table based on pixel coordinates.
$$ p_x = |\lfloor x \rfloor| \pmod N $$
$$ p_y = |\lfloor y \rfloor| \rfloor| \pmod N $$
$$ \text{baseOffset} = \text{table}[p_y \times N + p_x] $$
[G] Absolute value is used to handle negative pixel coordinates gracefully, mapping them to valid table indices.

**Step 2: Temporal Shift**
Compute a temporal shift that evolves slowly over frames to ensure temporal coherence.
$$ \text{temporalShift} = ((f \pmod{16}) \times \phi) \pmod{1.0} $$
The modulo 16 ensures the temporal sequence repeats every 16 frames, providing a balance between coherence and variation.

**Step 3: Subpixel Jitter (Halton Sequence)**
Generate subpixel jitter using the Halton sequence with bases 2 and 3. This adds high-frequency variation independent of the frame index.
$$ h_2 = \text{Halton}(f + 1, 2) - 0.5 $$
$$ h_3 = \text{Halton}(f + 1, 3) - 0.5 $$
The Halton sequence is defined as:
$$ \text{Halton}(index, base) = \sum_{k=1}^{\infty} \frac{d_k}{base^k} $$
where $d_k$ are the digits of $index$ in base $base$.

**Step 4: Final Jitter**
Combine the components. The exact combination formula is not explicitly detailed in the sources for the final output, but the spatial and temporal components are combined as:
$$ j = (\text{baseOffset} + \text{temporalShift}) \pmod{1.0} $$
[G] The subpixel jitter $h_2, h_3$ is typically used to offset the ray origin in screen space or to perturb the sampling position, rather than being added directly to the scalar $j$. If a single scalar jitter is required, it may be derived from $h_2$ or $h_3$ alone, or combined via a hash. The sources do not specify how $h_2$ and $h_3$ are combined into the final output $j$.

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def blue_noise_jitter(x: float, y: float, frame_index: int, table_size: int = 64) -> float:
    phi = 0.618033988749895
    
    # Initialize table if not already done
    if not table_initialized:
        table = [0.0] * (table_size * table_size)
        for i in range(table_size * table_size):
            table[i] = (i * phi) % 1.0
        table_initialized = True
    
    # Spatial base offset
    px = abs(int(x)) % table_size
    py = abs(int(y)) % table_size
    base_offset = table[py * table_size + px]
    
    # Temporal shift
    temporal_shift = ((frame_index % 16) * phi) % 1.0
    
    # Final jitter
    jitter = (base_offset + temporal_shift) % 1.0
    
    return jitter

def halton(index: int, base: int) -> float:
    result = 0.0
    f = 1.0 / base
    i = index
    while i > 0:
        result += f * (i % base)
        i = i // base
        f = f / base
    return result
```

## Subtopics

### Blue Noise Sampling
A technique for sampling pre-generated 3D blue noise textures using integer indexing and normalization. It is used when high-quality blue noise is required for complex sampling patterns (e.g., importance sampling) rather than simple jitter.
- **Definition:** Sample a 3D blue noise texture at a given position and channel.
- **Steps:**
  1. Scale the input position by the resolution: $pos \times res$.
  2. Cast components to unsigned integers: $index.x = \lfloor pos.x \rfloor$, $index.y = \lfloor pos.y \rfloor$, $index.z = \lfloor pos.z \rfloor$.
  3. Set $index.w$ to the channel number.
  4. Retrieve the value from the texture cache and normalize: $value / 256.0$.
- **Parameters:**
  - $res$: Resolution of the blue noise texture (16, 32, or 64).
  - $channel$: Channel index (0-3 for RGBA).
- **Pitfalls:** The resolution must match the loaded texture exactly. Mismatched resolutions cause out-of-bounds access or incorrect sampling.

## Operating characteristics
- **Time Complexity:** $O(1)$ for table lookup and arithmetic operations. Halton sequence computation is $O(\log_{base}(index))$, which is negligible for typical frame counts.
- **Memory Complexity:** $O(N^2)$ for the table, where $N$ is the table size (default 4096 floats for $N=64$).
- **Performance:** Very fast, suitable for real-time rendering. The use of integer indexing and simple arithmetic avoids expensive transcendental functions.
- **Strengths:** Temporal coherence reduces flickering. Blue noise distribution minimizes low-frequency aliasing.
- **Limitations:** The temporal sequence repeats every 16 frames, which may cause visible patterns in long animations. The spatial table is fixed, so it may not adapt to varying scene complexity.
- **When to use:** Real-time ray tracing, animated volume rendering, scenes requiring temporal stability.
- **When not to use:** Offline rendering where infinite temporal coherence is not required, or when higher-quality blue noise textures are available and can be sampled directly.

## Background
The method originates from the implementation of volumetric cloud rendering in real-time graphics, specifically in the `volumetric-clouds-wgsl` repository [S1, unofficial]. It leverages the golden ratio for low-discrepancy sequences, a technique common in quasi-Monte Carlo methods. The use of Halton sequences for subpixel jitter is a standard approach in ray tracing to ensure high-frequency variation. The blue noise sampling variant is described in the `volume-noise-generator` repository [S2, unofficial], which focuses on efficient sampling of pre-generated blue noise textures for volume rendering.

## Sources
[S1] nff747/volumetric-clouds-wgsl, https://github.com/nff747/volumetric-clouds-wgsl, fetched 2026-10-27 (unofficial)
[S2] sixthsurge/volume-noise-generator, https://github.com/sixthsurge/volume-noise-generator, fetched 2026-10-27 (unofficial)
