# Projector with a Lenticular Mirror Sheet

Also called: Lenticular Mirror Sheet Projection, Reflective Lenticular Display

## Overview
A projection-based multiview display that uses a single projector and a reflective lenticular mirror sheet as the screen to direct reflected beams toward different viewing perspectives [S1, unofficial]. It differs from transmissive lenticular displays by using reflection, allowing for brighter images and potentially simpler optical alignment in certain form factors, and from multiprojector systems by requiring only one light source [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A single projector, a reflective lenticular mirror sheet (a sheet of cylindrical lenses with a reflective backing), and a multiview image content stream.
- Output: A 3D image visible from multiple discrete horizontal viewpoints.

**Data Structures:**
- The lenticular sheet consists of a linear array of thick plano-convex cylindrical lenses (lenticules). The sheet is reflective, meaning the back surface of the lenses or a layer behind them reflects light.
- The projector image is spatially partitioned into strips, where each strip corresponds to the field of view of one or more lenticules.

**Steps:**
1. **Image Preparation:** The multiview image content is split into horizontal strips. Each strip contains the pixel data intended for the specific angular range covered by a corresponding lenticule strip on the screen.
2. **Projection:** The projector projects these multipixel image strips onto the surface of the lenticular mirror sheet. The projector is positioned such that its optical axis intersects the screen at a specific angle and distance.
3. **Optical Refraction and Reflection:**
   - Light from the projector enters the curved surface of the lenticular lenses.
   - The curvature of the cylindrical lenses refracts the light. Due to the geometry of the lenticules and the position of the projector, the light is directed towards the reflective backing.
   - The light reflects off the mirror layer.
   - Upon exiting the lenses (second pass through the lens material), the light is refracted again.
4. **Viewpoint Separation:** The combined effect of the two refractions (entry and exit) and the reflection directs the light from each projector pixel strip into a specific, narrow angular cone. Different strips are directed to different angles, creating distinct viewing zones (views) for the observer.
5. **Calibration:** The system requires precise calibration of the projector's position, angle, and the projected image geometry relative to the lenticular sheet's pitch and focal length to ensure that each view is correctly aligned with the corresponding lenticule strip [S1, unofficial].

**Formulas and Parameters:**
- The relationship between the projector position, lenticule pitch ($p$), lens focal length ($f$), and viewing angle ($\theta$) determines the mapping. While exact formulas are not provided in the sources, the principle relies on the lens equation and Snell's Law applied to the cylindrical geometry.
- Typical parameters include the lenticule pitch (often in the range of hundreds of micrometers to millimeters depending on resolution requirements) and the focal length of the lenticules, which must match the projector's throw distance and angle to focus the image correctly on the reflective surface and then collimate or direct it to the viewer [S1, unofficial].

**Pitfalls:**
- **Calibration Sensitivity:** Misalignment between the projector and the lenticular sheet results in crosstalk between views or blurred images. The calibration is sensitive to the projector's position and the screen's location [S1, unofficial].
- **Precision:** The projection image must be carefully calibrated to the lenticular screen location to produce a multiview 3D display [S1, unofficial].

## Subtopics

### Multiview 3D Display Using Multiple Projectors and Lenticular Sheet
A variant using multiple projectors, each creating images for a single view, projected onto a special lenticular reflective screen [S1, unofficial]. In the vertical direction, diffuse light in all directions. In the horizontal direction, light is focused onto the reflective diffuser screen and projected back to the projector direction. Light passes through the vertically oriented lenticular screen twice: first, lenses focus projector pixels onto the diffuse screen; second, lenses redistribute pixels back to the same angular direction [S1, unofficial]. This method preserves the resolution of projected images for each view and is scalable without significant cost increase compared to spatial multiplex methods, unlike the single-projector approach which may have lower per-view resolution if the projector's resolution is shared [S1, unofficial]. However, it suffers from exorbitant expense due to one projector per view and difficulty of calibration, as maintaining optical calibration for many projectors is challenging [S1, unofficial]. Experimental systems have been produced with more than 100 views [S1, unofficial].

## Operating characteristics
- **Time Complexity:** Not given in the sources.
- **Memory Complexity:** Not given in the sources.
- **Performance:** The single-projector design is less expensive than multiprojector systems but may have lower per-view resolution if the projector's total resolution is divided among views. The multiprojector variant preserves full resolution per view [S1, unofficial].
- **Strengths:** High optical efficiency due to reflection; scalable screen size in multiprojector variants; capable of producing high-quality 3D images with many views (e.g., >100 views) [S1, unofficial].
- **Limitations:** Single-projector systems require precise calibration and may have limited viewing angles or resolution. Multiprojector systems are expensive and difficult to calibrate [S1, unofficial].
- **When to use:** When high brightness and good optical efficiency are required, and when the cost of multiple projectors can be justified (multiprojector) or when a simpler, lower-cost setup is needed with acceptable resolution/view trade-offs (single projector).

## Background
This method was proposed by Krah at Apple [S1, unofficial]. It is part of the broader field of projection-based multiview 3D displays. Similar displays have been demonstrated in various laboratories [S1, unofficial].

## Sources
[S1] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
