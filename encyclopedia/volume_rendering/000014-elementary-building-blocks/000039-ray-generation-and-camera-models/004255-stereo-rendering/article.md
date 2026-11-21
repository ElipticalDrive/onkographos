# Stereo Rendering

Also called: Anaglyph Stereo, Red-Blue (Anaglyph) Stereo, Red-cyan stereo, Red-green stereo

## Overview
Stereo rendering generates a pair of images to simulate binocular parallax, where each eye sees a slightly different picture to determine depth [S1, book]. It differs from standard rendering by requiring two distinct camera viewpoints separated by an inter-ocular distance [S1, book]. Unlike [Depth of Field and Lens Effects](6232-depth-of-field-and-lens-effects.md) which simulates focus, stereo rendering simulates disparity [G]. The presentation method determines how these images are combined and viewed [S1, book].

## How it works
### Inputs and Outputs
- **Inputs**: Scene geometry, camera parameters (position, orientation, frustum), eye separation distance, and display type (single vs. dual) [S1, book].
- **Outputs**: A stereo image pair (either separate buffers, a combined anaglyph, or interlaced frames) [S1, book].

### Data Structures
The implementation relies on standard camera frustum data structures extended with stereo parameters [G].
- **Camera Frustum**: Defined by origin $\mathbf{o}$, direction $\mathbf{d}$, and projection matrix [G]. For single-display systems, the frustum must be sheared to account for the non-perpendicular view plane relative to the combined image [S1, book].
- **Image Buffers**: Standard RGB uses 24 bits per pixel (8 bits per channel) [S1, book]. Stereo double buffering requires quad buffering (Left Front, Left Back, Right Front, Right Back) [S1, book]. Memory for a 1Kx1K display with 24-bit RGB, 24-bit back buffer color, and 24-bit z-buffer per view is 144 bits per pixel or 18 MB total [S1, book].

### Steps
1. **Calculate Eye Positions**: Determine left and right eye origins based on the center eye position and eye separation distance [S1, book].
2. **Set Camera Configuration**: For two-display systems, perform camera azimuths to reach the left and right eye positions [S1, book]. For single-display systems, translate and shear the camera's viewing frustum to handle the non-perpendicular view plane [S1, book].
3. **Generate Images**: Render the scene from both eye positions to generate the left and right eye images [S1, book].
4. **Convert to Intensity**: Convert the RGB values of both views into grayscale intensities [S1, book].
5. **Combine Channels (Anaglyph)**: Create the resulting image where red values are taken from left eye image intensities and blue values (mixture of blue and green) are taken from right eye image intensities [S1, book].
6. **Display**: Invoke stereo rendering mode on the render window [S1, book].

### Formulas
The anaglyph channel mapping converts intensities $I_L$ (left) and $I_R$ (right) into output channels:
$$R_{out} = I_L$$
$$B_{out} = I_R$$
$$G_{out} = I_R \quad \text{(Red-green variant utilizes eye sensitivity to green)}$$
This discards original hue and saturation [S1, book].

### Parameters
- **Eye Separation Distance**: The distance between the left and right eye positions. Controlling this amount directly controls the generated parallax [S1, book]. Typical values approximate human inter-ocular distance (~6.5 cm) but are often adjusted for virtual scale [S1, book].

### Edge Cases and Pitfalls
- **Precision**: High bits-per-pixel requirements for double-buffered stereo can exhaust video memory (e.g., 144 bits/pixel for 1Kx1K) [S1, book].
- **Screen Coverage**: Time-multiplexed techniques require the application to take up the entire screen while rendering [S1, book].
- **Color Loss**: Anaglyph methods discard original hue and saturation [S1, book].

### Pseudocode
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def setup_stereo_rendering(render_window, stereo_type="RedBlue"):
    render_window.StereoRenderOn()
    if stereo_type == "RedBlue":
        render_window.SetStereoTypeToRedBlue()
    # Further rendering will be done in stereo
```

## Subtopics

### [Stereo Pseudo 3D Rendering via Image Stacks](14905-stereo-pseudo-3d-rendering-via-image-stacks.md)
Generates a stereo pair from a pre-existing 2D image stack to simulate binocular parallax and volumetric depth perception using standard web technologies. It is distinguished from full ray-casting volume rendering by relying on image overlay and CSS positioning rather than per-pixel shading, and from single-slice viewers by allowing simultaneous visibility of multiple depth layers. Choose when visualizing scientific volumetric data in a browser without specialized hardware. [Stereo Pseudo 3D Rendering via Image Stacks](14905-stereo-pseudo-3d-rendering-via-image-stacks.md)

### Time Multiplexed Stereo (Interlaced)
Each eye is updated at the original refresh rate (e.g., 60Hz) with full horizontal resolution but half vertical resolution [S1, book]. Renders the left eye image to the top half of the screen and the right eye image to the bottom half, then enters a special stereo mode on the graphics card to double the monitor's refresh rate (e.g., from 60Hz to 120Hz) [S1, book]. Choose when full horizontal resolution is prioritized over vertical resolution and screen takeover is acceptable [S1, book].

### Time Multiplexed Stereo (Interlaced)
Uses TV standards (NTSC/PAL) interlacing, rendering left eye to even lines and right eye to odd lines [S1, book]. Viewed with liquid crystal shutter glasses synchronized to the interlacing rate [S1, book]. Choose for television display compatibility, though it suffers from low resolution and flicker because each eye is updated at half the refresh rate [S1, book].

### Time Parallel Stereo
Displays both left and right images simultaneously on two separate screens (one per eye) [S1, book]. Requires two graphics cards or one card with two outputs [S1, book]. Choose when simultaneous full-resolution views are required and hardware cost is not a constraint [S1, book].

### Polarized Stereo Rendering
Preserves all color information by separating views using polarized light: one eye's image projected through a vertical filter, the other through a horizontal filter [S1, book]. User wears glasses with a vertical filter over one eye and a horizontal filter over the other [S1, book]. Choose when color fidelity is critical and polarized projection hardware is available [S1, book].

### Dual Buffer Stereo Rendering
Uses separate left and right image buffers, allowing stereo rendering without taking over the entire screen [S1, book]. Requires more memory or lower resolution (e.g., 144 bits per pixel for 1Kx1K with quad buffering) [S1, book]. Choose for high-quality interactive applications on supported hardware without full-screen takeover [S1, book].

### SIRDS (Single Image Random Dot Stereogram)
Displays both views in a single image using repeating dot patterns, where depth is incorporated by changing the interval between patterns [S1, book]. Viewed by focusing in front of or behind the image until two triangular cutouts in the top appear as one [S1, book]. Choose for static, depth-only images requiring no special hardware [S1, book].

## Operating characteristics
- **Time Complexity**: Linear with respect to pixels and scene complexity, multiplied by the number of eyes (typically 2) [S1, book].
- **Memory Complexity**: Increases significantly for double-buffered stereo; quad buffering can require up to 144 bits per pixel [S1, book].
- **Performance**: Time multiplexed methods halve vertical resolution per eye. Interlaced methods halve vertical resolution and suffer from flicker [S1, book]. Anaglyph methods have minimal performance overhead but poor color fidelity [S1, book].
- **Scaling**: Resolution scales with display hardware; interlaced and split-screen methods halve vertical resolution per eye, while dual buffer maintains full resolution at the cost of memory [S1, book].
- **Strengths**: Anaglyph is low-cost and works on any display (monitor, paper, or film) with inexpensive glasses [S1, book]. Dual buffer preserves full resolution and color but requires hardware support [S1, book].
- **Limitations**: Anaglyph loses color. Interlaced has low resolution and flicker. Time parallel requires expensive hardware [S1, book]. SIRDS requires user effort to focus and loses texture/color [S1, book].
- **When to use**: Use Anaglyph for low-cost distribution [S1, book]. Use Dual Buffer for high-quality interactive applications with supported hardware [S1, book]. Use SIRDS for static depth-only images [S1, book].

## Background
Stereo rendering originated from the need to simulate binocular vision in computer graphics [S1, book]. Foundational concepts of camera azimuths for two-display systems and frustum shearing for single-display systems were detailed in early computer graphics literature, specifically in the context of the Visualization Toolkit (VTK) and general advanced computer graphics textbooks [S1, book]. The term encompasses various methods including anaglyph, interlaced, and polarized techniques, adapted from television and display technologies [S1, book].

## Sources
[S1] Chapter 7 - Advanced Computer Graphics - VTK, https://examples.vtk.org/site/VTKBook/07Chapter7/, fetched 2026-10-29 (book)
