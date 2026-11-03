# Implicit Surface Reconstruction

The project's goal is to:

 * Compute an implicit MLS function approximating a 3D point cloud with given (but possibly unnormalized) normals.
 * Sample the implicit function on a 3D volumetric grid.
 * Apply the marching tets algorithm to extract a triangle mesh of this zero level set.
 * Experiment with various MLS reconstruction parameters.



For this project, I don't have the time to test enough parameters and do acceleration and other additional stuff. The code now implements a basic MLS function, future works include axis align, speed up the algorithm and also adjust the parameters for better output.
