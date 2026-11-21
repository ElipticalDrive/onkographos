# Volume Rendering, Interaction and Histogram Computation using Visualization Toolkit


The aim of this short assignment is to display a Computed Tomography (CT) image using volume rendering as well as iso-surface representation using visualization toolkit. A CT dataset was provided to us which would be provided on request.


## Getting Started


The following instructions shall ensure that the assignment is up and running on your local machine for
development and testing purposes. See ‘Running the tests’ section to know how to run the assignment.


## Prerequisites


Software Required: Python 2.7 (Anaconda3 2018.12 64 bit) for Windows 10 as IDE
Package Required : vtk 8.1.0
CT dataset will be shared on request

## Installation


* Follow the instructions in the documentation https://docs.anaconda.com/anaconda/install/windows/
for downloading Python version 2.7 for Anaconda 2018.12 for Windows Installer.
* After launching the Anaconda Navigator, create a new environment for running vtk programs
by referring to the following documentation:
https://docs.anaconda.com/anaconda/navigator/tutorials/manage-environments/#creating-a-new-environ
ment
* Import vtk, numpy and scipy packages into this newly created environment by following the
guidelines specified in:
https://docs.anaconda.com/anaconda/navigator/tutorials/manage-packages/#installing-a-package
* In the Anaconda Navigator, click ‘Home’ on the left panel, then choose the newly created
environment for running the vtk programs from the drop down provided under ‘Applications on’ located
on the top most section of the navigator.
* Click on the ‘Install’ button under Spyder. On completing the installation click the ‘Launch’ button
under it.


## Running the tests


Upon launching the spyder application, copy paste the code from assignment2_code.py file and save it
on your local machine. Now run the code by clicking the green play button provided in the title bar of the
application. Once you get the output pop-up use your mouse rotator to zoom-in and zoom out. Click on
any object and rotate it in different directions.


## Expected Successful Test Results: 

You would see a visualization pop up window with black background and three different representations of CT image. 
The first viewport shall demonstrate an image after applying volume rendering, second viewport will show the iso-surface image representation and the third view port will have both techniques applied to the objects of the first two viewports. All 3
objects can be rotated in this window in such a way that they all move in a synchronized orientation. A
jpg file of the rendered scene will also be saved in the same location as the python file.

You would see a 3D representation of the below scene in vtk on running my code.
Fig: Viewport 1 indicates Volume Rendering; Viewport 2 indicates Iso-Surface and Viewport 3
indicates a combination of Volume Rendering and Iso-Surface


## Explanation of what and how the test performs


* View Port 1: An image after applying volume rendering (what is displayed)
Significant classes/methods used: vtkDICOMImageReader(), vtkColorTransferFunction and
vtkPiecewiseFunction(), vtkGPUVolumeRayCastMapper(), vtkVolume(), vtkVolumeProperty()
(how is it displayed)
* View Port 2: An iso-surface image representation using marching cubes algorithm(what is displayed)
Significant classes used: vtkMarchingCube() (how is it displayed)
* View Port 3: Volume Rendered CT data and iso-surface extracted using marching cubes algorithm
Significant classes used: No new object was created here; the pre-used objects were rendered and the
desired effect of the combination was obtained.
* The camera was set for the first port using vtkCamera(), SetActiveCamera(). The camera set for the
port 2 and 3 is synchronised with port 1 using SetActiveCamera() and GetActiveCamera() in order to get
the same view for all ports.
* Desired Effect: On rotating any object out of the three ports, the remaining two would also move in
the same orientation.


## References


[1] https://vtk.org/doc/nightly/html/annotated.html
[2] https://github.com/Kitware/VTK
