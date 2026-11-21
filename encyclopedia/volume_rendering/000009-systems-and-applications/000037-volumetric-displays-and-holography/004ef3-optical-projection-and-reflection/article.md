# Optical Projection and Reflection


## Overview
Optical Projection and Reflection techniques generate volumetric or pseudo-volumetric images by reflecting light from a 2D display source onto semi-reflective surfaces, creating the illusion of floating 3D objects in free space [G]. This family solves the problem of presenting 3D content with natural occlusion cues without requiring physical particles or scanning beams in the viewing volume [G]. It differs from scanning methods (like [Galvanometer Scanner Volume Rendering](5277-galvanometer-scanner-volume-rendering.md)) by using static optical elements to create persistent images, and from holographic methods (like [Holographic Computer-Generated Holography (CGH) Calculation](6884-holographic-computer-generated-holography-cgh-calculation.md)) by relying on geometric reflection rather than wavefront interference [G].

## Shared principles
The core mechanism involves a display source emitting light that strikes a translucent or semi-reflective surface [G]. The surface reflects a portion of the light toward the viewer while transmitting the remainder [G]. By stacking multiple such surfaces or using specific geometries, distinct image slices are superimposed to create depth [G].

**Key Mathematical Principles:**

1. **Contrast and Occlusion:** The perception of a floating volume relies on high contrast between the projected image and the real background [G]. If $I_{img}$ is the intensity of the projected image and $I_{bg}$ is the intensity of the real background, the contrast $C$ at the boundary is defined as:
    $$ C = \frac{I_{img} - I_{bg}}{I_{img} + I_{bg}} $$
    High contrast (bright image, dark background) is essential for the "floating" effect [S1].

2. **Perspective Projection:** To simulate virtual images located behind the physical drawing volume, the display content must be projected based on the viewer's position [S1]. For an observer at point $E = (x_0, y_0, z_0)$ and an image point $X = (x, y, z)$, the perspective projection onto a plane $P$ (perpendicular to the line $EO$, where $O$ is the origin) is governed by the matrix $S$:
    $$ S = \begin{bmatrix} -r^2 & r^2 x_0 & 0 & 0 \\ -r x_0 z_0 & -r y_0 z_0 & r \rho^2 & 0 \\ 0 & 0 & 0 & r \rho \\ -\rho x_0 & -\rho y_0 & -\rho z_0 & r \rho \end{bmatrix} $$
    where:
    $$ r = \frac{1}{\sqrt{x_0^2 + y_0^2 + z_0^2}} $$
    $$ \rho = \frac{1}{\sqrt{x_0^2 + y_0^2}} $$
    [S1]

3. **Rendering Steps:** The system prepares image slices, applies perspective projection based on viewer position, emits light from the display source, and handles reflection/transmission at each layer to superimpose the final volume [S1, S2].

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def render_sliceview(viewer_pos, layers, display_params):
    """
    Render the SliceView volumetric display.
    
    Args:
        viewer_pos: Tuple (x0, y0, z0) representing viewer position E.
        layers: List of dicts, each containing:
            - 'image': 2D numpy array of the layer's content.
            - 'z': Depth of the layer.
            - 'R': Reflection coefficient.
            - 'T': Transmission coefficient.
        display_params: Dict containing optical parameters.
        
    Returns:
        None (displays images to respective layers)
    """
    x0, y0, z0 = viewer_pos
    
    # Calculate projection matrix components
    r = 1.0 / math.sqrt(x0**2 + y0**2 + z0**2)
    rho = 1.0 / math.sqrt(x0**2 + y0**2)
    
    # Construct projection matrix S
    S = [
        [-r**2, r**2 * x0, 0, 0],
        [-r * x0 * z0, -r * y0 * z0, r * rho**2, 0],
        [0, 0, 0, r * rho],
        [-rho * x0, -rho * y0, -rho * z0, r * rho]
    ]
    
    for layer in layers:
        image = layer['image']
        z = layer['z']
        R = layer['R']
        T = layer['T']
        
        # Apply perspective projection to the image
        # Sources do not specify the image warping algorithm; typically involves inverse mapping using the projection matrix [G].
        projected_image = apply_perspective_warp(image, S, z)
        
        # Display the projected image on the layer's display source
        # The semi-reflective surface will reflect R portion to viewer
        # and transmit T portion to subsequent layers/background
        # Sources do not specify hardware emission control; typically involves driving the display panel with the projected image buffer [G].
        display_layer(projected_image, R, T)

def apply_perspective_warp(image, S, z):
    """
    Warp the 2D image based on the perspective projection matrix S.
    """
    # Sources do not specify the image warping algorithm; typically involves inverse mapping using the projection matrix [G].
    return image 

def display_layer(image, R, T):
    """
    Emit light for the layer with specified reflection and transmission properties.
    """
    # Sources do not specify hardware emission control; typically involves driving the display panel with the projected image buffer [G].
    pass
```

## Subtopics

### [SliceView](7774-sliceview.md)
SliceView is a volumetric display system that extends the Pepper's Ghost optical illusion to multiple stacked translucent planes, cumulatively giving volume to an initially two-dimensional projection. It solves the problem of presenting 3D content in physical space without moving parts by superimposing distinct image slices at different depths, allowing the viewer to perceive a floating volume that occludes the real background. Unlike stereoscopic displays, it avoids vergence accommodation conflict (VAC) between the virtual object and the real background, and differs from persistence-of-vision displays by using static stacked planes rather than oscillating surfaces or levitated particles. Choose this method when natural occlusion and depth cues are required without mechanical scanning.

### [Mechanically Driven Mirrors](14032-mechanically-driven-mirrors.md)
Mechanically driven mirrors are beam deflectors used in volumetric displays to steer light beams toward specific points in a drawing volume. They solve the problem of directing light from a source to arbitrary $(x, y, z)$ coordinates within a 3D space by physically rotating mirrors. They are distinguished from electro-optic deflectors by being essentially lossless but suffering from slower response times. Choose this method when high optical efficiency is prioritized over high-speed beam steering.

### [Holographic Magnification via Diverging Light](14730-holographic-magnification-via-diverging-light.md)
This method provides a lensless magnification technique for holographic displays by using a diverging point light source to perform coarse beam steering, allowing a hologram to steer light at a finer angle to focus into an image. It solves the problem of achieving wide fields of view (60–100°) without the bulky refractive lenses required by conventional magnification. Unlike [SliceView](7774-sliceview.md) which relies on physical layer stacking, this method uses optical diffraction and divergence to create depth and magnification simultaneously. Choose this method when a wide field of view and lensless design are required.

### Perspective Projection Backdrop
A technique to simulate virtual images (located behind the physical drawing volume) in freespace volumetric displays by projecting content onto a background image plane that moves and rotates to match the viewer's position, creating motion parallax cues consistent with a distant object [S1].
**Definition:** Simulates virtual images in freespace displays using time-varying perspective projection [S1].
**Steps:**
1. Define observer position $E = (x_0, y_0, z_0)$ [S1].
2. Define 3D image point $X = (x, y, z)$ [S1].
3. Define background image plane $P$, typically perpendicular to line $EO$ where $O$ is origin [S1].
4. Calculate intersection of line $EX$ with plane $P$ to determine pixel coordinate [S1].
**Link:** No separate article.

### Pepper's Cone Display
A simple 360° display based on a conical semi-reflective surface that reflects an anamorphic image into a real undistorted aerial image. It has no moving parts [S2].
**Definition:** 360° display using conical semi-reflective surface [S2].
**Steps:**
1. Display an anamorphic image [S2].
2. Reflect it off a 360° symmetrical conical semi-reflective surface [S2].
3. Surface creates a real undistorted aerial image [S2].
4. As viewer turns device, image changes to offer 360° interactive view [S2].
**Pitfalls:** Constrained to a limited number of viewers because only one point of view is displayed at a time [S2].
**Link:** No separate article.

### Electro-optic Deflector
An analog electro-optic deflector using a series arrangement of electro-optic prisms. A light beam is deflected by varying the refractive index of each prism by applying an electric field to its top and bottom electroded faces. Alternating the electro-optic polarity allows all prisms to be excited from a single polarity electric field [S3, unofficial].
**Pitfalls:** The prism type suffers from high load capacitance, which limits random access times and increases complexity [S3, unofficial].
**Numbers:** Better than 300-spot resolution has been demonstrated in the prism type of electro-optic deflector [S3, unofficial].
**Comparison:** Unlike digital electro-optic deflectors which offer discrete beam positioning, this configuration yields an output continuously controllable in position [S3, unofficial].
**Link:** No separate article.

## Background
Optical projection and reflection techniques have roots in classical theatrical illusions, specifically Pepper's Ghost, which uses a large sheet of glass to reflect a hidden stage area into the main stage [S1, S2]. Modern volumetric applications have adapted these principles for digital displays. SliceView was developed as an extension of Pepper's Ghost to multiple planes to create volumetric depth cues [S2]. Perspective Projection Backdrop was introduced in 2021 to simulate virtual images in optical trap displays (OTDs) by employing time-varying perspective projection [S1]. Pepper's Cone was proposed as a low-cost, efficient 360° display solution using conical surfaces [S2]. Mechanically driven mirrors were documented in early volumetric display studies (1970s) as a method for beam deflection, noting trade-offs between speed, hysteresis, and damping [S3, unofficial]. These methods are used in research prototypes for volumetric displays, holography, and augmented reality interfaces where natural occlusion and parallax are desired without the complexity of scanning beams or particle traps [G].

## Sources
[S1] Simulating virtual images in optical trap displays, https://doi.org/10.1038/s41598-021-86495-6, fetched 2026-10-29
[S2] Multi-user volumetric 360° display based on retro-reflective transparent surfaces, https://doi.org/10.1364/oe.409181, fetched 2026-10-26
[S3] DTIC AD0753438: 3D Volumetric Display Study, https://archive.org/details/DTIC_AD0753438, fetched 2026-10-26 (unofficial)
