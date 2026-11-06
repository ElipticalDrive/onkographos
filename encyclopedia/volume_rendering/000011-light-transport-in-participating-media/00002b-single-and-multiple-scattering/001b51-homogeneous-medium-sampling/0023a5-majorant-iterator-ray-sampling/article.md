# Majorant Iterator Ray Sampling

Also called: MajorantIterator

## Overview
Majorant Iterator Ray Sampling is a concrete implementation of ray sampling in participating media that handles variable density and null scattering by using stack-allocated iterators. It avoids dynamic allocation and dynamic dispatch overhead by directly calling the medium's `SampleRay` method and iterating through `RayMajorantSegment` objects [S1]. It differs from generic free-path sampling by explicitly managing segment boundaries and transmittance accumulation across heterogeneous regions defined by the medium's majorant iterator [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A `Ray` (with origin, direction, medium, and `tMax`), a random number generator `rng`, a wavelength index `lambda`, and a callback function `callback(ray, mp, sigma_maj, T_maj)` that returns `true` to continue sampling or `false` to terminate.
- **Outputs:** The accumulated majorant transmittance `T_maj` at the end of the ray (if sampling completes without termination) or the result of the callback chain.

**Data Structures:**
- `ConcreteMedium`: The specific medium type implementing the iterator.
- `MajorantIterator`: Stack-allocated iterator returned by `medium->SampleRay()`.
- `RayMajorantSegment`: Struct containing:
  - `tMin`: Start distance of the segment.
  - `tMax`: End distance of the segment.
  - `sigma_maj`: Spectrum of the majorant extinction coefficient.

**Steps:**
1. **Cast and Initialize:** Cast the ray's medium to `ConcreteMedium`. Call `medium->SampleRay(ray, tMax, lambda)` to obtain a stack-allocated `MajorantIterator` [S1].
2. **Initialize State:** Set `T_maj = SampledSpectrum(1.f)` and `done = false` [S1].
3. **Iterate Segments:** Enter a loop to fetch segments via `iter.Next()`. If `iter.Next()` returns `null` (no more segments), return `T_maj` [S1].
4. **Handle Zero Majorant:** If `seg->sigma_maj[0] == 0`:
   - Calculate `dt = seg->tMax - seg->tMin`.
   - If `dt` is infinite, clamp `dt` to `std::numeric_limits<Float>::max()` to prevent NaN in `FastExp` [S1].
   - Update `T_maj *= FastExp(-dt * seg->sigma_maj)` [S1].
   - `continue` to the next segment [S1].
5. **Sample Within Segment:** If `sigma_maj[0] > 0`:
   - Set `tMin = seg->tMin` [S1].
   - Enter inner `while(true)` loop:
     - Sample distance: `t = tMin + SampleExponential(u, seg->sigma_maj[0])` [S1].
     - Refresh random number: `u = rng.Uniform<Float>()` to maintain independence [S1].
     - **If `t < seg->tMax` (sample inside segment):**
       - Update transmittance: `T_maj *= FastExp(-(t - tMin) * seg->sigma_maj)` [S1].
       - Get medium properties: `mp = medium->SamplePoint(ray(t), lambda)` [S1].
       - Call callback: `if (!callback(ray(t), mp, seg->sigma_maj, T_maj)) { done = true; break; }` [S1].
       - Reset state for next sample: `T_maj = SampledSpectrum(1.f)` and `tMin = t` [S1].
     - **If `t >= seg->tMax` (sample past segment):**
       - Calculate `dt = seg->tMax - tMin` [S1].
       - Update transmittance: `T_maj *= FastExp(-dt * seg->sigma_maj)` [S1].
       - `break` inner loop to fetch next segment [S1].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def majorant_iterator_ray_sampling(ray, rng, lambda_idx, callback):
    # Cast medium and initialize iterator
    medium = ray.medium.cast(ConcreteMedium)
    iter = medium.sample_ray(ray, ray.t_max, lambda_idx)
    
    T_maj = SampledSpectrum(1.0)
    done = False
    
    while not done:
        seg = iter.next()
        if seg is None:
            return T_maj
            
        # Handle zero-valued majorant
        if seg.sigma_maj[0] == 0:
            dt = seg.t_max - seg.t_min
            if is_inf(dt):
                dt = float_max()
            T_maj *= fast_exp(-dt * seg.sigma_maj)
            continue
            
        t_min = seg.t_min
        while True:
            # Sample distance from t_min
            t = t_min + sample_exponential(rng.uniform(), seg.sigma_maj[0])
            # Refresh random number for next iteration
            u = rng.uniform()
            
            if t < seg.t_max:
                # Sample is within segment
                T_maj *= fast_exp(-(t - t_min) * seg.sigma_maj)
                mp = medium.sample_point(ray.at(t), lambda_idx)
                if not callback(ray.at(t), mp, seg.sigma_maj, T_maj):
                    done = True
                    break
                # Reset for next sample in same segment
                T_maj = SampledSpectrum(1.0)
                t_min = t
            else:
                # Sample is past segment end
                dt = seg.t_max - t_min
                T_maj *= fast_exp(-dt * seg.sigma_maj)
                break
                
    return T_maj
```

**Parameters:**
- `lambda`: Wavelength index. Typical values depend on the spectral resolution of the renderer.
- `tMax`: Maximum ray length.

**Edge Cases and Pitfalls:**
- **Zero Majorant:** If `sigma_maj[0]` is zero, `SampleExponential` would return infinity. The code checks for this and skips exponential sampling, updating transmittance directly [S1].
- **Infinite `tMax`:** If `seg->tMax` is infinite, `dt` becomes infinite. Multiplying zero (from zero `sigma_maj`) by infinity yields NaN. The code clamps infinite `dt` to `std::numeric_limits<Float>::max()` to prevent NaN in `FastExp` [S1].

## Operating characteristics
**Time and Memory Complexity:**
- **Memory:** Stack-allocated iterator avoids dynamic allocation overhead [S1].
- **Time:** Avoids dynamic dispatch for `Next()` and `SampleRay()` calls, allowing potential inlining [S1].

**Strengths and Limitations:**
- **Strengths:** Efficient due to stack allocation and direct method calls; handles variable density and null scattering explicitly [S1].
- **Limitations:** Requires the medium to implement `ConcreteMedium` and `MajorantIterator`; specific to the pbrt implementation structure [S1].

**When to Use:**
- Use when rendering participating media with variable density where performance is critical and the medium type is known at compile time [S1].

## Background
**Origin:**
- Introduced in "Light Transport II: Volume Rendering / The Equation of Transfer" (pbrt v4) [S1].
- Part of the pbrt implementation of volume rendering, specifically handling the iteration over majorant segments for ray sampling [S1].

**History and Development:**
- Builds on the concept of majorant sampling for homogeneous media, extending it to heterogeneous media via iterators [S1].
- Designed to optimize performance by eliminating dynamic allocation and dispatch overhead in the inner sampling loop [S1].

**Usage:**
- Used in pbrt v4 for volume rendering in participating media [S1].

## Sources
[S1] Light Transport II: Volume Rendering / The Equation of Transfer, https://pbr-book.org/4ed/Light_Transport_II_Volume_Rendering/The_Equation_of_Transfer.html, fetched 2026-10-27
