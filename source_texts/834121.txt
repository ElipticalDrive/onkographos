# Interpolation-Methods-for-Applying-2D-Neural-Networks-to-3D-Gaussian-Splats

Gaussian splatting, a volume rendering method that enables real time displays of radiance fields and even dynamic 4D scenes, represent the cutting-edge of current 3D rendering techniques. This method has proven highly effective for higher dimensional media. A perennial challenge of performing machine learning on 3D and higher dimensional media is the difficulty of training models, both due to the inherent difficulty of finding optimal solutions in higher dimensions and due to the paucity of good datasets for anything other than 2D digital photos.

The sheer volume of excellent 2D datasets, and already-trained models that perform an absolute plethora of ML tasks, is an incredibly rich resource. That resource is the result of decades of research and enormous quantities of compute. As higher-dimensional rendering techniques proliferate, tapping such a resource for performing ML tasks on higher dimensional volumes could shave years off of research and developmental timelines, and hopefully reduce the amount of compute required for ML on volumes to reach the level of sophistication of current day ML on 2D images. Methods that allow for 2D image models to be directly used for ML on volumes are an important part of this, and deserve investigation.

This project aims to do just that.

Extending the interpolation method for interpolated convolution by [Hart, Whitney, and Morse 2023](https://openaccess.thecvf.com/content/WACV2023/papers/Hart_Interpolated_SelectionConv_for_Spherical_Images_and_Surfaces_WACV_2023_paper.pdf), this project will adapt that method to Gaussian splats and will investigate an effective pipeline for doing so. The anticipated result is a method of applying 2D ML models directly to Gaussian splats.

Preliminary results so far are encouraging. Here, a 3D Gaussian of a kid's play table was styled transfered using a 2D style transfer model and Picasso's "Self Portrait" is shown:

![A still from orginal 3D Gaussian and the style to be transfered to it](StyleTransfer3D.png)

![A still from the resulting 3D Gaussian](table_style_transfer.png)

Sample code is available [here](https://github.com/Raphael-ECU/SelecConv_splats)

Effectively, this project involving creating a pipeline from the raw data of a Gaussian splat to a projection into the 2D plane on which a 2D neural network can natively run, and mapping back up to the 3D volumetric data of the now-modified Gaussian splat. An important step in the pipeline is computation of surface normals for the point cloud subset of the Gaussian splat data. That computation is performed using KNN-based sampling and a computationally efficient normal vector estimation. The results for a scan of an armchair are presented below. As is visual evident, the magnitude of the surface normal at a given point provides an easy way of discriminating between those points on or near the actual surface of our 3D object, and those points far from the surface. Parallel normals are similarly indicative of the orientation of the surface in a given region.

![Chair normals](Normals.png)
