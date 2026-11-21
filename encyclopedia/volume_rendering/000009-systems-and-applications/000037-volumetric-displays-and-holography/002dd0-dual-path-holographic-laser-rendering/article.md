# Dual-Path Holographic Laser Rendering

Also called: Volumetric display system with LC-SLM and 3D laser scanner

## Overview
Dual-Path Holographic Laser Rendering is a volumetric display method that generates palm-sized 3D contents in real space by splitting a femtosecond laser into two distinct optical paths [S1]. One path utilizes a liquid-crystal spatial light modulator (LC-SLM) to generate holographic voxel patterns, while the other employs a 3D laser scanner for positional control [S1]. This cooperative dual-path approach solves the limitation of single-path systems by combining the high voxel density of holographic excitation with the flexible spatial scanning of galvanometers, enabling well-filled volumetric graphics and animations within a centimeter-sized volume [S1]. It differs from pure holographic methods by integrating mechanical scanning for axial positioning and from pure scanning methods by using CGH for lateral voxel distribution per slice [S1].

## How it works
**Inputs and Outputs**
*   **Input:** 3D point cloud data representing the target volumetric content.
*   **Output:** Visible volumetric graphics formed by multiphoton excitation in a gas-filled rendering space.

**Data Structures**
The system processes 3D point cloud data by dividing it into two volumes along the $z$-direction (the optical axis) [S1].
*   **Volume Division:** The point cloud is split into two subsets, one for each optical path.
*   **Slicing:** Each subset is sliced in the $z$-direction to create 2D layers.
*   **Sorting:** Slices are sorted to minimize the total scanning path length between consecutive slices [S1].

**Optical Layout and Parameters**
The system uses a Yb:KGW femtosecond laser (Pharos series PH1SP-10-1M, Light Conversion) with a center wavelength of 1030 nm, a repetition frequency of 10 kHz, and a pulse duration of < 155 fs [S1]. The laser beam is split into two optical paths using a half-wave plate and a polarizing beam splitter, delivering 480 µJ of pulse energy to each path [S1].

*   **Path 1 (Holographic):** The laser passes through an LC-SLM (IL-WU070, JVCKENWOOD) with a pixel pitch of 8 µm and a display area of 1920 × 1200 pixels [S1]. The SLM displays Computer-Generated Holograms (CGHs) to modulate the laser phase, controlling the number and position of focal points within the slice [S1].
*   **Path 2 (Scanning):** The laser passes through a two-axis galvanometer scanner (GVS312, Thorlabs) and a varifocal lens (TL-16-40-TC, Optotune) [S1].
*   **Common Optics:** Both paths use an F-theta lens with a focal length of 100 mm to focus the laser onto the rendering space [S1]. A concave lens with a focal length of -568 mm is placed immediately after the varifocal lens to protect optical elements from reflected light from the glass cell surface [S1].
*   **Rendering Space:** A 100 × 100 × 100 mm³ glass cell filled with Xenon gas at 750 Torr [S1]. The gas reduces the threshold energy for excitation and increases brightness intensity [S1].

**Rendering Area**
*   **Path 1:** Horizontal range of 50 × 50 mm² at an axial position of 88.4 mm from the lens [S1].
*   **Path 2:** Horizontal range of 60 × 60 mm² at an axial position of 114.4 mm from the lens [S1].
*   **Combined Axial Range:** 60 mm [S1].

**Numbered Steps**
1.  Obtain 3D point cloud data for the target content [S1].
2.  Divide the target 3D point cloud data into two volumes along the $z$-direction (aligned with the optical axes of both paths) [S1].
3.  Slice the divided point data in the $z$-direction to create 2D layers for each path [S1].
4.  Sort the slices such that the scanning path between consecutive slices is minimized [S1].
5.  Form the graphics by stacking these 2D slices in the $z$-direction [S1].
6.  Convert the 3D scanning data (positions for the galvanometer and varifocal lens) into voltage signals via a D/A converter [S1].
7.  Apply the voltage signals to the galvanometer scanner and the varifocal lens to position the emission points [S1].
8.  Display the corresponding CGH on the LC-SLM for each slice to define the lateral voxel pattern [S1].

**Formulas**
The horizontal position $(x, y)$ of an emission point is determined by:
$$ (x, y) = (f \theta_x, f \theta_y) $$
Where:
*   $f$ is the composite focal length of the F-theta lens, concave lens, and varifocal lens [S1].
*   $(\theta_x, \theta_y)$ is the deflection angle of the galvanometer scanner [S1].
This position varies depending on the axial position at which the graphics is rendered [S1].

**Pseudocode**
> **Unverified code.** The code below did not pass a test run against a reference implementation, or no test fits it, so it may contain errors.

```python
def render_dual_path(point_cloud, path1_z_range, path2_z_range):
    """
    Renders volumetric content using dual-path holographic laser rendering.
    
    Args:
        point_cloud: List of 3D points (x, y, z).
        path1_z_range: Tuple (z_min, z_max) for Path 1.
        path2_z_range: Tuple (z_min, z_max) for Path 2.
    """
    # Step 2: Divide point cloud into two volumes along z-direction
    points_path1 = [p for p in point_cloud if path1_z_range[0] <= p.z <= path1_z_range[1]]
    points_path2 = [p for p in point_cloud if path2_z_range[0] <= p.z <= path2_z_range[1]]
    
    # Step 3: Slice the divided point data in the z-direction
    # Determine unique z-slices for each path
    slices_path1 = get_z_slices(points_path1)
    slices_path2 = get_z_slices(points_path2)
    
    # Step 4: Sort slices so that the scanning path is the shortest
    # This implies an optimization of the order in which z-slices are rendered
    # to minimize travel time of the galvanometer/varifocal lens.
    sorted_slices_path1 = optimize_scan_order(slices_path1)
    sorted_slices_path2 = optimize_scan_order(slices_path2)
    
    # Combine sorted slices for cooperative rendering
    # The exact interleaving strategy is not specified, but both paths contribute
    # to the final volume.
    all_slices = merge_slices(sorted_slices_path1, sorted_slices_path2)
    
    # Step 5 & 7: Render each slice
    for slice_data in all_slices:
        z_pos = slice_data.z
        
        # Determine which path(s) contribute to this slice
        # If a slice belongs to both paths, both CGH and scanning must be synchronized.
        
        # Step 6: Convert 3D scanning data to voltage signal
        # For Path 2 (Scanner): Calculate deflection angles based on target x,y
        # and current z position.
        if slice_data.in_path2:
            theta_x, theta_y = calculate_deflection(slice_data.target_x, slice_data.target_y, z_pos)
            voltage_x, voltage_y = d_a_convert(theta_x, theta_y)
            set_galvanometer(voltage_x, voltage_y)
            
            # Set varifocal lens for axial position
            voltage_z = d_a_convert_for_varifocal(z_pos)
            set_varifocal_lens(voltage_z)
            
        # Step 8: Apply CGH for Path 1 (Holographic)
        if slice_data.in_path1:
            # Generate CGH for the 2D slice pattern
            cgh_pattern = generate_cgh(slice_data.points_in_slice)
            display_cgh_on_slm(cgh_pattern)
            
        # Trigger laser pulse
        trigger_laser()

def calculate_deflection(target_x, target_y, z_pos):
    """
    Calculates deflection angles using (x, y) = (f * theta_x, f * theta_y).
    f is the composite focal length.
    """
    # f is derived from the optical system parameters (F-theta, concave, varifocal)
    # Not explicitly given as a single number, but implied by the formula.
    # theta_x = x / f
    # theta_y = y / f
    # Note: The sources state f varies depending on axial position due to the
    # varifocal lens and concave lens combination.
    f = get_composite_focal_length(z_pos)
    theta_x = target_x / f
    theta_y = target_y / f
    return theta_x, theta_y

def get_composite_focal_length(z_pos):
    """
    Returns the composite focal length f for a given axial position.
    This depends on the specific settings of the varifocal lens and concave lens.
    Not given in the sources as a closed-form function.
    """
    # [G] Placeholder for the optical model relating z_pos to f.
    # In practice, this is calibrated or computed from lens parameters.
    pass

def get_z_slices(points):
    """
    Groups points by their z-coordinate into slices.
    """
    # [G] Standard binning or clustering of z-coordinates.
    pass

def optimize_scan_order(slices):
    """
    Sorts slices to minimize scanning path length.
    """
    # [G] Traveling Salesman Problem heuristic or simple nearest-neighbor sort.
    pass

def merge_slices(slices1, slices2):
    """
    Merges slices from both paths into a single render sequence.
    """
    # [G] Interleaving or sequential rendering strategy.
    pass

def generate_cgh(points_in_slice):
    """
    Generates a Computer-Generated Hologram for the given 2D points.
    """
    # [G] Standard CGH algorithm (e.g., Gerchberg-Saxton or direct calculation).
    pass

def d_a_convert(angle):
    """
    Converts angle to voltage.
    """
    # [G] Linear or calibrated conversion factor.
    pass

def d_a_convert_for_varifocal(z_pos):
    """
    Converts z-position to voltage for varifocal lens.
    """
    # [G] Linear or calibrated conversion factor.
    pass

def set_galvanometer(voltage_x, voltage_y):
    """
    Sets the galvanometer scanner.
    """
    pass

def set_varifocal_lens(voltage_z):
    """
    Sets the varifocal lens.
    """
    pass

def display_cgh_on_slm(cgh_pattern):
    """
    Displays the CGH pattern on the LC-SLM.
    """
    pass

def trigger_laser():
    """
    Triggers the femtosecond laser pulse.
    """
    pass
```

**Edge Cases and Pitfalls**
*   **Reflection Damage:** Reflected light from the glass cell surface can damage optical elements. A concave lens is placed immediately after the varifocal lens to mitigate this [S1].
*   **Precision:** The horizontal position $(x, y)$ varies with axial position due to the composite focal length $f$. Accurate calibration of $f$ as a function of $z$ is required [S1].
*   **Gas Pressure:** The rendering space requires Xenon gas at 750 Torr to reduce excitation threshold and increase brightness. Deviations may affect voxel formation [S1].

## Subtopics

### [Liquid-Crystal and Related SLMs](7068-liquid-crystal-and-related-slms.md)
Liquid-Crystal and Related SLMs are electro-optic or micromechanical devices used to modulate light with computed holographic fringes for volumetric display and holography. They solve the challenge of generating high-resolution fringe patterns by leveraging parallel spatial modulation rather than time-multiplexed scanning. Unlike scanned-AOM systems which require high-speed analog conversion and complex scanning mirrors, LCDs and DMDs are more readily interfaced to digital electronics, requiring only demagnification and optical concatenation.

## Operating characteristics
*   **Performance:** The system demonstrated animation at 20 fps and static images with an exposure time of 67 ms [S1].
*   **Voxel Density:** Using CGH to form three voxels achieves higher voxel density compared to graphics rendered by a single voxel [S1].
*   **Rendering Volume:** The combined axial rendering area is 60 mm, with horizontal ranges of 50 × 50 mm² (Path 1) and 60 × 60 mm² (Path 2) [S1].
*   **Strengths:** Enables palm-sized volumetric contents with well-filled voxels and higher voxel density through holographic excitation [S1].
*   **Limitations:** Limited to centimeter-sized volumes; requires complex optical alignment and gas-filled chamber [S1].
*   **Hardware Specifics:** Uses a Yb:KGW femtosecond laser (1030 nm, 10 kHz, <155 fs) and specific commercial components (JVCKENWOOD SLM, Thorlabs scanners, Optotune lens) [S1].

## Background
The method was developed by Kota Kumagai, Hisashi Oka, Kazuki Horikiri, Tetsuji Suzuki, and Yoshio Hayasaki from the Center for Optical Research and Education, Utsunomiya University, Japan [S1]. It was presented at SIGGRAPH Emerging Technologies in 2024 [S1]. The work aims to achieve real-space rendering of palm-sized volumetric contents by optimizing the optical system and scanning path for cooperative dual-path laser rendering [S1].

## Sources
[S1] Volumetric Display with Dual-Path Holographic Laser Rendering, https://doi.org/10.1145/3641517.3664387, fetched 2026-10-31
