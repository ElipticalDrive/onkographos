# vtkSlicerGPURayCastVolumeMapper

Also called: GPGPU Ray Cast Mapper

## Overview
Renders a volume using ray casting on a 3D texture [S1, unofficial]. It distinguishes itself by exposing parameters for distance-based color blending and ICPE scaling, building on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) which manages the general GPU-accelerated pipeline [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: 3D scalar field data, transfer functions, camera parameters, and bounding box definition [G].
- Output: 2D rendered image [G].

**Data Structures:**
The mapper maintains the following internal state and OpenGL handles [S1, unofficial]:
- `double VolumeBBoxVertices[8][3]`: Stores the 8 vertices of the volume's bounding box, each with 3 coordinates [S1, unofficial].
- `double VolumeBBoxVerticesColor[8][3]`: Stores the 8 vertices of the bounding box with associated color values, each with 3 coordinates [S1, unofficial].
- `GLfloat ParaMatrix[16]`: A 4x4 matrix uniform passed to the shader for ray casting parameters [S1, unofficial].
- `GLfloat ParaMatrix1[16]`: A second 4x4 matrix uniform for additional ray casting parameters [S1, unofficial].
- `GLuint Volume1Index`: OpenGL handle for the volume texture [S1, unofficial].
- `GLuint ColorLookupIndex`: OpenGL handle for the color lookup table (CLUT) [S1, unofficial].
- `GLuint RayCastVertexShader`: OpenGL handle for the vertex shader [S1, unofficial].
- `GLuint RayCastFragmentShader`: OpenGL handle for the fragment shader [S1, unofficial].
- `GLuint RayCastProgram`: OpenGL handle for the shader program linking the vertex and fragment shaders [S1, unofficial].

**Parameters:**
- `DistanceColorBlending` (float, default 0.0): Controls brightness falloff with distance; positive values make closer voxels brighter and farther voxels darker [S1, unofficial].
- `DepthPeelingThreshold` (float): Threshold value used for depth peeling operations to handle transparency ordering [S1, unofficial].
- `ICPEScale` (float): Scale parameter for Intensity Contrast Preserving Enhancement (ICPE), used to adjust contrast during rendering [S1, unofficial].
- `Clipping` (int): Boolean flag (0 or 1) to enable or disable clipping planes [S1, unofficial].
- `Technique` (int): Integer selector for the specific rendering technique or mode to be used by the shader [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Create OpenGL shaders (vertex and fragment) and link them into `RayCastProgram` [G]. Generate textures for the volume data (`Volume1Index`) and the color lookup table (`ColorLookupIndex`) [G].
2. **Parameter Setup**: Populate `VolumeBBoxVertices` with the 8 corners of the input volume's bounding box [S1, unofficial]. Populate `VolumeBBoxVerticesColor` with corresponding colors if applicable [S1, unofficial].
3. **Matrix Calculation**: Compute `ParaMatrix` and `ParaMatrix1` based on the current camera view, projection, and volume bounding box [G]. These matrices define the ray origins and directions in texture space [G].
4. **Shader Uniforms**: Upload `ParaMatrix`, `ParaMatrix1`, `DistanceColorBlending`, `DepthPeelingThreshold`, `ICPEScale`, `Clipping`, and `Technique` to the shader program uniforms [S1, unofficial].
5. **Rendering**: Bind the volume texture and CLUT [S1, unofficial]. Draw a geometry that triggers the ray casting in the fragment shader [G]. The fragment shader samples the 3D texture along the rays defined by the matrices, applies the transfer functions, and composites the result [G].
6. **Post-Processing**: Apply depth peeling if enabled, using `DepthPeelingThreshold` to determine transparency handling [S1, unofficial].

**Formulas:**
Not given in the sources [G].

**Edge Cases and Pitfalls:**
- **Hardware Support**: Hardware rendering is unsupported if the input data has more than one independent component or if the hardware lacks required OpenGL extensions [S1, unofficial].

## Subtopics

### [VTK.js Volume Rendering with Transfer Functions](13344-vtk-js-volume-rendering-with-transfer-functions.md)
A volume rendering technique implemented in VTK.js that uses a volume actor, volume mapper, and transfer functions (color and opacity) to render 3D scalar field data (volumes) in a WebGL/WebGPU context. It provides a declarative JavaScript API for configuring piecewise transfer functions and gradient-based opacity modulation directly within the browser environment, building on [vtkSlicerGPURayCastVolumeMapper](7450-vtkslicergpuraycastvolumemapper.md) which handles the underlying GPU ray casting execution.

### vtkSlicerGPUVolumeMapper
An abstract superclass for GPU volume rendering using 3D texture mapping in the fragment shader [S1, unofficial]. Actual rendering work is delegated to `vtkSlicerGPURayCastVolumeTextureMapper3D` [S1, unofficial]. It supports any data type with one component, or two/four non-independent components [S1, unofficial]. Composite blending and intermixed opaque geometry are supported [S1, unofficial]. Multiple volumes can be rendered if sorted back-to-front [S1, unofficial]. Internal volume resolution is resampled to a power of two in each direction, capped at 128x256x256 voxels for 1 or 2 component data, or 128x128x256 voxels for 4 component data [S1, unofficial]. Resolution limits are hardcoded after a `GL_PROXY_TEXTURE3D` check because some graphics drivers falsely report support despite lacking texture memory [S1, unofficial]. It does not support more than one independent component or maximum intensity projection [S1, unofficial].

### GPU Volume Raycaster with KDE and Opacity Modulation
Renders the KDE-generated density volume interactively on the GPU using raycasting, integrating shading models and user-defined opacity emphasis ranges [S2, unofficial].
1. Create a 3D floating-point texture with normalized gradient in RGB channels and scaled density in the alpha channel [S2, unofficial].
2. For each sample along the ray, look up the density value in the 3D texture [S2, unofficial].
3. Calculate color by using the density value as an index into a color map [S2, unofficial].
4. Combine the color and normalized gradient with either Phong or Gooch shading model to compute the shaded color for the sample [S2, unofficial].
5. Compute alpha value α based on density ρ and emphasis range (ρmin, ρmax) with multipliers α1 (in-range) and α2 (out-of-range) [S2, unofficial]. Values inside the range are scaled by α1 [S2, unofficial]. Values outside are scaled by a linear interpolation of α1 and α2 [S2, unofficial].
Parameters: α1 (in-range opacity multiplier), α2 (out-of-range opacity multiplier), ρmin and ρmax (density range of emphasis) [S2, unofficial].
Opacity formula [S2, unofficial]:
$$
\alpha = \begin{cases} 
\rho \alpha_1 & \rho_{\min} < \rho < \rho_{\max} \\
\rho \left( \left(1 - \frac{\rho}{\rho_{\min}}\right) \alpha_2 + \frac{\rho}{\rho_{\min}} \alpha_1 \right) & \rho \le \rho_{\min} \\
\rho \left( \left(1 - \frac{\rho - \rho_{\max}}{1 - \rho_{\max}}\right) \alpha_1 + \frac{\rho - \rho_{\max}}{1 - \rho_{\max}} \alpha_2 \right) & \rho \ge \rho_{\max}
\end{cases}
$$

## Operating characteristics
- **Time Complexity**: Not given in the sources [G].
- **Memory Complexity**: Not given in the sources [G].
- **Performance**: Performance depends on hardware capabilities and the specific OpenGL extensions supported [G].
- **Strengths**: Provides specific controls for distance-based color blending and ICPE [S1, unofficial].
- **Limitations**: Does not support more than one independent component or maximum intensity projection [S1, unofficial]. Hardware rendering fails if required extensions are missing [S1, unofficial].
- **When to use**: Use for GPU-accelerated volume rendering of scalar fields with single or non-independent multi-component data where distance-based blending or ICPE is desired [S1, unofficial].
- **When not to use**: Do not use for data with multiple independent components or when maximum intensity projection is required [S1, unofficial].

## Background
It is designed to work within the CTK (Common Toolkit) and VTK ecosystem, leveraging Qt for the UI and VTK for data handling [S1, unofficial].

## Sources
[S1] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
[S2] Illustrative Rendering of Particle Systems, https://doi.org/10.2312/pe.vmv.vmv13.177-185, fetched 2026-10-26 (unofficial)
