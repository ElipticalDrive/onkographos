# Oscillating Swept Volume Display

Also called: Oscivox, Undulating Volumetric Display

## Overview
The Oscillating Swept Volume Display (OSVD) is a volumetric display method that replaces linear or continuous rotation with an undulating (back-and-forth) motion to reduce mechanical jerk at the sweep endpoints [S1, unofficial]. It builds on [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md) by maintaining synchronous projection but altering the kinematic profile to a sinusoidal oscillation, allowing for smoother acceleration/deceleration compared to constant-velocity rotation or linear reciprocation with hard stops [S1, unofficial].

## How it works
**Inputs and Outputs:**
- **Input:** A 3D voxel buffer and a hardware synchronization signal (SPIN_SYNC) from the motion controller [S1, unofficial].
- **Output:** Pixel data sent to an LED matrix or projector, synchronized to the instantaneous angle of the oscillating screen [S1, unofficial].

**Data Structures:**
The voxel buffer is a 3D array of `pixel_t` (8-bit RGB332) with dimensions $X=64, Y=64, Z=32$ [S1, unofficial].
- **Memory Layout:** Row-major.
  - `VOXEL_X_STRIDE = 1`
  - `VOXEL_Z_STRIDE = 64`
  - `VOXEL_Y_STRIDE = 2048`
  - Index formula: `VOXEL_INDEX(x, y, z) = x + y * 2048 + z * 64` [S1, unofficial].
- **Total Voxels:** 262,144 [S1, unofficial].
- **Double Buffer:** The shared memory buffer is sized at `2 * 262,144 * sizeof(pixel_t)` to allow frame swapping without tearing [S1, unofficial].

**Parameters:**
- **Pixel Format:** RGB332 (8-bit) [S1, unofficial].
  - Red: 3 bits (mask `0xE0`), Green: 3 bits (mask `0x1C`), Blue: 2 bits (mask `0x03`) [S1, unofficial].
  - Conversion macros:
    - `R332(p) = (((p) >> 5) & 7) * 36`
    - `G332(p) = (((p) >> 2) & 7) * 36`
    - `B332(p) = ((p) & 3) * 85` [S1, unofficial].
- **Fixed-Point Precision:**
  - `REVOLUTION_PRECISION = 20` (bits for frequency/period scaling) [S1, unofficial].
  - `ANGLE_PRECISION = 10` (bits for angle index) [S1, unofficial].
  - `SINCOS_PRECISION = 12` (bits for sine/cosine table values) [S1, unofficial].
- **Sine/Cosine Table:** 1024 entries (`1 << 10`) [S1, unofficial]. Values are fixed-point integers scaled by $2^{12}$ [S1, unofficial].
- **GPIO Mapping (Raspberry Pi BCM2711):**
  - RGB Data: Pins 12, 9, 6, 5, 8, 7, 21, 13, 20, 26, 19, 16 [S1, unofficial].
  - Row Select (A-E): Pins 4, 15, 18, 17, 14 [S1, unofficial].
  - Control: `RGB_BLANK`=11, `RGB_CLOCK`=0, `RGB_STROBE`=3, `SPIN_SYNC`=1 [S1, unofficial].

**Algorithm Steps:**
1. **Initialize Sine/Cosine Table:** Iterate $i$ from 0 to 1023. Compute angle $a = i \cdot \frac{2\pi}{1023}$. Store `round(cos(a) * 4096)` in `sincosfixed[i][0]` and `round(sin(a) * 4096)` in `sincosfixed[i][1]` [S1, unofficial].
2. **Initialize Span Offsets (Partial):** Compute pitch $ef / 64$. Set $vx = -ef/2$. Set $vy = de + \sqrt{bd^2 - ac^2}$. Set $vsh = 1/(pitch \cdot 0.8)$. Set $vsv = 1/(pitch \cdot 0.8)$. Loop $i$ from 0 to 1023: $\theta = i \cdot \frac{2\pi}{1023}$. Compute $bx = \cos(\theta) \cdot ab - ac$. Compute $by = \sin(\theta) \cdot ab$. Compute $ddy = \sqrt{bd^2 - bx^2}$ [S1, unofficial].
3. **Angle Tracking Loop (Real-time):** Read the hardware timer (`current_tick`) and the sync pin state. On a rising edge of the sync pin: Measure the period: $T = \text{current\_tick} - \text{sync\_prev\_tick}$. Clamp period: $T = \max(2^{10}, T)$ to prevent division by zero or overflow [S1, unofficial]. Compute revolution frequency: $f = \frac{2^{20}}{T}$. Update current angle: $\theta = (\text{current\_tick} - \text{sync\_prev\_tick}) \cdot f$. Update `sync_prev_tick = current_tick` [S1, unofficial].
4. **Projection:** Map the current 3D voxel coordinates to 2D screen coordinates based on the instantaneous angle $\theta$ derived from the oscillating motion. Project the corresponding slice onto the LED matrix. Synchronize the image update with the screen's position to ensure temporal integration blends the slices into a continuous volume [G].

**Pitfalls:**
- Revolution period measurement can be zero or extremely small on startup or noise, causing division by zero or frequency overflow. Handled by clamping `revolution_period = max(1<<10, revolution_period)` [S1, unofficial].
- File loading loop uses `do-while(frames > 1)` but `frames` is calculated as `file_size / volume_size`. If `frames` is 0 or 1, the loop still runs once. The `rtot` accumulator wraps at `volume_size` but the loop condition doesn't check `rtot` against `volume_size` for termination, relying on `frames` [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math
import time

# Constants from source [S1, unofficial]
REVOLUTION_PRECISION = 20
ANGLE_PRECISION = 10
SINCOS_PRECISION = 12
TABLE_SIZE = 1 << 10  # 1024
SINCOS_SCALE = 1 << SINCOS_PRECISION  # 4096

def init_sincos_table():
    """Initialize the sine/cosine lookup table [S1, unofficial]."""
    sincosfixed = [[0, 0] for _ in range(TABLE_SIZE)]
    for i in range(TABLE_SIZE):
        angle = (i * 2 * math.pi) / 1023.0
        sincosfixed[i][0] = round(math.cos(angle) * SINCOS_SCALE)
        sincosfixed[i][1] = round(math.sin(angle) * SINCOS_SCALE)
    return sincosfixed

def init_span_offsets(ef, de, bd, ac, ab):
    """Initialize span offset constants [S1, unofficial]."""
    pitch = ef / 64.0
    vx = -ef / 2.0
    vy = de + math.sqrt(max(0, bd**2 - ac**2))
    vsh = 1.0 / (pitch * 0.8)
    vsv = 1.0 / (pitch * 0.8)
    offsets = []
    for i in range(TABLE_SIZE):
        theta = (i * 2 * math.pi) / 1023.0
        bx = math.cos(theta) * ab - ac
        by = math.sin(theta) * ab
        ddy = math.sqrt(max(0, bd**2 - bx**2))
        offsets.append((bx, by, ddy))
    return vx, vy, vsh, vsv, offsets

def angle_tracking_loop(timer_uS, gpio_get_pin, sync_prev_tick, revolution_period, revolution_frequency):
    """Angle tracking loop [S1, unofficial]."""
    tick_curr = timer_uS()
    sync = gpio_get_pin(1)  # SPIN_SYNC pin
    sync_level = 1  # Assume active high

    if sync != sync_level:
        sync_level = sync
        if sync:  # Rising edge
            period = tick_curr - sync_prev_tick
            # Clamp period to minimum 1<<10 to prevent division by zero [S1, unofficial]
            if period < (1 << 10):
                period = 1 << 10
            frequency = (1 << REVOLUTION_PRECISION) // period
            sync_prev_tick = tick_curr
            return sync_prev_tick, period, frequency
    return sync_prev_tick, revolution_period, revolution_frequency

def main_loop():
    sincos = init_sincos_table()
    vx, vy, vsh, vsv, offsets = init_span_offsets(100, 50, 80, 10, 60)

    sync_prev_tick = 0
    revolution_period = 1 << 26
    revolution_frequency = 1

    while True:
        sync_prev_tick, revolution_period, revolution_frequency = angle_tracking_loop(
            lambda: int(time.time() * 1000000), lambda pin: 1, sync_prev_tick, revolution_period, revolution_frequency
        )

        # Compute current angle index for table lookup [S1, unofficial]
        delta_ticks = int(time.time() * 1000000) - sync_prev_tick
        current_angle_index = (delta_ticks * revolution_frequency) >> ANGLE_PRECISION
        current_angle_index %= TABLE_SIZE

        # Projection mapping logic not fully detailed in the source; implement angle-to-screen coordinate transformation here.
        # Use sincos[current_angle_index] and offsets to map 3D voxel coordinates to 2D LED matrix positions.
        pass
```

## Subtopics

### Reciprocating Volumetric Display (VX1)
A volumetric display that creates 3D images by projecting horizontal layer slices onto a single piece of rear-projection glass moving back and forth in the air, synchronized with a high-speed projector [S2, unofficial]. It breaks a 3D form into horizontal layer slices, moves the glass back and forth at 15 cycles per second using harmonic resonance springs, and tracks the glass location to synchronize with a 4,000 frames per second projector so each slice is projected at exactly the right height [S2, unofficial]. The slices are stacked and re-stacked rapidly so the eye cannot track the motion, creating a floating object illusion [S2, unofficial]. The image is redrawn on both the up and down swing of the glass to achieve a 30 frames per second hologram video refresh rate [S2, unofficial]. Parameters include a glass movement frequency of 15 cycles per second, a projector frame rate of 4,000 frames per second, a video refresh rate of 30 frames per second, and a display volume of 18x18x8 cm (approx. 7x7x3 inches) [S2, unofficial]. Pitfalls include projector beam divergence, multiple color projections, and synchronization issues requiring advanced programming and processing [S2, unofficial]. It is used in the Voxon VX1 table, sold for US$10,000 [S2, unofficial].

## Operating characteristics
- **Mechanical:** Reduces jerk at endpoints compared to linear reciprocation or constant-velocity rotation with hard stops, potentially allowing for higher acceleration or longer mechanical life [S1, unofficial].
- **Performance:** Implemented on a Raspberry Pi 4 driving a 64x64 LED matrix. The driver exposes a memory-mapped voxel buffer compatible with the Multivox ecosystem [S1, unofficial].
- **Limitations:** The source notes the project was a test object and "didn't take it beyond this test object," implying limited performance data or optimization for large volumes [S1, unofficial].
- **Precision:** Fixed-point arithmetic is used for angle and frequency calculations to avoid floating-point overhead on the embedded controller [S1, unofficial].

## Background
The Oscillating Swept Volume Display, also known as Oscivox, was developed by AncientJames as an experimental swept volume display [S1, unofficial]. It was designed to explore the benefits of linear volume displays (such as simpler mechanics or specific projection geometries) while mitigating the mechanical jerk associated with reversing direction in linear or rotational sweeps by using an undulating (sinusoidal-like) motion profile [S1, unofficial]. The implementation is part of the Multivox ecosystem, providing a driver for Raspberry Pi-based volumetric displays [S1, unofficial].

## Sources
[S1] AncientJames/oscivox, https://github.com/AncientJames/oscivox, fetched 2026-10-26 (unofficial)
[S2] Voxon hologram volumetric display (2019), https://newatlas.com/vr/voxon-photonics-3d-hologram-volumetric-displays/, fetched 2026-10-26 (unofficial)
