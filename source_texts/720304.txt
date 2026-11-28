Put Your design decisions here. What modules you defined and how they
interact to solve the ray tracing problem.

2 Modules:
JSONParser - parses the json file for information pertaining to the objects,
camera, and lights
Render - uses the information parsed by the json file to create spheres
and planes with shading and shadows

Interaction:
Object of JSONParser is created within Render class
Using that object necessary information is pulled from the .json file for
the image
Following that getter methods are used within JSONParser module to transfer 
that information into private variables within Render module

Generation of the image:
Once the information has been pulled from the .json file and is stored within
Render module private variables the generation of the image is done
solely within the Render module.
Many methods within the Render perform vector operations: 
Location findFocalPoint(); // find the coordinate values of the focus point
Location findL(Location centerObject, Location focalPoint); // distance find vector between two points
Location findDuv(Location point, Location focalPoint); // find directional unit vector
double magnitude(Location theVector); // find magnitude
Location findW(Location focalPoint, Location Duv, double tca); // findW for sphere
double findMagnitudeFirstMinusSecond(Location w, Location centerObject); // find magnitude of vector between two points
double findDotProduct(Location first, Location second); // calculate dot product
double findThc(double radius, double d); // find thc for sphere

After this the overarching function is: 
void findAllIntersect(int numThreads, int curThread);
Within this function the intersect is calculated, the closest object that is intersected is determined and all shadow, shading and light calculations are determined
using other methods. Then resulting pixel for each pixel of the image is determined and pushed into a vector in order from Left to right and moving from top to bottom
to be later used in the creation of the image.

Within the main it is only necessary to either call the Render method findAllIntersect to generate an image or createThreads {void createThreads(unsigned int numThread)}
to generate an image depending on whether threading is necessary or not.

Then to create the actual output png file the Render method void createImage(string filename); is called to generate the image with the desired output name. This places all the 
pixels that were within the vector of pixels into the image and generates the file.
