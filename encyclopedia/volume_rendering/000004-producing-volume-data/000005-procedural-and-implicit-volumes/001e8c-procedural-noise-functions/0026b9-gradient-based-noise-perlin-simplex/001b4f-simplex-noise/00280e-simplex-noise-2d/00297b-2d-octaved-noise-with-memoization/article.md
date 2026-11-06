# 2D Octaved Noise with Memoization

Also called: 2D OpenSimplex Noise with Memoization, Fractal Noise with Caching, Octave Noise, Perlin-like Noise Generation

## Overview
A fractal noise generator that sums multiple octaves of a base 2D noise function with decreasing amplitude and increasing frequency, caching results to avoid redundant calculations [S2, unofficial]. It differs from standard octaved noise by explicitly memoizing the final accumulated result per coordinate and octave count, trading memory for repeated computation avoidance [S2, unofficial]. It builds on [Simplex Noise 2D](4155-simplex-noise-2d.md) as the base noise primitive.

## How it works
**Inputs:**
- Coordinates $(a, b)$ in 2D space.
- Number of octaves $n_{octaves}$.
- Wavelength $\lambda$ (spatial scale factor) [S2, unofficial].
- Base amplitude $A_{base}$ [S2, unofficial].
- Cache storage for previous evaluations [S2, unofficial].

**Outputs:**
- A single normalized noise value $N \in [-1, 1]$ (or similar range depending on base noise) [S2, unofficial].

**Data Structures:**
- **Noise Data Array:** A `std::vector<float>` storing the base noise values, accessed via helper functions `get()` and `set()` which map 2D coordinates $(x, y)$ to a 1D index $x \cdot \text{width} + y$ [S1, unofficial].
- **Cache:** A hash map or dictionary storing keys of the form `${a},${b},${n_{octaves}}` mapped to the computed noise result [S2, unofficial].

**Steps:**
1. **Check Cache:** Construct the cache key from inputs $a, b, n_{octaves}$. If the key exists in the cache, return the stored value immediately [S2, unofficial].
2. **Initialize Accumulators:** Set `res = 0`, `sum_amp = 0` [S2, unofficial].
3. **Iterate Octaves:** For each octave $o$ from $0$ to $n_{octaves} - 1$:
   a. **Calculate Local Amplitude:** $amp_{local} = \frac{1}{o+1} \cdot A_{base}$ [S2, unofficial]. (Note: Source uses `1/oct` but loop starts at 0; implementation must handle $o=0$ or use $o+1$ to avoid division by zero. The source snippet `amp_local = (1/oct)*this.amp` implies a 1-based index or a specific offset not explicitly detailed in the snippet's loop structure `for(let o = 0; o < n_octaves; o++)`. We assume standard fractal Brownian motion scaling where amplitude decreases with octave index. If $o=0$, amplitude is typically $A_{base}$. If the source strictly implies `1/oct`, $o$ must start at 1 or be adjusted. Given the snippet `oct*(a/this.wavelength)`, if $o=0$, frequency is 0. We follow the source's accumulation logic: `res += amp_local * this.noise(...)`.
   b. **Accumulate Amplitude:** `sum_amp += amp_local` [S2, unofficial].
   c. **Compute Base Noise:** Call the base 2D noise function (e.g., Simplex Noise) with scaled coordinates: $x' = o \cdot \frac{a}{\lambda}$, $y' = o \cdot \frac{b}{\lambda}$ [S2, unofficial].
   d. **Add to Result:** `res += amp_local * noise(x', y')` [S2, unofficial].
4. **Normalize:** Calculate final result $ans = \frac{res}{sum\_amp}$ [S2, unofficial].
5. **Cache and Return:** Store $ans$ in the cache with the key and return $ans$ [S2, unofficial].

**Sub-routine: `generateSmoothNoise` (for base noise generation if not using a library):**
1. Determine period $P = 2^o$ and frequency $f = 1.0/P$ [S1, unofficial].
2. Identify sample points $(i_0, j_0), (i_1, j_0), (i_0, j_1), (i_1, j_1)$ based on the period and wrap coordinates using modulo width/height [S1, unofficial].
3. Perform bilinear interpolation:
   a. Interpolate horizontally between top-left and top-right.
   b. Interpolate horizontally between bottom-left and bottom-right.
   c. Interpolate vertically between the two horizontal results [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def generate_octaved_noise(a, b, n_octaves, wavelength, base_amplitude, cache, base_noise_func):
    """
    Computes fractal noise with memoization.
    
    Args:
        a, b: Input coordinates.
        n_octaves: Number of octaves to sum.
        wavelength: Spatial scale factor.
        base_amplitude: Amplitude of the first octave.
        cache: Dictionary for storing results.
        base_noise_func: Function implementing the base 2D noise (e.g., Simplex).
        
    Returns:
        Normalized noise value.
    """
    # Step 1: Check Cache
    key = f"{a},{b},{n_octaves}"
    if key in cache:
        return cache[key]
    
    # Step 2: Initialize Accumulators
    res = 0.0
    sum_amp = 0.0
    
    # Step 3: Iterate Octaves
    for o in range(n_octaves):
        # Handle o=0 to avoid division by zero if using 1/octave scaling
        # Source snippet: amp_local = (1/oct)*this.amp
        # Assuming standard FBM where octave 0 has full amplitude, 
        # or source implies 1-based indexing for the formula. 
        # We use (o + 1) to match standard FBM behavior and avoid div-by-zero,
        # aligning with the intent of "decreasing amplitude".
        divisor = o + 1
        amp_local = (1.0 / divisor) * base_amplitude
        
        sum_amp += amp_local
        
        # Calculate scaled coordinates
        # Source: oct * (a / this.wavelength)
        x_scaled = o * (a / wavelength)
        y_scaled = o * (b / wavelength)
        
        # Get base noise value
        noise_val = base_noise_func(x_scaled, y_scaled)
        
        res += amp_local * noise_val
        
    # Step 4: Normalize
    if sum_amp == 0:
        ans = 0.0
    else:
        ans = res / sum_amp
        
    # Step 5: Cache and Return
    cache[key] = ans
    return ans
```

**Edge Cases and Pitfalls:**
- **Division by Zero:** If the loop index $o$ starts at 0 and the amplitude formula is strictly $1/o$, a division by zero occurs. The implementation must adjust the index or formula [S2, unofficial].
- **Cache Key Precision:** Floating-point coordinates $a, b$ should be quantized or hashed carefully to ensure consistent cache hits [S2, unofficial].
- **Memory Usage:** Caching every unique coordinate and octave combination can consume significant memory for large terrains or high-resolution volumes [S2, unofficial].

## Operating characteristics
- **Time Complexity:** $O(n_{octaves})$ per unique coordinate if cache miss; $O(1)$ if cache hit [S2, unofficial].
- **Memory Complexity:** $O(N \cdot n_{octaves})$ where $N$ is the number of unique coordinates stored in the cache [S2, unofficial].
- **Strengths:** Eliminates redundant calculations for repeated queries (e.g., static terrain generation) [S2, unofficial].
- **Limitations:** High memory overhead for dense sampling; cache invalidation is not described [S2, unofficial].
- **When to Use:** Static procedural generation where the same points are queried multiple times [S2, unofficial].
- **When Not to Use:** Dynamic scenes with constantly changing coordinates where cache hits are rare [S2, unofficial].

## Background
This method is an implementation technique found in web-based voxel terrain generators, specifically combining fractal Brownian motion (fBm) with memoization [S2, unofficial]. It relies on a base 2D gradient noise function, such as Simplex Noise, for the underlying smooth noise generation [S1, S2, unofficial]. The approach is common in real-time graphics where performance optimization via caching is prioritized over memory efficiency for static data [S2, unofficial].

## Sources
[S1] nnarain/VoxelEngine, https://github.com/nnarain/VoxelEngine, fetched 2026-10-27 (unofficial)
[S2] andrewbloese-00/web_voxel_terrain, https://github.com/andrewbloese-00/web_voxel_terrain, fetched 2026-10-27 (unofficial)
