# Mechanically Driven Mirrors

Also called: Galvanometer driven, Piezoelectrically driven

## Overview
Mechanically driven mirrors are beam deflectors used in volumetric displays and holography to steer light beams. They are essentially lossless but suffer from slow response times compared to electro-optic alternatives [S1, unofficial]. They differ from [SliceView](7774-sliceview.md) by using dynamic reflection rather than static semi-reflective layers, and from [Holographic Magnification via Diverging Light](14730-holographic-magnification-via-diverging-light.md) by using physical inertia rather than liquid crystal phase arrays for beam steering.

## How it works
**Inputs and Outputs:**
- Input: Commanded deflection angle $\theta$.
- Output: Reflected light beam at angle $2\theta$ (assuming normal incidence).

**Data Structures:**
Not given in the sources.

**Steps:**
1. Receive a command for a target deflection angle.
2. Actuate the mirror mechanism (galvanometer or piezoelectric) to rotate the mirror surface.
3. Reflect the incident light beam off the mirror surface.
4. Stabilize the mirror to the target angle before emitting the next spot or frame [S1, unofficial].

**Formulas:**
- Natural frequency $f_n$ determines the time response in random access operation [S1, unofficial].
- Maximum spot capability $N_{max}$ for a refresh rate $R_{refresh}$ is derived from the natural frequency:
  $$ N_{max} \approx \frac{f_n}{R_{refresh}} $$
  [S1, unofficial]

**Parameters:**
- **Piezoelectric units:**
  - Lowest natural frequency: $\approx 12 \text{ KHz}$ [S1, unofficial].
  - Maximum spot capability at $30 \text{ Hz}$ refresh: $400$ spots [S1, unofficial].
- **Galvanometer units:**
  - Lowest natural frequency: $\approx 3 \text{ KHz}$ [S1, unofficial].
  - Maximum spot capability at $30 \text{ Hz}$ refresh: $100$ spots [S1, unofficial].

**Pitfalls and Edge Cases:**
- **Hysteresis (Galvanometer):** Galvanometer drives are susceptible to hysteresis effects as large as $4\%$ [S1, unofficial]. To overcome this, the system must always approach a newly commanded deflection angle from the same direction. This requirement nearly doubles the random access time [S1, unofficial].
- **Hysteresis (Piezoelectric):** Piezoelectric drives circumvent hysteresis by using active position sensing [S1, unofficial].
- **Underdamped Oscillation:** High-response mechanical mirror drives often operate in a highly underdamped manner, producing slowly decaying oscillations about the mean orientation [S1, unofficial]. This enlarges the apparent spot size, decreases boundary definition, and causes misregistration [S1, unofficial]. Additional damping is required to prevent viewing degradation, which further decreases drive response [S1, unofficial].
- **Pointing Accuracy:** Pointing accuracies of $0.1$ spot diameter are generally sufficient [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import time

class MechanicallyDrivenMirror:
    def __init__(self, drive_type, natural_frequency_hz):
        """
        Initialize a mechanically driven mirror.
        
        Args:
            drive_type: 'galvanometer' or 'piezoelectric'.
            natural_frequency_hz: The lowest natural frequency of the unit.
        """
        self.drive_type = drive_type
        self.natural_frequency_hz = natural_frequency_hz
        self.current_angle = 0.0
        self.last_direction = None # For galvanometer hysteresis management

    def set_target_angle(self, target_angle):
        """
        Command the mirror to a new deflection angle.
        
        Args:
            target_angle: The desired deflection angle in degrees.
        """
        if self.drive_type == 'galvanometer':
            # Galvanometer hysteresis mitigation:
            # Always approach from the same direction.
            # If changing direction, must traverse full range or use a specific protocol.
            # This nearly doubles random access time.
            if self.last_direction is not None:
                if (target_angle > self.current_angle) != (self.last_direction > 0):
                    # Direction change detected; full settling required
                    pass 
            self.last_direction = 1 if target_angle > self.current_angle else -1
        
        # Actuate mirror
        self._actuate(target_angle)
        
        # Wait for settling based on natural frequency and damping
        # High response drives are underdamped; oscillations decay slowly.
        # Additional damping reduces response time.
        settling_time = self._calculate_settling_time(target_angle)
        time.sleep(settling_time)
        
        self.current_angle = target_angle

    def _actuate(self, angle):
        """
        Physically move the mirror to the target angle.
        """
        # Implementation depends on hardware (voltage/current control)
        pass

    def _calculate_settling_time(self, angle):
        """
        Estimate time required for the mirror to settle within 0.1 spot diameter accuracy.
        """
        # Based on natural frequency and damping characteristics
        # Not given in the sources. [G]
        return 1.0 / self.natural_frequency_hz

    def reflect_beam(self, incident_beam):
        """
        Reflect an incident beam off the current mirror orientation.
        
        Args:
            incident_beam: Vector representing the incident light direction.
            
        Returns:
            Reflected beam vector.
        """
        # Reflection calculation based on current_angle
        # Not given in the sources. [G]
        pass
```

## Subtopics

### Beam deflector (BD) steering
A liquid crystal-based transmission phase array that optically tilts coherent backlight beams to expand the holographic viewing angle [S2]. It differs from mechanical mirrors by using electro-optic phase shifts rather than physical inertia. Maximum steering angle is $\sim 15^\circ$ with an angular resolution of $0.02^\circ$ at a wavelength of $520 \text{ nm}$ [S2]. A key pitfall is that the maximum steering angle decreases as the light source area increases due to etendue constraints [S2].

## Operating characteristics
- **Time Complexity:** Response time is limited by the mechanical natural frequency. Galvanometer units ($\approx 3 \text{ KHz}$) are slower than piezoelectric units ($\approx 12 \text{ KHz}$) [S1, unofficial].
- **Memory Complexity:** Not given in the sources.
- **Measured Performance:** At a $30 \text{ Hz}$ refresh rate, piezoelectric units support a maximum of $400$ spots, while galvanometer units support $100$ spots [S1, unofficial].
- **Scaling:** Scaling is limited by the trade-off between damping and response time; additional damping reduces oscillations but decreases drive response [S1, unofficial].
- **Strengths:** Essentially lossless efficiency [S1, unofficial].
- **Limitations:** Slow response time compared to electro-optic deflectors; susceptibility to hysteresis (galvanometer); underdamped oscillations causing misregistration [S1, unofficial].
- **When to use:** When high optical efficiency is required and the application can tolerate slower frame rates or spot counts.
- **When not to use:** When high-speed random access or high spot counts are required without the overhead of hysteresis compensation.

## Background
Mechanically driven mirrors are among the simplest beam deflectors available for volumetric displays. They were studied in the context of 3D volumetric display systems in the 1970s [S1, unofficial]. The technology distinguishes between galvanometer-driven and piezoelectric-driven units based on their actuation mechanisms and natural frequencies [S1, unofficial].

## Sources
[S1] DTIC AD0753438: 3D Volumetric Display Study, https://archive.org/details/DTIC_AD0753438, fetched 2026-10-26 (unofficial)
[S2] Slim-panel holographic video display, https://doi.org/10.1038/s41467-020-19298-4, fetched 2026-10-26
