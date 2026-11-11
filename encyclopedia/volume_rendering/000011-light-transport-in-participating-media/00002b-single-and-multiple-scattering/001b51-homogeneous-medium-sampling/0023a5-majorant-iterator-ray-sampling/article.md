# Majorant Iterator Ray Sampling

Also called: MajorantIterator

## Overview
Majorant Iterator Ray Sampling is a concrete implementation of ray sampling in participating media that handles variable density and null scattering by using stack-allocated iterators [S1]. It avoids dynamic allocation and dynamic dispatch overhead by directly calling the medium's `SampleRay` method and iterating through `RayMajorantSegment` objects [S1]. It differs from generic free-path sampling by explicitly managing segment boundaries and transmittance accumulation across heterogeneous regions defined by the medium's majorant iterator [S1].

## How it works
**Inputs and Outputs:**
- **Inputs:** A `Ray` (with origin, direction, medium, and `tMax`), a random number generator `rng`, a wavelength index `lambda`, and a callback function `callback(ray, mp, sigma_maj, T_maj)` that returns `true` to continue sampling or `false` to terminate [S1].
- **Outputs:** The accumulated majorant transmittance `T_maj` at the end of the ray (if sampling completes without termination) [S1].

**Data Structures:**
- `ConcreteMedium`: The specific medium type implementing the iterator [S1].
- `MajorantIterator`: Stack-allocated iterator returned by `medium->SampleRay()` [S1].
- `RayMajorantSegment`: Struct containing `tMin` (start distance), `tMax` (end distance), and `sigma_maj` (spectrum of the majorant extinction coefficient) [S1].

**Steps:**
1. **Cast and Initialize:** Cast the ray's medium to `ConcreteMedium` and call `medium->SampleRay(ray, tMax, lambda)` to obtain a stack-allocated `MajorantIterator` [S1].
2. **Initialize State:** Set `T_maj` to a spectrum of ones and `done` to `false` [S1].
3. **Iterate Segments:** Enter a loop to fetch segments via `iter.Next()` [S1]. If `iter.Next()` returns `null` (no more segments), return `T_maj` [S1].
4. **Handle Zero Majorant:** If `seg->sigma_maj[0] == 0`:
   - Calculate `dt = seg->tMax - seg->tMin` [S1].
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
import math
import sys

# ConcreteMedium: specific medium type implementing the iterator [S1]
class ConcreteMedium:
    def cast(self): return self
    def sample_ray(self, ray, t_max, lambda_idx): return MajorantIterator()
    @property
    def sigma_maj(self): return [1.0] # Placeholder spectrum

class MajorantIterator:
    def next(self): return None # Placeholder for segment iteration

def sample_exponential(u, sigma_maj_0):
    """Sample distance from exponential distribution with rate sigma_maj[0]."""
    return -math.log(u) / sigma_maj_0

def fast_exp(x_spectrum):
    """Element-wise exponential of a spectrum (list of floats)."""
    return [math.exp(v) for v in x_spectrum]

def is_inf(x):
    """Check if a scalar is infinite."""
    return math.isinf(x)

def float_max():
    """Return the largest finite float."""
    return sys.float_info.max

def majorant_iterator_ray_sampling(ray, rng, lambda_idx, callback):
    # Cast medium and initialize iterator
    medium = ray.medium.cast(ConcreteMedium)
    iter = medium.sample_ray(ray, ray.t_max, lambda_idx)

    # T_maj is a spectrum (list of floats), initialized to 1.0
    T_maj = [1.0] * len(ray.medium.sigma_maj)
    done = False
    u = rng.uniform()  # Initialize u before first sample [G]

    while not done:
        seg = iter.next()
        if seg is None:
            return T_maj

        # Handle zero-valued majorant for the first wavelength
        if seg.sigma_maj[0] == 0:
            dt = seg.t_max - seg.t_min
            if is_inf(dt):
                dt = float_max()
            T_maj = [t * math.exp(-dt * s) for t, s in zip(T_maj, seg.sigma_maj)]
            continue

        t_min = seg.t_min
        while True:
            # Sample distance using the first wavelength's majorant
            t = t_min + sample_exponential(u, seg.sigma_maj[0])
            # Refresh random number to maintain independence [G]
            u = rng.uniform()

            if t < seg.t_max:
                # Sample is within segment
                T_maj = [t_val * math.exp(-(t - t_min) * s) for t_val, s in zip(T_maj, seg.sigma_maj)]
                mp = medium.sample_point(ray.at(t), lambda_idx)
                if not callback(ray.at(t), mp, seg.sigma_maj, T_maj):
                    done = True
                    break
                # Reset transmittance and tMin for next sample
                T_maj = [1.0] * len(T_maj)
                t_min = t
            else:
                # Sample is past segment end
                dt = seg.t_max - t_min
                T_maj = [t_val * math.exp(-dt * s) for t_val, s in zip(T_maj, seg.sigma_maj)]
                break

    return T_maj
```

**Parameters:**
- `lambda`: Wavelength index. Typical values: Not given in the sources.
- `tMax`: Maximum ray length. Typical values: Not given in the sources.

**Edge Cases and Pitfalls:**
- **Zero Majorant:** If `sigma_maj[0]` is zero, `SampleExponential` would return infinity [S1]. The code checks for this and skips exponential sampling, updating transmittance directly [S1].
- **Infinite `tMax`:** If `seg->tMax` is infinite, `dt` becomes infinite [S1]. Multiplying zero (from zero `sigma_maj`) by infinity yields NaN [S1]. The code clamps infinite `dt` to `std::numeric_limits<Float>::max()` to prevent NaN in `FastExp` [S1].

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
