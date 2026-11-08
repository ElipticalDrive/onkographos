# Texture-Based Volume Rendering

Also called: Geometry-based volume visualization, Proxy Geometry Rendering, Slice-based rendering, TBVR, Texture Based Ray Casting

## Overview
Texture-Based Volume Rendering (TBVR) is a technique that renders volumetric data by mapping it to 2D proxy geometry (polygons) and rasterizing them using the graphics pipeline's texture hardware [S1, S5, S6, unofficial]. It solves the problem of efficiently sampling and compositing 3D scalar fields on GPUs by leveraging hardware-accelerated texture lookups and alpha blending [S2, book][S8]. Unlike direct ray casting which computes samples per pixel, TBVR computes samples per fragment of a slice polygon [S2, book]. It is distinguished from [Image-order rendering](52-name.md) by its object-order nature (rendering geometry rather than pixels) and from [Surface Rendering Pipeline](2614-surface-rendering-pipeline.md) by its ability to render translucent internal structures rather than just opaque surfaces [S1, S5, unofficial].

## How it works
### Inputs and Outputs
**Inputs:**
*   Volumetric data: A 3D scalar field stored as a 3D texture or a stack of 2D textures [S1, S5, S6, unofficial].
*   Transfer function: A 1D texture mapping scalar values to color and opacity [S2, book][S8].
*   Camera parameters: View and projection matrices [S4, unofficial].
*   Rendering parameters: Number of slices, clipping planes, lighting constants [S4, unofficial].

**Outputs:**
*   Framebuffer image with composited volume data [S1, unofficial][S8].

### Data Structures
**Volume Texture:**
*   **3D Texture:** Single texture object containing the entire volume. Format: `RedFormat` (single channel), Type: `UnsignedByteType` (or float), Filter: `LinearFilter` (trilinear interpolation) [S4, unofficial].
*   **2D Texture Stack:** Array of 2D textures. For object-aligned slices, data is replicated in three principal directions (x, y, z), requiring 3x memory [S1, S5, unofficial].

**Proxy Geometry:**
*   **Vertices:** Defined by the intersection of slicing planes with the volume bounding box [S1, S5, unofficial].
    *   Example Bounding Box: 8 vertices, e.g., $(\pm 80, \pm 128, \pm 110.5)$ [S4, unofficial].
*   **Texture Coordinates:** 3D coordinates $(u, v, w)$ assigned to each vertex, mapping the polygon to the volume space [S1, unofficial][S8].
    *   Normalized range: $[0, 1]$ for each axis [S4, unofficial].
    *   Computed as: $u = \frac{x}{W} + 0.5$, $v = \frac{y}{H} + 0.5$, $w = \frac{z}{D} + 0.5$ where $W,H,D$ are texture dimensions [S4, unofficial].

**Transfer Function:**
*   1D texture mapping scalar intensity to RGBA [S2, book][S8].

### Numbered Steps
1.  **Initialize:**
    *   Load volumetric data into CPU memory [S2, book][S8].
    *   Process data if necessary (e.g., compute gradients, downsample) [S2, book][S8].
    *   Create transfer function lookup table (1D texture) [S2, book][S8].
    *   Download volume data to GPU as 3D texture or 2D texture stack [S1, S4, S7, unofficial].
    *   Create fragment shaders for compositing and optional illumination [S2, book][S7, unofficial].

2.  **Update (on view/parameter change):**
    *   Compute proxy geometry: Intersect slicing planes with the volume bounding box [S1, S5, unofficial][S8].
    *   Sort vertices in counter-clockwise order around the polygon centroid [S5, unofficial][S8].
    *   Construct polygon primitives (e.g., triangle fan from centroid) [S5, unofficial].
    *   Assign 3D texture coordinates to each vertex [S1, unofficial][S8].
    *   If using 2D texture stacks, select the appropriate stack based on the viewing direction (closest principal axis) [S1, S5, unofficial].
    *   Refresh textures if rendering mode or transfer function parameters change [S2, book][S8].

3.  **Draw:**
    *   Set up rendering state:
        *   Disable lighting and culling [S2, book][S8].
        *   Enable alpha blending: Source = `GL_ONE`, Destination = `GL_ONE_MINUS_SRC_ALPHA` (for Over operator) [S1, unofficial][S2, book][S8].
        *   Disable depth testing or disable depth buffer writing if blending with opaque geometry [S8].
    *   Bind volume texture and transfer function texture [S2, book][S8].
    *   Draw slice polygons in **back-to-front** order (for Over operator) or front-to-back (for Under operator) [S1, unofficial][S2, book][S8].
    *   Restore rendering state [S2, book][S8].

### Formulas
**Volume Rendering Integral (Discrete Approximation):**
The color $C$ and opacity $\alpha$ are accumulated along the ray segment corresponding to the slice thickness. For the Over operator:
$$ C_{out} = C_{in} + (1 - \alpha_{in}) \cdot C_{sample} \cdot \alpha_{sample} $$
$$ \alpha_{out} = \alpha_{in} + (1 - \alpha_{in}) \cdot \alpha_{sample} $$
Where $C_{sample}$ and $\alpha_{sample}$ are obtained from the transfer function lookup using the sampled scalar value [S2, book][S8].

**Hardware Blending (Over Operator):**
$$ Source = 1 $$
$$ Destination = 1 - Source\ Alpha $$
[S1, unofficial]

**Under Operator:**
$$ Source = 1 - Destination\ Alpha $$
$$ Destination = 1 $$
[S1, unofficial]

**Illumination (Blinn-Phong):**
If illumination is applied in the fragment shader:
$$ Color = Ambient + Diffuse + Specular $$
$$ Ambient = K_a \cdot L_{ambient} $$
$$ Diffuse = K_d \cdot L_{color} \cdot \max(\vec{N} \cdot \vec{L}, 0) $$
$$ Specular = K_s \cdot L_{color} \cdot \max(\vec{H} \cdot \vec{N}, 0)^{shininess} $$
Where $\vec{H} = \frac{\vec{L} + \vec{V}}{||\vec{L} + \vec{V}||}$ [S4, unofficial].

### Parameters
*   **PLANE_INC:** Increment value for plane slicing, typically $1.0$ [S4, unofficial].
*   **u_slices:** Uniform `vec3` containing texture dimensions $(TEXTURE\_SIZE\_X, TEXTURE\_SIZE\_Y, TEXTURE\_SIZE\_Z)$ used to normalize coordinates [S4, unofficial].
*   **u_thresh:** Uniform `float` threshold for discarding fragments based on texture value [S4, unofficial].
*   **u_alphaScale:** Uniform `float` multiplier for fragment alpha in transparent mode [S4, unofficial].
*   **Clipping Planes:** `u_clipPlaneX/Y/Z` (float) and `u_clipCubeMax/Min` (vec3) for spatial culling [S4, unofficial].
*   **Shading Constants:** $K_a=(0.1,0.1,0.1)$, $K_d=(0.6,0.6,0.6)$, $K_s=(0.2,0.2,0.2)$, shininess=$100.0$, ambientLight=$(0.3,0.3,0.3)$, lightColour=$(1.0,1.0,1.0)$ [S4, unofficial].
*   **Light Positions:** $L_1=(100.0, -600.0, 100.0)$, $L_2=(0.0, 300.0, 0.0)$ [S4, unofficial].

### Edge Cases and Pitfalls
*   **Object-Aligned Slices Artifacts:** Using 2D texture stacks requires replicating data in three directions, increasing memory by 3x [S1, S5, unofficial]. Switching between stacks causes "image popping" artifacts and lag during texture download [S1, S5, unofficial].
*   **Sampling Distance Variation:** In object-aligned slices, the sampling distance changes with the viewpoint, causing intensity variations [S1, S5, unofficial].
*   **Bilinear Interpolation Artifacts:** 2D textures use bilinear interpolation, causing artifacts at the edges of each slice [S7, unofficial]. 3D textures use trilinear interpolation, which is smoother but may have slower access speeds [S7, unofficial].
*   **Coordinate System Rotation:** If the model is rotated (e.g., y-axis to z-axis), clipping plane logic must be adjusted accordingly (e.g., `clipPlaneY` checks `texPosition.z`) [S4, unofficial].
*   **Sampling Rate Dependency:** The rendering quality depends on the sampling rate (distance between samples). Artifacts can be avoided by using extra slices and trilinear filtering [S2, book][S6, unofficial].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
# Vertex Shader
# Input: position (vec3), modelMatrix (mat4), projectionMatrix (mat4), modelViewMatrix (mat4)
# Output: gl_Position (vec4), texPosition (vec4), vCam (vec3)

varying vec4 texPosition;
varying vec3 vCam;

void main() {
    vCam = cameraPosition;
    texPosition = modelMatrix * vec4(position, 1.0);
    gl_Position = projectionMatrix * modelViewMatrix * vec4(position, 1.0);
}

# Fragment Shader
# Input: texPosition (vec4), u_data (sampler3D), u_slices (vec3), u_thresh (float), u_alphaScale (float)
# Output: gl_FragColor (vec4)

uniform sampler3D u_data;
uniform vec3 u_slices;
uniform float u_thresh;
uniform float u_alphaScale;

void main() {
    # Compute normalized texture coordinates
    vec3 texCoords = vec3(
        (texPosition.x / u_slices.x) + 0.5,
        (texPosition.y / u_slices.y) + 0.5,
        (texPosition.z / u_slices.z) + 0.5
    );

    # Sample volume data
    vec4 color = texture(u_data, texCoords);

    # Discard if below threshold
    if (color.r < u_thresh) {
        discard;
    }

    # Set alpha and color
    # In simple mode, color is grayscale, alpha is scaled
    color.b = color.g = color.r;
    color.a *= u_alphaScale;

    gl_FragColor = color;
}

# Optional: Illumination Fragment Shader
# Adds Blinn-Phong shading using gradients computed from volume data

uniform sampler3D u_data;
uniform vec3 u_slices;
uniform float u_thresh;
uniform float u_alphaScale;
uniform vec3 lightPos;
uniform vec3 lightPos2;
uniform vec3 cameraPosition;

varying vec4 texPosition;
varying vec3 vCam;

void main() {
    vec3 texCoords = vec3(
        (texPosition.x / u_slices.x) + 0.5,
        (texPosition.y / u_slices.y) + 0.5,
        (texPosition.z / u_slices.z) + 0.5
    );

    vec4 color = texture(u_data, texCoords);
    if (color.r < u_thresh) {
        discard;
    }

    # Compute normal via finite differences
    float delta = 1.0 / 256.0;
    vec3 sample1, sample2;
    
    sample1.x = texture(u_data, texCoords - vec3(delta, 0.0, 0.0)).x;
    sample2.x = texture(u_data, texCoords + vec3(delta, 0.0, 0.0)).x;
    vec3 N_x = normalize(vec3(sample2.x - sample1.x, 0.0, 0.0)); # Simplified, actual normal needs all components
    
    # More accurate normal computation:
    float dx = texture(u_data, texCoords + vec3(delta, 0.0, 0.0)).x - texture(u_data, texCoords - vec3(delta, 0.0, 0.0)).x;
    float dy = texture(u_data, texCoords + vec3(0.0, delta, 0.0)).x - texture(u_data, texCoords - vec3(0.0, delta, 0.0)).x;
    float dz = texture(u_data, texCoords + vec3(0.0, 0.0, delta)).x - texture(u_data, texCoords - vec3(0.0, 0.0, delta)).x;
    vec3 N = normalize(vec3(dx, dy, dz));

    vec3 L = normalize(lightPos - texPosition.xyz);
    vec3 V = normalize(cameraPosition - texPosition.xyz);
    vec3 H = normalize(L + V);

    # Blinn-Phong constants
    vec3 Ka = vec3(0.1, 0.1, 0.1);
    vec3 Kd = vec3(0.6, 0.6, 0.6);
    vec3 Ks = vec3(0.2, 0.2, 0.2);
    float shininess = 100.0;
    vec3 lightColour = vec3(1.0, 1.0, 1.0);
    vec3 ambientLight = vec3(0.3, 0.3, 0.3);

    vec3 ambient = Ka * ambientLight;
    float diffuseLight = max(dot(L, N), 0.0);
    vec3 diffuse = Kd * lightColour * diffuseLight;
    float specularLight = pow(max(dot(H, N), 0.0), shininess);
    if (diffuseLight <= 0.0) { specularLight = 0.0; }
    vec3 specular = Ks * lightColour * specularLight;

    vec3 shade1 = ambient + diffuse + specular;
    
    # Second light (simplified)
    vec3 L2 = normalize(lightPos2 - texPosition.xyz);
    vec3 H2 = normalize(L2 + V);
    float diffuseLight2 = max(dot(L2, N), 0.0);
    vec3 diffuse2 = Kd * lightColour * diffuseLight2;
    float specularLight2 = pow(max(dot(H2, N), 0.0), shininess);
    if (diffuseLight2 <= 0.0) { specularLight2 = 0.0; }
    vec3 specular2 = Ks * lightColour * specularLight2;
    vec3 shade2 = ambient + diffuse2 + specular2;

    color.rgb = shade1 + shade2;
    color.a *= u_alphaScale;
    color.b = color.g = color.r;

    gl_FragColor = color;
}
```

## Subtopics

### Object-Aligned Slices
Uses three stacks of 2D textures aligned with principal axes. Fast to render but suffers from 3x memory usage, popping artifacts, and intensity variations. Choose when memory is abundant and view changes are minimal [S1, S5, unofficial].

### View-Aligned Slices
Uses polygons orthogonal to the viewing direction. More memory efficient and consistent sampling distance, but slower due to dynamic geometry generation. Choose for interactive rotation [S1, S5, unofficial].

### 3D Texture Rendering
Uses a single 3D texture with view-aligned slices. Avoids data replication and bilinear interpolation artifacts. Choose for high-quality rendering on modern GPUs [S2, book][S7, unofficial].

## Operating characteristics
*   **Time Complexity:** $O(N \cdot M)$ where $N$ is the number of slices and $M$ is the number of fragments per slice. Rendering time is dominated by rasterization and texture lookups [S2, book][S6, unofficial].
*   **Memory Complexity:**
    *   3D Texture: $O(V)$ where $V$ is the volume size [S2, book].
    *   2D Texture Stack (Object-Aligned): $O(3V)$ due to data replication [S1, S5, unofficial].
*   **Measured Performance:** Fast and simple for object-aligned slices, but suffers from lag during texture downloads when switching stacks [S1, S5, unofficial]. View-aligned slices are slower but more memory efficient [S1, unofficial].
*   **Strengths:** Leverages hardware acceleration for texture sampling and blending; simple implementation; good for static views or slow rotations [S1, S5, unofficial].
*   **Limitations:** Artifacts from sampling distance changes (object-aligned); popping artifacts; memory overhead for 2D stacks; dependent texture reads can be slow on older hardware [S1, unofficial][S2, book][S5, S7, unofficial].
*   **When to Use:** Interactive visualization where view changes are slow or limited; hardware with good 3D texture support [S1, unofficial][S2, book].
*   **When Not to Use:** High-speed rotation with object-aligned slices; scenes requiring precise sampling control without artifacts [S1, S5, unofficial].

## Background
Texture-Based Volume Rendering was developed in the early 2000s to leverage the emerging 3D texture capabilities of consumer GPUs [S1, unofficial][S2, book]. Key papers include Krüger and Westermann (2003) on acceleration techniques and the NVIDIA GPU Gems Chapter 39 [S1, unofficial][S2, book]. It evolved from 2D texture slice-based methods to 3D texture-based methods as hardware improved [S6, S7, unofficial]. It is widely used in medical visualization and scientific data analysis [S4, unofficial].

## Sources
[S1] GPU volume rendering, http://www.slideserve.com/hilliard/volume-rendering-using-graphics-hardware, fetched 2026-10-26 (unofficial)
[S2] Chapter 39. Volume Rendering Techniques | NVIDIA Developer, https://developer.nvidia.com/gpugems/gpugems/part-vi-beyond-triangles/chapter-39-volume-rendering-techniques, fetched 2026-10-26 (book)
[S3] Volume Visualization and Compositing on Large-Scale Displays Using Handheld Touchscreen Interaction, https://doi.org/10.25781/kaust-d5m7l, fetched 2026-10-26 (unofficial)
[S4] DoctorTone/MedicalViz, https://github.com/DoctorTone/MedicalViz, fetched 2026-10-27 (unofficial)
[S5] PDF CS 519: Scientific Visualization - University of Illinois Urbana-Champaign, https://courses.grainger.illinois.edu/cs519/fa2017/Lecture%2012%20-%20Volume%20Visualization.pdf, fetched 2026-10-27 (unofficial)
[S6] PDF A Survey of GPU-Based Volume Rendering of Unstructured Grids, https://www.sci.utah.edu/~stevec/papers/gputet_survey.pdf, fetched 2026-10-27 (unofficial)
[S7] Volume Rendering Lecture Nov-30-2005 (Spanish), https://www.ciroduran.com/personal2/wp-content/uploads/2009/07/clase-volumerendering.pdf, fetched 2026-10-29 (unofficial)
[S8] GPU Gems ch39, Volume rendering techniques, https://developer.nvidia.com/sites/all/modules/custom/gpugems/books/GPUGems/gpugems_ch39.html, fetched 2026-10-29
