# vtkSlicerGPURayCastVolumeMapper

Also called: GPGPU Ray Cast Mapper

## Overview
Renders a volume using ray casting on a 3D texture [S1, unofficial]. It sets itself apart by providing specific parameters for distance-based color blending and ICPE (Intensity Contrast Preserving Enhancement) scaling, building on [GPU Multi-Volume Ray Casting Mapper](2461-gpu-multi-volume-ray-casting-mapper.md) which manages the general GPU-accelerated pipeline.

## How it works
**Inputs and Outputs:**
- Input: 3D scalar field data (volume), transfer functions (color and opacity), camera parameters, and bounding box definition.
- Output: 2D image with depth information.

**Data Structures:**
The mapper maintains the following internal state and OpenGL handles [S1, unofficial]:
- `double VolumeBBoxVertices[8][3]`: Stores the 8 vertices of the volume's bounding box, each with 3 coordinates (x, y, z).
- `double VolumeBBoxVerticesColor[8][3]`: Stores the 8 vertices of the bounding box with associated color values, each with 3 coordinates.
- `GLfloat ParaMatrix[16]`: A 4x4 matrix uniform passed to the shader for ray casting parameters.
- `GLfloat ParaMatrix1[16]`: A second 4x4 matrix uniform for additional ray casting parameters.
- `GLuint Volume1Index`: OpenGL handle for the volume texture.
- `GLuint ColorLookupIndex`: OpenGL handle for the color lookup table (CLUT).
- `GLuint RayCastVertexShader`: OpenGL handle for the vertex shader.
- `GLuint RayCastFragmentShader`: OpenGL handle for the fragment shader.
- `GLuint RayCastProgram`: OpenGL handle for the shader program linking the vertex and fragment shaders.

**Parameters:**
- `DistanceColorBlending` (float, default 0.0): Controls the brightness falloff with distance. A value of 0.0 implies no blending; positive values make closer voxels brighter and farther voxels darker [S1, unofficial].
- `DepthPeelingThreshold` (float): Threshold value used for depth peeling operations to handle transparency ordering [S1, unofficial].
- `ICPEScale` (float): Scale parameter for Intensity Contrast Preserving Enhancement (ICPE), used to adjust contrast during rendering [S1, unofficial].
- `Clipping` (int): Boolean flag (0 or 1) to enable or disable clipping planes [S1, unofficial].
- `Technique` (int): Integer selector for the specific rendering technique or mode to be used by the shader [S1, unofficial].

**Algorithm Steps:**
1. **Initialization**: Create OpenGL shaders (vertex and fragment) and link them into `RayCastProgram`. Generate textures for the volume data (`Volume1Index`) and the color lookup table (`ColorLookupIndex`).
2. **Parameter Setup**: Populate `VolumeBBoxVertices` with the 8 corners of the input volume's bounding box. Populate `VolumeBBoxVerticesColor` with corresponding colors if applicable.
3. **Matrix Calculation**: Compute `ParaMatrix` and `ParaMatrix1` based on the current camera view, projection, and volume bounding box. These matrices define the ray origins and directions in texture space.
4. **Shader Uniforms**: Upload `ParaMatrix`, `ParaMatrix1`, `DistanceColorBlending`, `DepthPeelingThreshold`, `ICPEScale`, `Clipping`, and `Technique` to the shader program uniforms.
5. **Rendering**: Bind the volume texture and CLUT. Draw a geometry (typically a quad or the bounding box) that triggers the ray casting in the fragment shader. The fragment shader samples the 3D texture along the rays defined by the matrices, applies the transfer functions, and composites the result.
6. **Post-Processing**: Apply depth peeling if enabled, using `DepthPeelingThreshold` to determine transparency handling.

**Edge Cases and Pitfalls:**
- **Hardware Support**: Hardware rendering is unsupported if the input data has more than one independent component or if the hardware lacks required OpenGL extensions [S1, unofficial].
- **Precision**: Calculations are performed using 8 bits per RGBA channel. 15-bit float compositing is not supported [S1, unofficial].

## Subtopics

### vtkSlicerGPUVolumeMapper
An abstract superclass for GPU volume rendering using 3D texture mapping in the fragment shader. Actual rendering work is delegated to `vtkSlicerGPURayCastVolumeTextureMapper3D` [S1, unofficial]. It supports any data type with one component, or two/four non-independent components. Composite blending and intermixed opaque geometry are supported. Multiple volumes can be rendered if sorted back-to-front [S1, unofficial]. Internal volume resolution is resampled to a power of two in each direction, capped at 128x256x256 voxels for 1 or 2 component data, or 128x128x256 voxels for 4 component data [S1, unofficial]. Resolution limits are hardcoded after a `GL_PROXY_TEXTURE3D` check because some graphics drivers falsely report support despite lacking texture memory [S1, unofficial]. It does not support more than one independent component or maximum intensity projection [S1, unofficial].

## Operating characteristics
- **Time Complexity**: Not given in the sources.
- **Memory Complexity**: Not given in the sources.
- **Performance**: Performance depends on hardware capabilities and the specific OpenGL extensions supported. Resolution is capped at 128x256x256 (1/2 component) or 128x128x256 (4 component) due to driver limitations [S1, unofficial].
- **Strengths**: Supports multiple volumes, composite blending, and intermixed opaque geometry. Provides specific controls for distance-based color blending and ICPE [S1, unofficial].
- **Limitations**: Does not support more than one independent component or maximum intensity projection. Hardware rendering fails if required extensions are missing [S1, unofficial].
- **When to use**: Use for GPU-accelerated volume rendering of scalar fields with single or non-independent multi-component data where distance-based blending or ICPE is desired.
- **When not to use**: Do not use for data with multiple independent components or when maximum intensity projection is required.

## Background
Origin: Implemented as part of the VolumeRenderer project by jszgz [S1, unofficial]. It is designed to work within the CTK (Common Toolkit) and VTK ecosystem, leveraging Qt for the UI and VTK for data handling.

## Sources
[S1] jszgz/VolumeRenderer, https://github.com/jszgz/VolumeRenderer, fetched 2026-10-27 (unofficial)
