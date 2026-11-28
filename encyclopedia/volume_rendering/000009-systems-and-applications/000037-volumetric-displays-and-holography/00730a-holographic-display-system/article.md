# Holographic Display System

Also called: HoloDis system, Holographic rendering system, IMEC HoloDis, IMEC Holographic Display, Scalable holographic display

## Overview
A scalable hardware and software system for multi-user interactive computer graphics that uses an array of micro-displays and a holographic screen to emit light beams of different color and intensity in controlled directions, composing a continuous 3D view [S1, unofficial]. It differs from pure computational CGH methods by relying on a specific optical geometry where a holographic screen performs the final wavefront transformation, enabling horizontal-only parallax without the massive computational load of full volumetric wavefront synthesis [S1, unofficial].

## How it works
**Inputs:**
- Graphics commands (OpenGL-compatible) [S1, unofficial].
- DVI streams driving the micro-displays [S1, unofficial].

**Outputs:**
- A 3D light field with horizontal parallax only [S1, unofficial].

**Data Structures:**
- **Micro-display Array:** A specially arranged array of projectors. Each projector generates an array of pixels with controlled intensity and color [S1, unofficial].
- **Holographic Screen:** A passive optical element that transforms incident light beams into an asymmetrical pyramidal form [S1, unofficial].

**Steps:**
1. **Graphics Rendering:** An OpenGL-compliant library on a client PC redefines OpenGL behavior to multicast graphics commands to server PCs [S1, unofficial].
2. **Command Re-interpretation:** Server PCs re-interpret commands for holographic rendering, generating DVI streams [S1, unofficial].
3. **Projection:** Projectors generate an array of pixels at controlled intensity and color onto the holographic screen [S1, unofficial].
4. **Beam Generation:** Each point of the holographic screen transmits different colored light beams into different directions in front of the screen [S1, unofficial].
5. **Optical Transformation:** The holographic screen transforms the incident light beams into an asymmetrical pyramidal form, creating horizontal-only parallax [S1, unofficial].

**Parameters:**
- **Field of View (FOV):** 50 degrees [S1, unofficial].
- **Angular Resolution:** 0.8 degrees horizontal [S1, unofficial].
- **Vertical Diffusion:** 60 degrees [S1, unofficial].
- **Horizontal Diffusion:** 0.8 degrees [S1, unofficial].
- **Prototype Resolution:** 7.4M pixels [S1, unofficial].
- **Refresh Rate:** 10-15 Hz [S1, unofficial].
- **Drive Interface:** Three DVI streams [S1, unofficial].

**Edge Cases and Pitfalls:**
- **Vertical Parallax:** Not implemented in the prototype because it would require an order of magnitude increase in data size, rendering times, and system complexity for little gain in visual performance in standard settings [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Pseudocode for the Holographic Display System control flow
# Based on [S1, unofficial]

def initialize_system():
    """
    Initialize the client-server architecture and OpenGL hooks.
    """
    client_pc = ClientPC()
    server_pcs = [ServerPC() for _ in range(3)] # Driven by 3 DVI streams
    client_pc.hook_opengl() # Redefines OpenGL behavior to multicast commands

def render_frame(scene_objects):
    """
    Render a single frame of the holographic display.
    """
    # 1. Multicast graphics commands to servers
    commands = client_pc.reinterpret_opengl_commands(scene_objects)
    
    # 2. Servers process commands and generate DVI streams
    dvi_streams = []
    for server in server_pcs:
        stream = server.process_commands(commands)
        dvi_streams.append(stream)
        
    # 3. Projectors generate pixels on the holographic screen
    # The screen is an array of pixels with controlled intensity and color
    holographic_screen = HolographicScreen()
    for i, stream in enumerate(dvi_streams):
        holographic_screen.project(stream)
        
    # 4. Holographic screen transforms beams
    # Each point on the screen emits light beams of different color/intensity
    # into different directions, creating horizontal-only parallax
    # The distribution is an asymmetrical pyramid:
    # Vertical size = vertical FOV (60 degrees)
    # Horizontal size = neighboring emitting directions (0.8 degrees resolution)
    light_field = holographic_screen.transform_beams()
    
    return light_field

def main():
    initialize_system()
    while True:
        scene = get_next_scene()
        render_frame(scene)
        # Refresh rate is 10-15 Hz
        wait_for_refresh()
```

## Subtopics

### IMEC HoloDis
A nanoscale SiGe-MEMS chip with over 1 billion micro-diffractive nanodevices (DNDs) acting as visible-light-diffracting elements [S2, unofficial]. It uses a pixel pitch of 500 nm to achieve a viewing angle of approximately 38° (19° off axis) with 635 nm illumination [S2, unofficial]. The ultimate system targets a 60° diffraction angle and HD quality [S2, unofficial].

### Scalable Holographic Display (CRS4)
The prototype described in the main text, using an array of micro-displays and a holographic screen for horizontal-only parallax [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not explicitly defined in terms of algorithmic complexity, but rendering is driven by DVI streams at 10-15 Hz for 7.4M pixels [S1, unofficial].
- **Memory/Complexity:** Adding vertical parallax would increase data size, rendering times, and system complexity by an order of magnitude [S1, unofficial].
- **Performance:** Prototype achieved 7.4M pixels at 10-15 Hz refresh rate [S1, unofficial].
- **Strengths:** Scalable design for multi-user interactive graphics; controlled emission of light beams; horizontal-only parallax sufficient for standard settings [S1, unofficial].
- **Limitations:** No vertical parallax in the prototype; limited viewing angle (50° FOV, 0.8° angular resolution) compared to ultimate goals (60° diffraction angle) [S1, unofficial].

## Background
The system was presented by Balogh et al. in 2005 [S1, unofficial]. It targets multi-user interactive computer graphics applications. The feasibility was evaluated with a working hardware and software prototype [S1, unofficial]. IMEC also developed a related nanoscale MEMS-based holographic display (HoloDis) [S2, unofficial].

## Sources
[S1] A Scalable Hardware and Software System for the Holographic Display of Interactive Graphics Applications, https://doi.org/10.2312/egs.20051036, fetched 2026-10-29 (unofficial)
[S2] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
