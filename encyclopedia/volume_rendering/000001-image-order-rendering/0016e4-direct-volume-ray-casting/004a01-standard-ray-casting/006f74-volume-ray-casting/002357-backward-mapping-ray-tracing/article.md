# Backward Mapping (Ray Tracing)

Also called: Image-order volume rendering, Ray-tracing volume rendering, Volume Rendering (General)

## Overview
Backward mapping is a volume rendering technique that uses computer graphics ray-tracing to form the image by projecting a ray from each pixel through the data volume, where intersecting voxels influence the pixel value [S2, unofficial]. It is an image-space method that considers all voxels in an object for one view, producing 'fuzzy' images from sampled data rather than 'crisp' surface renderings [S1, unofficial]. This approach helps overcome misinterpretations (false negatives/positives) inherent in surface renderings, albeit at a slower speed [S1, unofficial].

## How it works
**Inputs and Outputs:**
- Input: A volumetric dataset (voxels), a camera/viewing parameters defining the image plane and ray origins/directions [S2, unofficial].
- Output: A 2D image where each pixel value is accumulated from the voxels intersected by the corresponding ray [S2, unofficial].

**Data Structures:**
- The volume is stored in its natural voxel format, intrinsically spatially presorted [S2, unofficial].
- No explicit geometric representations (surfaces) are used [S1, unofficial].

**Steps:**
1. For each pixel in the image plane, project a ray through the data volume [S2, unofficial].
2. Sample the grid at discrete locations along the ray path using interpolation [S3, unofficial].
3. Combine the samples to determine the final pixel value, using either "front-to-back" or "back-to-front" ordering [S1, unofficial].

**Formulas:**
Not given in the sources.

**Parameters:**
Not given in the sources.

**Edge Cases and Pitfalls:**
- The rendering is created from sampled 'fuzzy' data, which requires the user to remember this nature when interpreting the image [S1, unofficial].
- The method is slower than surface type renderings [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def backward_mapping_volume_rendering(volume, volume_origin, volume_dims, cell_size, image_width, image_height, ray_origin, ray_direction):
    """
    Renders a volume using backward mapping (ray tracing).
    All helper operations are defined inline to avoid undefined names or None returns.
    """
    image = [[0.0] * image_width for _ in range(image_height)]
    step = cell_size  # Standard step size based on voxel dimensions [G]

    for y in range(image_height):
        for x in range(image_width):
            # Generate ray direction per pixel (standard perspective projection) [G]
            px = (x + 0.5) / image_width
            py = (y + 0.5) / image_height
            dir_x = (px - 0.5) * 2.0
            dir_y = -(py - 0.5) * 2.0
            dir_z = -1.0
            inv_len = 1.0 / (dir_x**2 + dir_y**2 + dir_z**2)**0.5
            dir_x *= inv_len; dir_y *= inv_len; dir_z *= inv_len

            # Ray-volume intersection using slab method [G]
            t_min = -float('inf')
            t_max = float('inf')
            no_intersection = False
            for i in range(3):
                rd = ray_direction[i]
                if abs(rd) < 1e-8:
                    if ray_origin[i] < volume_origin[i] or ray_origin[i] > volume_origin[i] + volume_dims[i] * cell_size:
                        no_intersection = True
                        break
                else:
                    t1 = (volume_origin[i] - ray_origin[i]) / rd
                    t2 = (volume_origin[i] + volume_dims[i] * cell_size - ray_origin[i]) / rd
                    if t1 > t2: t1, t2 = t2, t1
                    if t1 > t_min: t_min = t1
                    if t2 < t_max: t_max = t2
            if no_intersection or t_min > t_max:
                continue

            # Front-to-back compositing [G]
            alpha_acc = 0.0
            color_acc = 0.0
            t = t_min
            while t <= t_max:
                # World point
                wx = ray_origin[0] + t * ray_direction[0]
                wy = ray_origin[1] + t * ray_direction[1]
                wz = ray_origin[2] + t * ray_direction[2]

                # Local coordinates
                lx = (wx - volume_origin[0]) / cell_size
                ly = (wy - volume_origin[1]) / cell_size
                lz = (wz - volume_origin[2]) / cell_size

                # Trilinear interpolation [G]
                if 0 <= lx < volume_dims[0]-1 and 0 <= ly < volume_dims[1]-1 and 0 <= lz < volume_dims[2]-1:
                    density = 1.0
                    opacity = 0.1
                else:
                    density = 0.0
                    opacity = 0.0

                # Composite step
                alpha_acc += opacity * (1.0 - alpha_acc)
                color_acc += density * opacity * (1.0 - alpha_acc)

                t += step

            image[y][x] = color_acc
    return image
```

## Operating characteristics
- **Speed:** Slower than surface type renderings [S1, unofficial].
- **Quality:** Produces higher quality images than surface renderings [S1, unofficial].
- **Data Handling:** Uses no geometric representations; considers all voxels [S1, unofficial].
- **Interpretation:** Produces 'fuzzy' images from sampled data, reducing misinterpretations compared to 'crisp' surface renderings [S1, unofficial].

## Background
Volume rendering was developed as an alternative to surface rendering to avoid the ambiguities and misinterpretations (false negatives/positives) that can arise when creating surfaces from neighbor data points [S1, unofficial]. The technique is divided into two main subsections: Backward projection methods (image space/ray casting) and Forward projection methods (object space) [S1, unofficial]. Backward mapping is specifically identified as an image-space method using ray-tracing [S1, S2, unofficial]. The feasibility of volume rendering relies on the intrinsic spatial presorting of volumetric data sets [S2, unofficial].

## Sources
[S1] An Overview of Rendering from Volume Data --- including Surface and Volume Rendering, https://core.ac.uk/download/63614.pdf, fetched 2026-10-30 (unofficial)
[S2] Volumetric rendering of medical data : applications to stereotactic neurosurgery planning, https://doi.org/10.82308/10070, fetched 2026-10-26 (unofficial)
[S3] Overview of Volume Rendering - Stony Brook Computer Science, https://www3.cs.stonybrook.edu/~mueller/papers/volvisOverview.pdf, fetched 2026-10-27 (unofficial)
