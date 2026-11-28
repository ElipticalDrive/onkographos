# Medical Visualization

## Contributors 
Name | Github
------|----------
Andrew Mohsen | Andrew2077
Mohamed Osama | osama51
Mohamed Alaa | mohammed-44



## Introduction:

In this project we used VTK.js & HTML with React to build a web app 
The app contains the following funcitonality

- Open widget with an example from vtk.js to display *human chest* alongside some options in the form of checkboxes to manipulate the 3D image. This view can be cropped by the user. 
<img src="./images/chest1.PNG" alt="crop in chest" width="500" height="300">
<img src="./images/chest2.PNG" alt="another crop" width="500" height="300">


- Open widget with an example from vtk.js to display *human skull* alongside a slider and a push button to manipulate variable ISO value in a 3D image.
<img src="./images/head.PNG" alt="Surface Rendering" width="500" height="300">
<img src="./images/head2.PNG" alt="Surface Rendering" width="500" height="300">

## Web implementation

The whole implementation is based on the examples of VTK of the human skull and human chest with react elements.

We wrote our main function in App.js for two seperate models in two seperate functions.
The UI for both examples were added in the same file using react which facilitated the implementation of both javascript and HTML codes.
the App function was imported in the index.js file which will then be used inside of reactDOM.render

REACTDOM translates the HTML part into a readable sentences by javascript

In the Vtk part we created a fullscreen render window and created the renderer, we also connected the actor with the renderer and made an object from the mapper, then connected it to the actor. The marching Cube was then created and connected to the mapper.


To switch between the two examples we used an if condition with flag
that detects which example will be rendered, the if condition was applied to the reader part  


```javascript
let flag=1;
function iso (){
if (flag == 0){ 
        flag = 1;
        reader.setUrl(`https://kitware.github.io/vtk-js/data/volume/LIDC2.vti`).then(() => { ,,,
...     //render first model
} else {
        flag = 0;
        console.log(flag)
        reader
        .setUrl(`https://kitware.github.io/vtk-js/data/volume/headsq.vti`, { loadData: true })
        .then(() => {,,,
        //render second model
        }

```
## Issues Faced

- Due to time limitations, we faced some problems and decided to drop some features that would've taken forever to implement.
