# Moving Slit Display
Also called: Holotron
## Overview
The Moving Slit Display is a volumetric 3D display technique that projects 2D image patterns onto a moving screen to form a stack of spatial images along the Z-axis, creating a true 3D image in space viewable without glasses [S1, unofficial]. It is distinguished from its siblings by using a physical slit aperture that moves laterally in front of a high-speed display (typically a CRT) to multiplex multiple views into a sequence of partial images, rather than using static lenses or oscillating screens [S1, unofficial].
## How it works
**Inputs and Outputs:**
- Input: A set of $N$ distinct 2D views of a 3D scene.
- Output: A continuous volumetric image perceived by the viewer.

**Data Structures:**
The display system consists of a high-speed display device (historically a Cathode Ray Tube, CRT) and a mechanical slit aperture.
- **Display Buffer:** A frame buffer capable of high refresh rates. The buffer layout for a single frame consists of $N$ columns of image data, where each column corresponds to one of the $N$ views.
- **Slit Aperture:** A physical mask with a narrow vertical slit. The slit width determines the angular resolution and brightness trade-off.

**Steps:**
1. **View Multiplexing:** For each frame, arrange the $N$ views side-by-side in the display buffer. Specifically, the $i$-th column of the $i$-th view is placed adjacent to the others. This forms a "partial image" composed of $N$ columns [S1, unofficial].
2. **Beam Restriction:** The horizontal deflection of the electron beam (in a CRT implementation) is restricted to the momentary position of the partial-image area. This ensures that only the relevant columns are illuminated at any given instant [S1, unofficial].
3. **Slit Movement:** The slit aperture moves laterally across the face of the display. The speed of the slit must be synchronized with the frame rate of the display [S1, unofficial].
4. **Sequential Projection:** As the slit moves, it exposes different parts of the display to the viewer. During a single pass of the slit, the display sequentially shows the set of $N$ views as a sequence of partial images, each composed of $N$ columns [S1, unofficial].
5. **Persistence Integration:** The phosphor persistence of the display (or the integration time of the human eye) blends these sequential partial images into a continuous volumetric image [S1, unofficial].

**Parameters:**
- **Number of Views ($N$):** The number of distinct perspectives multiplexed. Higher $N$ increases angular resolution but reduces brightness per view.
- **Slit Width:** Smaller apertures achieve denser distribution of virtual image sources but greatly reduce the brightness of each view [S1, unofficial].
- **CRT Requirements:** The sampling frequency and phosphor persistence must be sufficiently high to support the rapid sequential display of columns. If standard CRTs are insufficient, a multiple electron gun CRT in conjunction with a multiple moving-slit panel has been proposed to reduce these requirements [S1, unofficial].

**Pseudocode:**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def moving_slit_display(n_views, slit_speed, display_refresh_rate):
    """
    Simulates the control logic for a Moving Slit Display.
    
    Args:
        n_views (int): Number of views to multiplex.
        slit_speed (float): Speed of the slit movement (pixels per second).
        display_refresh_rate (float): Refresh rate of the display (Hz).
    """
    # Calculate the number of columns per view
    # Assuming a square display for simplicity, width W, height H
    # Each view occupies W/n_views columns
    
    # Initialize display buffer
    # Buffer size: H x W
    display_buffer = create_buffer(height=H, width=W)
    
    # Generate N views
    views = generate_n_views(n_views)
    
    # Synchronize slit movement with display refresh
    # The slit moves continuously, but the display updates discretely
    
    while True:
        # Get current slit position
        slit_x = get_slit_position(time)
        
        # Determine which columns are currently behind the slit
        # The slit has a finite width, but for ideal multiplexing,
        # we assume it reveals one column set at a time or a small window
        
        # Update display buffer with multiplexed views
        # Place column i of view i at position (i * col_width, :)
        for i in range(n_views):
            col_start = i * (W // n_views)
            col_end = col_start + (W // n_views)
            display_buffer[:, col_start:col_end] = views[i][:, 0]
            
        # Render the display buffer
        render(display_buffer)
        
        # Wait for next refresh
        wait_for_refresh(display_refresh_rate)
```

**Edge Cases and Pitfalls:**
- **Brightness vs. Resolution:** Selecting a small aperture to achieve dense distribution of virtual image sources greatly reduces the brightness of each view [S1, unofficial].
- **CRT Limitations:** Standard CRTs may not have sufficient sampling frequency or phosphor persistence. This necessitates specialized hardware like multiple electron gun CRTs [S1, unofficial].
- **Synchronization:** Precise synchronization between the slit movement and the display refresh is critical to avoid image tearing or misalignment.

## Operating characteristics
**Time and Memory Complexity:**
- Time Complexity: The time complexity is dominated by the display refresh rate and the mechanical speed of the slit. The system must update the display buffer at a high frequency to match the slit movement.
- Memory Complexity: The memory requirement is proportional to the number of views $N$ and the resolution of each view.

**Measured Performance:**
- Not given in the sources.

**Scaling:**
- Scaling to higher resolutions requires higher display refresh rates and faster slit movement mechanisms.

**Strengths and Limitations:**
- Strengths: Provides a true volumetric image without the need for glasses. Can achieve a wide field of view.
- Limitations: Brightness is significantly reduced due to the slit aperture. Requires high-speed displays and precise mechanical synchronization.

**When to Use:**
- Use when a glasses-free volumetric display is required and high brightness is not the primary concern.

**When Not to Use:**
- Do not use when high brightness or low power consumption is critical.

## Background
The Moving Slit Display, also known as the Holotron, was proposed by St. John [S1, unofficial]. It is a variant of the multi-planar volumetric 3D display technique. The concept was developed to address the limitations of static parallax barriers and lenticular lenses by introducing motion to the aperture, allowing for a more flexible and potentially higher-quality volumetric image.

## Sources
[S1] Three-dimensional display technologies, https://pmc.ncbi.nlm.nih.gov/articles/PMC4269274, fetched 2026-10-29 (unofficial)
