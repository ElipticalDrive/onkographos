# Cross-section Texture Synthesis with Flow Vectors

Also called: Interactive system for designing and browsing volumetric illustrations, Internal Texture Synthesis, Volumetric Illustration Synthesis, Volumetric Illustration System

## Overview
This method synthesizes internal textures for 3D models by generating cross-sectional images on demand using 2D reference images, rather than storing explicit 3D volumetric data [S1]. It solves the problem of manually defining complex internal structures for volumetric illustrations by mapping pixels from a 2D control map (containing flow vectors) to a reference volume, ensuring the synthesized texture matches the local neighborhood structure and orientation defined by the flow [S1]. Unlike [Isotropic Texture Synthesis](13939-isotropic-texture-synthesis.md), which assumes uniform distribution without orientation dependency, this method supports layered and oriented textures by using flow vectors to guide the sampling geometry in the reference volume [S1].

## How it works

### Inputs and Outputs
**Inputs:**
1. A target 3D surface mesh model [S1].
2. A 2D reference image, optionally with a specific region selected by rubber banding [S1].
3. A texture type selection: Isotropic, Layered, or Oriented [S1].
4. Guidance information: None for Isotropic textures; a 2D control map (flow vectors) for Layered and Oriented textures [S1].
5. A 3D reference volume constructed from the reference image based on the texture type [S1].

**Outputs:**
* A synthesized 2D image representing the internal texture of a specific cross-section of the 3D model [S1].

### Data Structures

**Reference Volume:**
* A 3D data structure storing texture information derived from the 2D reference image [S1].
* For Isotropic textures: The reference image region is transferred to all faces of a reference cube [S1].
* For Layered textures: Associated with a vertical flow orientation [S1].
* For Oriented textures: The volume is sliced at appropriate angles to generate 2D reference images for each pixel [S1].

**Control Map (Flow Vectors):**
* A 2D array corresponding to the cross-section resolution (approximately 300x300 pixels) [S1].
* Each pixel contains a flow vector defining the local orientation and slant angle for texture synthesis [S1].

**Multi-resolution Pyramid:**
* Used for similarity comparison during synthesis [S1].
* For Isotropic and Oriented textures: Three levels [S1].
* Lower resolution: 3x3 square neighbor [S1].
* Higher resolution: 5x5 L-shaped neighbor [S1].
* For Layered textures: Lower resolution uses a 3x3 square neighbor and higher resolution uses a 5x3 rectangular neighbor [S1].

**Cache:**
* Stores sampled images from the reference volume [S1].
* Key: Discretized slant angle [S1].
* Value: Sampled 2D image from the reference volume [S1].

### Steps

1. Model Preparation: The user cuts the target 3D model using a freeform stroke on the screen, which opens with animation to reveal the cross-section [S1].
2. Region Selection: The cross-section is divided into closed regions, and the user clicks a target region to specify the 3D area to be textured [S1].
3. Texture Type Selection: The user selects one of three types: Isotropic, Layered, or Oriented [S1].
4. Reference Image Import: The user drags a 2D image file into the system and may select a specific region via rubber banding [S1].
5. Reference Volume Construction: For Isotropic textures, the selected region is transferred to all faces of a reference cube; for Layered and Oriented textures, the system constructs a 3D reference volume or prepares slicing parameters based on the flow orientation [S1].
6. Flow Vector Assignment (Layered/Oriented only): The user provides guidance information (flow vectors) for the cross-section, defining the layering direction or slice angle [S1].
7. Cross-section Synthesis: For each pixel in the target cross-section:
   a. Approximate the pixel's neighborhood as a small flat rectangle [S1].
   b. Determine the slant angle of this rectangle relative to the flow vector [S1].
   c. Sample the reference volume to find a corresponding neighborhood [S1].
   d. Compare similarity between the cross-section neighborhood and the sampled reference neighborhood [S1].
   e. Select the best matching pixel/region from the reference volume [S1].
   f. Assign the color to the cross-section pixel [S1].

### Formulas and Geometry

**Neighborhood Approximation:**
For a target pixel $p$ on the cross-section, the neighborhood $N_p$ is approximated as a small flat rectangle [S1].

**Slant Angle Matching:**
For a pixel $q$ in the reference volume, the sampled neighborhood $N_q$ is a small flat rectangle [S1]. The slant angle $\theta_q$ of $N_q$ (angle between the rectangle and the vertical flow vector) must equal the slant angle $\theta_p$ of $N_p$ on the cross-section [S1]:
$$ \theta_q = \theta_p $$

**Similarity Comparison:**
The system compares the similarity between $N_p$ and $N_q$ using multi-resolution pyramids [S1].

**Caching Key:**
The slant angle $\theta$ is discretized with a step size of $\Delta \theta = \pi/32$ [S1]. The cache key is $k = \lfloor \theta / \Delta \theta \rfloor$ [S1].

### Parameters

* Sampling Region Size: The slanted 2D square region sampled from the reference volume is 45x45 pixels [S1].
* Discretization Step: $\pi/32$ radians for slant angles [S1].
* Cross-section Resolution: Approximately 300x300 pixels [S1].
* Neighbor Sizes:
  * Isotropic/Oriented (Low): 3x3 square [S1].
  * Isotropic/Oriented (High): 5x5 L-shaped [S1].
  * Layered (Low): 3x3 square [S1].
  * Layered (High): 5x3 rectangular [S1].

### Pseudocode

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

# Constants
ISOTROPIC = 0
LAYERED = 1
ORIENTED = 2
SLANT_STEP = math.pi / 32
SAMPLE_SIZE = 45

def dot(v1, v2):
    return sum(a * b for a, b in zip(v1, v2))

def norm(v):
    return math.sqrt(dot(v, v))

def normalize(v):
    n = norm(v)
    return [x / n for x in v] if n > 0 else v

def empty_image(width, height):
    return [[0.0] * width for _ in range(height)]

def build_pyramid(image, texture_type):
    levels = []
    current = image
    for _ in range(3):
        levels.append(current)
        h, w = len(current), len(current[0])
        nh, nw = h // 2, w // 2
        next_img = [[0.0] * nw for _ in range(nh)]
        for y in range(nh):
            for x in range(nw):
                next_img[y][x] = current[2*y][2*x]
        current = next_img
    return levels

def get_neighbor_mask(texture_type, level):
    if texture_type in (ISOTROPIC, ORIENTED):
        return [(dy, dx) for dy in (-1, 0, 1) for dx in (-1, 0, 1)] if level == 0 else [(dy, dx) for dy in (-2, -1, 0, 1, 2) for dx in (-2, -1, 0, 1, 2) if abs(dy) + abs(dx) <= 3]
    else:
        return [(dy, dx) for dy in (-1, 0, 1) for dx in (-1, 0, 1)] if level == 0 else [(dy, dx) for dy in (-2, -1, 0, 1, 2) for dx in (-1, 0, 1)]

def compute_similarity(patch1, patch2, mask):
    diff = 0.0
    count = 0
    for dy, dx in mask:
        diff += abs(patch1[dy][dx] - patch2[dy][dx])
        count += 1
    return diff / count if count > 0 else float('inf')

def find_best_match(neighborhood, ref_image, texture_type):
    pyramids = [build_pyramid(neighborhood, texture_type), build_pyramid(ref_image, texture_type)]
    best_offset = (0, 0)
    best_score = float('inf')
    for level in range(3):
        mask = get_neighbor_mask(texture_type, level)
        h, w = len(pyramids[0][level]), len(pyramids[0][level][0])
        rh, rw = len(pyramids[1][level]), len(pyramids[1][level][0])
        for y in range(rh - h):
            for x in range(rw - w):
                patch = [[pyramids[1][level][y+dy][x+dx] for dx in range(w)] for dy in range(h)]
                score = compute_similarity(pyramids[0][level], patch, mask)
                if score < best_score:
                    best_score = score
                    best_offset = (y, x)
    return best_offset

def synthesize_cross_section(cross_section_pixels, control_map, reference_volume, texture_type, cache):
    synthesized_image = empty_image(cross_section_pixels[0][1], cross_section_pixels[0][0])
    for px, py in cross_section_pixels:
        flow_vector = control_map[py][px]
        slant_angle = math.acos(max(-1.0, min(1.0, dot(normalize(flow_vector), [0, 0, 1]))))
        discretized_angle = math.floor(slant_angle / SLANT_STEP) * SLANT_STEP
        if discretized_angle not in cache:
            sampled_ref = empty_image(SAMPLE_SIZE, SAMPLE_SIZE)
            for sy in range(SAMPLE_SIZE):
                for sx in range(SAMPLE_SIZE):
                    u, v = (sx - SAMPLE_SIZE/2) / SAMPLE_SIZE, (sy - SAMPLE_SIZE/2) / SAMPLE_SIZE
                    plane_normal = normalize([math.cos(slant_angle), math.sin(slant_angle), 0])
                    point_3d = [u, v, 0]
                    color = reference_volume.sample(point_3d)
                    sampled_ref[sy][sx] = color
            cache[discretized_angle] = sampled_ref
        sampled_ref = cache[discretized_angle]
        offset = find_best_match(neighborhood_cross, sampled_ref, texture_type)
        synthesized_image[py][px] = sampled_ref[offset[0]][offset[1]]
    return synthesized_image
```

### Edge Cases and Pitfalls

* Rotation Ambiguity: Rotation about the flow vector is ignored because the reference volume is assumed to have an isotropic structure on cross-sections perpendicular to the flow orientation [S1].
* Cascading Resampling: Quality issues arise due to cascading resampling and resulting distortion [S1].
* Performance: Layered and oriented textures require more computation than standard texture synthesis because layered textures require rotating neighbors and oriented textures require generating a 2D reference image by slicing the reference volume at an appropriate angle for each pixel [S1].
* Boundary Conditions: The sampling region is 45×45 pixels, which is chosen to fit within the reference volume completely [S1].

## Operating characteristics

* Time Complexity: Not explicitly given in Big-O notation [S1]. Synthesis times vary by model and texture type [S1]. On a Pentium M 1.6-GHz processor with 1 GB RAM: Meat: 22 sec, Cucumber: 4 sec, Bamboo: 14 sec, Stomach: 18 sec, Tooth: 32 sec [S1].
* Memory Complexity: The data size is significantly smaller than explicit volumetric data [S1]. For a 300x300 cross-section, the quality is comparable to 300^3 colored voxels (approx. 80 MB storage), but the actual model data is much smaller (e.g., Meat: 622 kb, Cucumber: 53 kb, Bamboo: 291 kb, Stomach: 402 kb, Tooth: 307 kb) [S1].
* Strengths: Significantly reduces data storage requirements compared to explicit 3D volumetric data [S1]. Allows designers to add 3D internal textures without specifying each voxel manually [S1]. Supports interactive modeling with progressive synthesis [S1].
* Limitations: Computational cost is higher for layered and oriented textures compared to isotropic [S1]. Quality issues due to cascading resampling and distortion [S1]. Cannot be used for applications requiring real volumetric data, such as translucent rendering or volumetric simulation [S1].
* When to Use: Suitable for creating lightweight volumetric illustrations for communication purposes (e.g., education, medical visualization) where explicit volumetric data is not required [S1].
* When Not to Use: Not suitable for applications requiring accurate volumetric properties like translucency or simulation [S1].

## Background

* Origin: Proposed in 2004 by the authors of "Volumetric Illustration" [S1].
* History: The system builds on previous work in texture synthesis (Heeger and Bergen 1995, Wei 2001) and 3D texture mapping (Turk 2001, Praun et al. 2000) [S1]. It addresses the difficulty of designing detailed internal textures of 3D volumes using standard 2D devices [S1].
* Applications: Useful for conveying volumetric information between teachers and students, doctors and patients, or virtual-reality content providers and consumers [S1].

## Sources

[S1] Volumetric illustration, https://doi.org/10.1145/1186562.1015723, fetched 2026-10-30
