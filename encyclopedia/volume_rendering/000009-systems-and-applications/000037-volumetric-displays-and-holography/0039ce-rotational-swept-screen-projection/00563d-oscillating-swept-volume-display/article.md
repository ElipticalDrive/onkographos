# Oscillating Swept Volume Display

Also called: Oscivox, Undulating Volumetric Display

## Overview
The Oscillating Swept Volume Display (OSVD) is a volumetric display method that replaces continuous rotation or linear translation with an undulating (back-and-forth) mechanical motion to reduce mechanical jerk at the sweep endpoints [S1, unofficial]. It builds on [Rotational Swept-Screen Projection](8589-rotational-swept-screen-projection.md) by maintaining synchronous projection but altering the kinematic profile to a sinusoidal oscillation, allowing for smoother acceleration/deceleration compared to constant-velocity rotation or linear reciprocation with hard stops.

## How it works
**Inputs and Outputs:**
- **Input:** A 3D voxel buffer and a hardware synchronization signal (SPIN_SYNC) from the motion controller.
- **Output:** Pixel data sent to an LED matrix or projector, synchronized to the instantaneous angle of the oscillating screen.

**Data Structures:**
The voxel buffer is a 3D array of `pixel_t` (8-bit RGB332) with dimensions $X=64, Y=64, Z=32$.
- **Memory Layout:** Row-major.
  - `VOXEL_X_STRIDE = 1`
  - `VOXEL_Z_STRIDE = 64`
  - `VOXEL_Y_STRIDE = 2048`
  - Index formula: `VOXEL_INDEX(x, y, z) = x + y * 2048 + z * 64`
- **Total Voxels:** 262,144.
- **Double Buffer:** The shared memory buffer is sized at `2 * 262,144 * sizeof(pixel_t)` to allow frame swapping without tearing [S1, unofficial].

**Parameters:**
- **Pixel Format:** RGB332 (8-bit).
  - Red: 3 bits (mask `0xE0`), Green: 3 bits (mask `0x1C`), Blue: 2 bits (mask `0x03`).
  - Conversion macros:
    - `R332(p) = (((p) >> 5) & 7) * 36`
    - `G332(p) = (((p) >> 2) & 7) * 36`
    - `B332(p) = ((p) & 3) * 85`
- **Fixed-Point Precision:**
  - `REVOLUTION_PRECISION = 20` (bits for frequency/period scaling)
  - `ANGLE_PRECISION = 10` (bits for angle index)
  - `SINCOS_PRECISION = 12` (bits for sine/cosine table values)
- **Sine/Cosine Table:** 1024 entries (`1 << 10`). Values are fixed-point integers scaled by $2^{12}$.
- **GPIO Mapping (Raspberry Pi BCM2711):**
  - RGB Data: Pins 12, 9, 6, 5, 8, 7, 21, 13, 20, 26, 19, 16.
  - Row Select (A-E): Pins 4, 15, 18, 17, 14.
  - Control: `RGB_BLANK`=11, `RGB_CLOCK`=0, `RGB_STROBE`=3, `SPIN_SYNC`=1 [S1, unofficial].

**Algorithm Steps:**

1. **Initialize Sine/Cosine Table:**
   Iterate $i$ from 0 to 1023.
   - Compute angle $a = i \cdot \frac{2\pi}{1023}$.
   - Store `round(cos(a) * 4096)` in `sincosfixed[i][0]`.
   - Store `round(sin(a) * 4096)` in `sincosfixed[i][1]`.

2. **Initialize Span Offsets (Partial):**
   - Compute pitch $ef / 64$.
   - Set $vx = -ef/2$.
   - Set $vy = de + \sqrt{bd^2 - ac^2}$.
   - Set $vsh = 1/(pitch \cdot 0.8)$.
   - Set $vsv = 1/(pitch \cdot 0.8)$.
   - Loop $i$ from 0 to 1023:
     - $\theta = i \cdot \frac{2\pi}{1023}$.
     - $bx = \cos(\theta) \cdot ab - ac$.
     - $by = \sin(\theta) \cdot ab$.
     - $ddy = \sqrt{bd^2 - bx^2}$.
   *(Note: The specific geometric meaning of variables $ef, de, bd, ac, ab$ is not fully defined in the source, but these constants are computed once at startup [S1, unofficial].)*

3. **Angle Tracking Loop (Real-time):**
   - Read the hardware timer (`current_tick`) and the sync pin state.
   - On a rising edge of the sync pin:
     - Measure the period: $T = \text{current\_tick} - \text{sync\_prev\_tick}$.
     - **Clamp Period:** $T = \max(2^{10}, T)$ to prevent division by zero or overflow [S1, unofficial].
     - Compute revolution frequency: $f = \frac{2^{20}}{T}$.
     - Update current angle: $\theta = (\text{current\_tick} - \text{sync\_prev\_tick}) \cdot f$.
     - Update `sync_prev_tick = current_tick`.

4. **Projection:**
   - Map the current 3D voxel coordinates to 2D screen coordinates based on the instantaneous angle $\theta$ derived from the oscillating motion.
   - Project the corresponding slice onto the LED matrix.
   - Synchronize the image update with the screen's position to ensure temporal integration blends the slices into a continuous volume [G].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants from source
REVOLUTION_PRECISION = 20
ANGLE_PRECISION = 10
SINCOS_PRECISION = 12
TABLE_SIZE = 1024
SINCOS_SCALE = 1 << SINCOS_PRECISION  # 4096

def init_sincos_table():
    """Initialize the sine/cosine lookup table."""
    sincosfixed = [[0, 0] for _ in range(TABLE_SIZE)]
    for i in range(TABLE_SIZE):
        # Angle calculation: i * 2pi / 1023
        # Note: Source uses 1023 as divisor for 1024 points, likely to close the loop
        angle = (i * 2 * math.pi) / 1023.0
        cos_val = round(math.cos(angle) * SINCOS_SCALE)
        sin_val = round(math.sin(angle) * SINCOS_SCALE)
        sincosfixed[i][0] = cos_val
        sincosfixed[i][1] = sin_val
    return sincosfixed

def get_current_angle(timer_uS, gpio_get_pin, sync_prev_tick, revolution_period, revolution_frequency):
    """
    Calculate current angle based on timer and sync pulses.
    Returns: (new_angle, new_sync_prev_tick, new_period, new_frequency)
    """
    tick_curr = timer_uS()
    sync_level = 1  # Assuming active high sync for this example
    
    # Read sync pin (simplified hardware access)
    sync = gpio_get_pin(1)  # SPIN_SYNC pin 1
    
    if sync != sync_level:
        sync_level = sync
        if sync:  # Rising edge
            period = tick_curr - sync_prev_tick
            # Clamp period to minimum 1<<10 to avoid division by zero
            if period < (1 << 10):
                period = 1 << 10
            
            # Compute frequency: (1<<20) / period
            frequency = (1 << REVOLUTION_PRECISION) // period
            
            return tick_curr, period, frequency
        else:
            # Falling edge or other state change, keep previous state
            return None, None, None
            
    # If no edge, angle advances based on current frequency and time delta
    # Note: The source implies angle is computed from tick delta * frequency
    # This part is inferred from "Update current angle as (current_tick - sync_prev_tick) * frequency"
    # However, strictly speaking, between sync pulses, the angle accumulates.
    # The source code snippet `current_angle()` suggests:
    # revolution_frequency is updated on sync.
    # The angle is likely accumulated or computed relative to the last sync.
    
    # Re-reading source: "Update current angle as (current_tick - sync_prev_tick) * frequency"
    # This formula seems to compute the angle *since the last sync pulse*.
    # For a full oscillation, we need to map this to the sine/cosine table.
    
    # Let's assume the function returns the delta angle or the raw value to be indexed.
    # The source uses `revolution_frequency` which is fixed point.
    
    # To implement strictly from source:
    # We need to track the angle index.
    # The source doesn't show the full accumulation loop, only the update on sync.
    # We will return the computed frequency and period for the caller to manage angle accumulation.
    
    return tick_curr, period, frequency

def main_loop():
    sincos = init_sincos_table()
    
    sync_prev_tick = 0
    revolution_period = 1 << 26  # Initial large period
    revolution_frequency = 1
    
    # GPIO and Timer setup would go here
    # timer_uS = ...
    # gpio_get_pin = ...
    
    while True:
        # Check for sync edge
        result = get_current_angle(timer_uS, gpio_get_pin, sync_prev_tick, revolution_period, revolution_frequency)
        
        if result[0] is not None:
            sync_prev_tick, revolution_period, revolution_frequency = result
            
        # Compute current angle index for table lookup
        # The source implies angle is derived from time delta * frequency
        # Since frequency is fixed-point (20 bits), and time is in microseconds,
        # we need to scale appropriately.
        # The source uses `current_tick - sync_prev_tick` which is the delta time.
        # But for continuous display, we need the absolute angle or phase.
        # The source code snippet is incomplete regarding the full angle accumulation.
        # However, the key insight is the undulating motion.
        # The angle $\theta$ maps to the sine/cosine table to determine the screen position.
        
        # Assuming we have a way to get the current phase of the oscillation
        # The source mentions "undulating motion", implying $\theta$ varies sinusoidally.
        # The sync pulse likely marks the peak or zero-crossing of the oscillation.
        
        # For the purpose of this implementation guide, we assume the hardware
        # provides a sync pulse at a consistent phase (e.g., peak amplitude).
        # The angle between sync pulses is integrated.
        
        # Placeholder for angle integration logic not fully detailed in source
        # current_angle_index = (delta_time * revolution_frequency) >> SHIFT
        # This would be used to index into sincos table for projection mapping.
        pass
```

## Operating characteristics
- **Mechanical:** Reduces jerk at endpoints compared to linear reciprocation or constant-velocity rotation with hard stops, potentially allowing for higher acceleration or longer mechanical life [S1, unofficial].
- **Performance:** Implemented on a Raspberry Pi 4 driving a 64x64 LED matrix. The driver exposes a memory-mapped voxel buffer compatible with the Multivox ecosystem [S1, unofficial].
- **Limitations:** The source notes the project was a test object and "didn't take it beyond this test object," implying limited performance data or optimization for large volumes [S1, unofficial].
- **Precision:** Fixed-point arithmetic is used for angle and frequency calculations to avoid floating-point overhead on the embedded controller [S1, unofficial].

## Background
The Oscillating Swept Volume Display, also known as Oscivox, was developed by AncientJames as an experimental swept volume display [S1, unofficial]. It was designed to explore the benefits of linear volume displays (such as simpler mechanics or specific projection geometries) while mitigating the mechanical jerk associated with reversing direction in linear or rotational sweeps by using an undulating (sinusoidal-like) motion profile [S1, unofficial]. The implementation is part of the Multivox ecosystem, providing a driver for Raspberry Pi-based volumetric displays [S1, unofficial].

## Sources
[S1] AncientJames/oscivox, https://github.com/AncientJames/oscivox, fetched 2026-10-26 (unofficial)
