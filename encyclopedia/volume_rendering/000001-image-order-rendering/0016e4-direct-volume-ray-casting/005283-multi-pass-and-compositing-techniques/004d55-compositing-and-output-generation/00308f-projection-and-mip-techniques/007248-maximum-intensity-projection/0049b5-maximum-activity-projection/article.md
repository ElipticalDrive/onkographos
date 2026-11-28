# Maximum Activity Projection

Also called: Max Activity Projection, Max projection, Max0.0, Max0.024, Max0.049, Maximal Voxel Projection

## Overview
Maximum Activity Projection (MAP) is a volume rendering technique for "hot spot" imaging in nuclear medicine that enhances the continuity of structures compared to slice display by incorporating depth information and spatial relationships into a 2D image [S1, unofficial]. It sets itself apart from its sibling [Maximum Intensity Projection](902-maximum-intensity-projection.md) by explicitly applying simulated exponential attenuation along the viewing ray before selecting the maximum voxel, thereby weighting voxels by their depth and preventing the "illusion of reversal of rotation" during cine display [S1, unofficial]. Unlike [Summed Projection](902-maximum-intensity-projection.md) which accumulates background activity and reduces contrast, MAP preserves the high contrast of tomographic slices for areas of increased activity while reducing image noise [S1, unofficial].

## How it works
**Inputs and Outputs:**
*   **Input:** A 3D volume data cube $D$ of dimensions $N_x \times N_y \times N_z$ (typically $64 \times 64 \times 64$ in clinical SPECT implementations) containing voxel activity values [S1, unofficial]. An attenuation coefficient $\alpha$ (in cm$^{-1}$) [S1, unofficial]. A viewing direction defined by a set of angles (e.g., 64 angles over 360 degrees for cine display) [S1, unofficial].
*   **Output:** A 2D rendered image $I$ of dimensions $N_x \times N_y$ (matching the transaxial slice dimensions) [S1, unofficial].

**Data Structures:**
*   $D$: A 3D array of floating-point or integer values representing tracer uptake [S1, unofficial].
*   $I$: A 2D array of floating-point values representing the projected intensity [S1, unofficial].
*   $\alpha$: A scalar float representing the attenuation coefficient [S1, unofficial].

**Parameters:**
*   $\alpha$: The attenuation coefficient. Typical values used in literature are:
    *   $0$ cm$^{-1}$ (No attenuation, referred to as **Max0.0**) [S1, unofficial].
    *   $0.024$ cm$^{-1}$ (**Max0.024**) [S1, unofficial].
    *   $0.049$ cm$^{-1}$ (**Max0.049**) [S1, unofficial].
    *   An attenuation coefficient of $0.024$ cm$^{-1}$ is used for low background settings like bone scintigraphy [S1, unofficial].

**Algorithm Steps:**
1.  **Stack Slices:** Construct the 3D volume cube $D$ by stacking the transaxial slices obtained from filtered backprojection of the raw projection data [S1, unofficial].
2.  **Define Rays:** For each pixel $(x, y)$ in the output image $I$, define a ray passing through the volume $D$ along the viewing direction [S1, unofficial]. In the standard implementation described, rays are cast perpendicular to the transaxial slices (i.e., along the z-axis for a posterior view), iterating through $z = 1$ to $N_z$ [S1, unofficial].
3.  **Apply Attenuation:** For each voxel $D_{x,y,z}$ along the ray, calculate the depth-weighted value [S1, unofficial]. The depth $z$ is measured from the front surface of the object or the start of the ray [S1, unofficial]. The attenuated value is computed as $D_{x,y,z} \cdot e^{-\alpha z}$ [S1, unofficial].
4.  **Select Maximum:** Identify the voxel with the highest attenuated value along the ray [S1, unofficial].
5.  **Assign Output:** Assign this maximum value to the corresponding pixel $I_{x,y}$ in the rendered image [S1, unofficial].

**Formula:**
The core rendering equation is [S1, unofficial]:

$$ I_{x,y} = \max_{z} (D_{x,y,z} \cdot e^{-\alpha z}) $$

Where:
*   $I_{x,y}$ is the intensity of the pixel at coordinates $(x,y)$ in the rendered image [S1, unofficial].
*   $D_{x,y,z}$ is the activity value of the voxel at coordinates $(x,y,z)$ in the volume [S1, unofficial].
*   $\alpha$ is the attenuation coefficient [S1, unofficial].
*   $z$ is the depth along the ray [S1, unofficial].
*   $\max_{z}$ is the maximum operator selecting the greatest value along the ray [S1, unofficial].

**Noise Accumulation Factor (NF):**
The maximum operator selects noise peaks, raising the mean value in the rendered image [S1, unofficial]. This effect is quantified by the Noise Accumulation Factor (NF) [S1, unofficial]. The rendered background intensity $L_{Brendered}$ is given by [S1, unofficial]:

$$ L_{Brendered} = B \cdot (1 + NF) $$

Where $B$ is the background activity. The net contrast ratio $C_{net}$ is affected by NF as [S1, unofficial]:

$$ C_{net} = (C_{th}/C_e + 1) \cdot NF - 1 $$

For small target lesions where thickness $n$ approaches unity, $NF \approx 1$ [S1, unofficial]. In a uniform object 7 voxels thick with mean activity 100 and Poisson noise, MAP increased the mean value by 13%, implying $NF \approx 1.13$ for that specific geometry [S1, unofficial].

**Pseudocode:**

> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
import math

def maximum_activity_projection(volume, alpha, angles):
    """
    Renders a 3D volume using Maximum Activity Projection with simulated attenuation.
    
    Args:
        volume: 3D list of lists of lists (Nx, Ny, Nz) of voxel activities.
        alpha: Float, attenuation coefficient (e.g., 0.0, 0.024, 0.049).
        angles: List of rotation angles (in degrees) for cine display.
        
    Returns:
        rendered_images: List of 2D lists (Nx, Ny) for each angle.
    """
    Nx = len(volume)
    Ny = len(volume[0])
    Nz = len(volume[0][0])
    rendered_images = []
    
    for angle in angles:
        # Initialize output image with zeros
        I = [[0.0 for _ in range(Ny)] for _ in range(Nx)]
        
        # In the standard clinical implementation described in [S1, unofficial],
        # rays are cast along the Z-axis (depth) for a fixed view.
        # For rotation, the volume data is typically rotated, 
        # or the ray direction is transformed. 
        # Here we assume the volume is pre-rotated or we are rendering 
        # a single posterior view as described in the primary implementation.
        
        for x in range(Nx):
            for y in range(Ny):
                max_val = float('-inf')
                
                # Iterate along the depth (z-axis)
                for z in range(Nz):
                    # Get voxel value
                    voxel_val = volume[x][y][z]
                    
                    # Apply simulated exponential attenuation
                    # Note: z is used as depth. In a strict physical model, 
                    # z might need to be scaled by voxel size (cm/voxel).
                    # The source uses z directly in the exponent with alpha in cm^-1,
                    # implying z is in cm or alpha is normalized. 
                    # We follow the source formula directly.
                    attenuated_val = voxel_val * math.exp(-alpha * z)
                    
                    # Select maximum
                    if attenuated_val > max_val:
                        max_val = attenuated_val
                
                I[x][y] = max_val
        
        rendered_images.append(I)
        
    return rendered_images
```

**Edge Cases and Pitfalls:**
*   **Rotation Illusion:** Without simulated attenuation ($\alpha=0$), viewing in a rotating format (cine display) may cause an illusion of reversal of rotation [S1, unofficial]. Adding attenuation prevents this [S1, unofficial].
*   **Contrast Degradation:** Simulated attenuation and noise cause contrast degradation compared to slice data [S1, unofficial]. For a cylindrical phantom, contrast degradation is 19% for Max0.0, 32% for Max0.024, and 54% for Max0.049 [S1, unofficial].
*   **Loss of Low-Uptake Regions:** Areas of decreased uptake are not shown [S1, unofficial]. In objects with mildly increased uptake centrally and more intense peripheral uptake (e.g., tumor with necrotic center), the central activity will not be visible [S1, unofficial].
*   **Noise Increase in Background:** The maximum operator selects noise peaks, increasing the detected background value and slightly decreasing lesion contrast [S1, unofficial].

## Operating characteristics
**Time and Memory Complexity:**
*   **Time:** $O(N_x \cdot N_y \cdot N_z)$ per view, as every voxel is visited once to find the maximum along the ray [G]. For $N$ views, complexity is $O(N \cdot N_x \cdot N_y \cdot N_z)$ [G].
*   **Memory:** $O(N_x \cdot N_y \cdot N_z)$ to store the volume, plus $O(N_x \cdot N_y)$ for the output image [G].

**Measured Performance:**
*   **Noise Reduction:** MAP reduces image noise compared to slice data and raw projection [S1, unofficial]. RMS noise measurements: Slice (7.2%), Max0.0 (3.7%), Max0.024 (3.4%), Max0.049 (3.6%), Projection (4.6%) [S1, unofficial].
*   **Contrast:** MAP preserves contrast better than summed projection [S1, unofficial]. In simulated bone lesions, lesion contrast in MAP is approximately equal to slice display and superior to summed reprojection [S1, unofficial].
*   **Hardware:** Implemented on a MicroVAX-II minicomputer in the original study [S1, unofficial].

**Strengths and Limitations:**
*   **Strengths:** Enhances continuity of structures [S1, unofficial]. Preserves spatial relationships [S1, unofficial]. Reduces noise compared to slices [S1, unofficial]. Does not require segmentation or thresholding [S1, unofficial]. Retains information from the entire volume [S1, unofficial].
*   **Limitations:** Cannot display areas of decreased uptake [S1, unofficial]. Contrast degrades with higher attenuation and noise [S1, unofficial]. Central low-uptake regions in high-uptake lesions are obscured [S1, unofficial].

**When to Use:**
*   Use for "hot spot" imaging in nuclear medicine (e.g., bone scintigraphy, hemangioma detection) where highlighting high-activity structures is the goal [S1, unofficial].
*   Use when a 3D effect is desired via cine display, requiring simulated attenuation to prevent rotation illusions [S1, unofficial].

**When Not to Use:**
*   Do not use if the clinical task requires visualizing areas of decreased uptake or necrotic centers within tumors [S1, unofficial].
*   Do not use if quantitative accuracy of background levels is critical without correcting for the Noise Accumulation Factor [S1, unofficial].

## Background
Maximum Activity Projection was proposed by Jerold W. Wallis and colleagues in 1990 [S1, unofficial]. It was developed to address the limitations of conventional slice display in nuclear medicine, which is complex to interpret due to the need to examine many slices and follow structures across them [S1, unofficial]. MAP combines the spatial relationships of planar imaging with the high contrast of tomographic slices [S1, unofficial]. It was implemented on a MicroVAX-II and used in clinical SPECT studies for bone scintigraphy and hemangioma detection [S1, unofficial]. The method is distinct from segmentation-based volume rendering as it requires no user-selected parameters or thresholding [S1, unofficial].

## Sources

[S1] "Volume rendering in three-dimensional display of SPECT images", http://jnm.snmjournals.org/cgi/pmidlookup?view=long&pmid=2384811, fetched 2026-10-29 (unofficial)
