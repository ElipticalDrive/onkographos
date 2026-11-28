# Direct-Volume-Rendering

You will use the volume rendering capabilities of vtk to produce direct volume renderings of the head of a mummy. The dataset mummy.128.vtk is in the same format as the head datasets.
vol_ren.tcl - This script will read the given data and visualize it using the volume rendering (Ray Casting). The given data represents a mummy preserved for a long time. So the skin is dried and not very crisp. The dried skins iso value was found to be around 70 to 90. The skull iso value was around 100 to 120. In order to visualize this data set a opacity transfer function and a color transfer function are constructed. The opacity for values ranging from 0 - 50 is chosen to be 0.0 and 55 - 80 is chosen to be 0.1 (semi translucent) and finally the bone values ranging from 90 - 120 is given a complete opaque value of 1.0. The colors are chosen in such a way that the skin range has a light blue and the bone has a complete white and all other values have a color value of 0.0. The wrap around the body is not included even though the iso value was found to be around 25. This is because it obscures the real data that we are trying to visualize.
vol_mip.tcl  -  This file will create the maximum intensity projection
of the image. This looks more like an x-ray of the mummy. It uses the
inbuilt method in VTK called  vtkVolumeRayCastMIPFunction .  The opacity
transfer function plays a major role in this technique and the color
transfer function is used to adjust the contrast and get good looking
images.
Your work:
1. Rewrite the two tcl code into python code. Note that there are two button (Save, Quit) functions.
2. Test for different transfer functions and colors.
3. (Required for graduate students) Discuss the usage of
          the vtk filters used here.
Turn in:
   
Please put the pictures you got into the report (and graduate students discussion of the filters). Please submit your report and your python code.
