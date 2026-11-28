Hurricane Isabel Scientific Data Visualization

This repository contains Python scripts utilizing the Visualization Toolkit (VTK) to process and render scientific simulation data of Hurricane Isabel. The project is divided into two distinct rendering techniques: 2D Isocontour Extraction and 3D Direct Volume Rendering.


Contributors:
  -Nishant Sharma
  -Parth Parashar

Part 1: 2D Isocontour Extraction

This script extracts an isocontour from a 2D uniform grid dataset (VTKImageData). Instead of relying on built-in VTK contour filters, this implementation uses a custom, simplified counterclockwise edge-walking algorithm to manually find crossing points, calculate linear interpolation, and build the contour segments from scratch.

Input: 2D dataset and a user-defined Isovalue.

Output: Generates a Output.vtp file.

Viewing: The resulting geometry can be loaded and viewed in ParaView.

<img width="635" height="601" alt="image" src="https://github.com/user-attachments/assets/8e39f6cd-cbef-4af6-aee6-ea3d8ca7c032" />


Part 2: 3D Direct Volume Rendering

This script implements a direct volume rendering pipeline to visualize the 3D pressure field of the hurricane. It maps scalar values to colors and opacities to create a volumetric, semi-transparent cloud.

Ray-Casting: Utilizes vtkSmartVolumeMapper for volumetric rendering.

Transfer Functions: Implements exact data-to-color (vtkColorTransferFunction) and data-to-opacity (vtkPiecewiseFunction) mappings based on hurricane simulation benchmarks.

Advanced Shading: Includes an interactive prompt to enable/disable Phong Shading (Ambient, Diffuse, and Specular light scattering) for realistic 3D depth.

Context: Wraps the visualization in a vtkOutlineFilter bounding box for spatial awareness.

<img width="970" height="844" alt="image" src="https://github.com/user-attachments/assets/10d429f4-fab4-497e-bd28-ea526ade7bb7" />


Requirements

To run these scripts locally, you need Python 3.x and the VTK library:

pip install vtk


Usage

Running Part 1 (2D Contours)

Ensure your dataset (Isabel_2D.vti) is in the root directory.

Run the script:

python 2D_Isocontour_Plot.py


Enter your desired isovalue when prompted. Open the resulting Output.vtp in ParaView.

Running Part 2 (3D Volume Rendering)

Ensure your dataset (Isabel_3D.vti) is in the root directory.

Run the script:

python 3D_Volume_Rendering.py


Type yes or no when prompted to enable Phong Shading. An interactive 1000x1000 GUI window will launch.

Left-Click & Drag: Rotate the volume.

Right-Click & Drag: Zoom.
