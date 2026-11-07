# Stitcher
Reconstruction of 3D Surfaces

# Introduction

  Based on [Fuchs, Henry, Zvi M. Kedem, and Samuel P. Uselton. "Optimal surface reconstruction from planar contours." Communications of the ACM 20.10 (1977): 693-702.](https://dl.acm.org/doi/abs/10.1145/359842.359846?casa_token=QG0Dw4j_pZMAAAAA:5hFtaJwiRkrrcLh3Pz_5Po1DdeNN30CWM9BPfFtFr7dXSv86-_VAvzRlOoOsdf7Z_-a0CwTKLi7ipQ),  I developed a code that could reconstruct a 3D surface given 2D planar contours to start from. The idea is to connect M points in a plane to N points on another plane in the most parsimonious way possible, which is accomplish-able by minimizing the total length of the connection lines. On the process of doing so, the script can also connect contours that belong to a same plane by making connecting their nearest points.

  A possible desired output is a self-avoiding surface, instead of one that JUST minimizes total length. Such surface is currently searched and created by default but can be ignored by setting Surface()._intersection_range = 0.

  One tool that will be used to improve the final surface is:

      . Artificially improving the resolution by interpolating the coefficients of the Fourier's Series of each contour

  The goal of this code is to be able to create 3D models of brains for futher analysis in our lab, [metaBIO](https://metabio.netlify.app).

# Installation

  Simply run:

  ```
  pip install numpy brain-stitcher
  ```

  or, alternatively, you can use clone this repository and use the reconstruction.py script without installation. Simply import all the classes containined in the file:

  ~~~python
  from reconstruction import * 
  ~~~

# Libraries Used
  For this library there's only one dependency:

    . Numpy


  But for futher manipulation, one may use other python libraries to handle the .obj output. Another solution for futher manipulation is to use mesh editors: [Meshlab](https://www.meshlab.net) and [Blender](https://www.blender.org) are free to use and have a large community for support.
  
# The Input

  Create either a sequence of Points() or a numpy array as numpy.array([x,y,z]*N) and pass it to the Perimeter() class: Perimeter(list_of_points).
  From there, one could use a series of correction algorithms from the Perimeter class to match necessary reconstruction conditions.

      .remove_overlap()
      .area_vec()
      .fix_distance()
      .fix_intersection()
      .area_vec()

  It is also possible to merge plane contours. The algorithm will simply connect the two closest points and form a continous loop.
      .islands_ensemble(perimeter2)

  Don't forget to run fix_intersection() again, specially if the two perimeters are close to one another.

  Finally, stack up the Perimeter() in a Surface() by first creating the surface object: surface= stitcher.reconstruction.Surface(); and then adding a perimeter: surface.add_island(perimeter)

# The Output
  ![Brain Example](img/example_result_alpha00.png)
  To get the output, simply call Surface().Vertices() and Surface().Edges(). The output can be saved to an obj file and visualized at any obj viewer, such as [Meshlab](https://www.meshlab.net).

# Usage
  Since Stitcher is simply a library with 3 classes, I've included a basic script (main.py) that handles basic processing of cortical data. The file_reader is also interesting as it provides a roi_read method for the Osirix output, which is the software me and my coleagues have been using for manual segmentation of ROIs.

  Simply copy the scripts main.py and file_reader.py to your local repository and from there, modifiy the necessary parameters to perform the reconstruciton. Also, keep in mind that the main.py needs a support file for each subject which is called main.json. This json file is sctructured as follows:

  ~~~json
  {
    "Stitches3D": [{
    "1": [["f1.json"], ["f2.json"]],
    "2": [["f2.json", "f3.json", "f4.json"], ["f5.json"]]}],

    "FileDir": "path/to/contours",
    "OutputDir": "path/to/save/results",
    "Name": "OBJECT-NAME",

    "CloseExtra": [{
    "2":[[0,[1,2]]]}],

    "CloseSurface": [{
    "1": [0],
    "2": [1]}]
  }
  ~~~
