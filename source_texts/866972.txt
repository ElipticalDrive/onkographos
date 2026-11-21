# Install & Usage

## Pre-requirements 
The extension is developed with Slicer 4.13 preview with Python 3.9 + VTK 9.1

## Setup
1. Clone the project  `git clone https://github.com/ziyunxiao/SlicerVolumePlaneCut.git`
2. Start Slicer
3. Install the extension. 
   1. Go to Module Extension Wizard under "developer tools"/"Extension Wizard"
   2. Click Select Extension
   3. Select project folder "VolumePlaneCut"
   4. The extension is loaded. And you can go to Utilities/Planecut for later use

## Usage
1. Download sample data. Go to File -> Download Sample Data. Select "CTChest"

![image](https://user-images.githubusercontent.com/48234795/161730252-09357987-5c0f-465d-89dd-4bf73da5dce5.png)

2. In the modules drop down list, select "Volume Rendering". Select Volume to "CTChest" and turn on the display ROI. Change appearence of volume rendering by selecting a preset.

![image](https://user-images.githubusercontent.com/48234795/161730773-09a6b3cf-cbb3-406d-a20c-aa48e8de8edd.png)

3. From the modules drop down list, Go to Utilities -> PlaneCut. Three sliders will appear with the help of which the the volume can be rotated. Choose your rotation and click on "Apply" button.

![image](https://user-images.githubusercontent.com/48234795/161731137-b62f23a3-ba92-48ec-b68c-95d40e7ce7e5.png)

# Run test cases
To test the extension, click on "reload and test" button. This will automatically download the sample data, perform volume rendering, enable cropping and the rotate the volume rendering at 45 degrees for further cropping to be performed.
![image](https://user-images.githubusercontent.com/48234795/161731969-1f97f517-88be-481f-ae0d-94a51c6e6d91.png)



# Development tips
## reference objects and method

Test your code in Python console
```py

import slicer

volumeNode = slicer.mrmlScene.GetFirstNodeByClass("vtkMRMLScalarVolumeNode")
volumeNode.GetID()

print("Volume size: {0}".format(volumeNode.GetImageData().GetDimensions()))

volRenLogic = slicer.modules.volumerendering.logic()
displayNode = volRenLogic.GetFirstVolumeRenderingDisplayNode(volumeNode)

displayNode = volumeNode.GetDisplayNode(volumeNode)

print(f"Display Node: { displayNode.GetID() }")
roiNode = displayNode.GetMarkupsROINode()
print(f"Display Markup Node: { roiNode.GetID() }")

# volume property
displayNode.GetVolumePropertyNode().GetVolumeProperty()

# load volume from file
volume = slicer.util.loadVolume(file_name)
# numpy array from Volume
voxels = slicer.util.arrayFromVolume(volumeNode)

# croping
displayNode.SetCroppingEnabled(True)
roiNode.GetDisplayNode().SetVisibility(True)


# planes
planes = vtk.vtkPlanes()
roiNode.GetPlanes(planes)
roiNode.SetVisibility(False)
n1 = planes.GetNumberOfPlanes()
print(f"Number of Planes {n1}")


# view node
view1Node = slicer.mrmlScene.GetFirstNodeByClass("vtkMRMLViewNode")
print(f"view1 node: {view1Node.GetID()}")

viewNode = slicer.app.layoutManager().threeDWidget(0).mrmlViewNode()
viewNode.SetBackgroundColor(0,0,0)
viewNode.SetBackgroundColor2(0,0,0)

# 3view
view = slicer.app.layoutManager().threeDWidget(0).threeDView()
view.mrmlViewNode().SetBackgroundColor(0,0,0)
view.mrmlViewNode().SetBackgroundColor2(0,0,0)
# reset 3d views
slicer.util.resetThreeDViews()



# get renderer
renderWindow = view.renderWindow()
renderers = renderWindow.GetRenderers()
print(renderers.GetFirstRenderer().GetClassName())
renderer = renderers.GetItemAsObject(0)
camera = cameraNode.GetCamera()

# get Interactor
view = slicer.app.layoutManager().threeDWidget(0).threeDView()
renWin = view.renderWindow()
iren = renWin.GetInteractor()


# Volume Rendering ROI
volumeRoi = slicer.mrmlScene.GetFirstNodeByClass("vtkMRMLMarkupsROINode")
volumeRoi.GetID()
volumeRoi.GetCenter()
# (vtkmodules.vtkCommonDataModel.vtkVector3d)[0.38079845905303955, -22.919204711914062, -175.25]

# Rotate roi widget
trans = vtk.vtkTransform()
trans.RotateX(45)
# trans.GetMatrix()
volumeRoi.ApplyTransform(trans)

```
## Volume rendering

```py

def showVolumeRendering(volumeNode,adjust_preset=False):
  print("Show volume rendering of node " + volumeNode.GetName())
  volRenLogic = slicer.modules.volumerendering.logic()
  displayNode = volRenLogic.CreateDefaultVolumeRenderingNodes(volumeNode)
  displayNode.SetVisibility(True)
  scalarRange = volumeNode.GetImageData().GetScalarRange()
  if adjust_preset:
    if scalarRange[1]-scalarRange[0] < 1500:
        # Small dynamic range, probably MRI
        displayNode.GetVolumePropertyNode().Copy(volRenLogic.GetPresetByName("MR-Default"))
    else:
        # Larger dynamic range, probably CT
        displayNode.GetVolumePropertyNode().Copy(volRenLogic.GetPresetByName("CT-Chest-Contrast-Enhanced"))


showVolumeRendering(volumeNode)

```

Download smaple data
```py
import SampleData
sampleDataLogic = SampleData.SampleDataLogic()
sampleDataLogic.downloadMRHead()
```

Get Node by class
```py
slicer.mrmlScene.GetFirstNodeByClass("vtkMRMLSegmentationNode")
```

Accessing slice vtkRenderWindows from slice views
The example below shows how to get the rendered slice window.

```py
lm = slicer.app.layoutManager()
redWidget = lm.sliceWidget('Red')
redView = redWidget.sliceView()
wti = vtk.vtkWindowToImageFilter()
wti.SetInput(redView.renderWindow())
wti.Update()
v = vtk.vtkImageViewer()
v.SetColorWindow(255)
v.SetColorLevel(128)
v.SetInputConnection(wti.GetOutputPort())
v.Render()
```
