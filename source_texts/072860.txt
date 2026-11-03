# VTK-QT-ITK-Heart-App

versions:VTK-8.2.0 QT-5.15.2 ITK-5.2.1 VMTK(If you do not want to manually extract the vascular center line, then VMTK is not necessary)

You first need to install these libraries and reference them in your project file

We use VTK to display DICOM file sequences (including three orthogonal slices and three views), use ITK to segment DICOM images and read detailed DICOM file information, use VMTK for vascular centerline extraction, and finally integrate these functions in the GUI interface using QT.
The main functions of this APP are as follows: 
1. dicom image reading and three-view display 
2. Segmentation based on seed points 
3. 3D volume rendering
4. Vascular centerline extraction
5. Vascular cross-sectional display
Author: Medical image visualization team
