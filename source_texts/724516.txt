🌐 3D Wireframe Camera: Angular Perspective Projection
This project implements a fundamental 3D rendering pipeline using pure TypeScript and the HTML Canvas API, encapsulated within an Angular standalone component structure. It demonstrates how to use 4x4 matrix transformations for handling camera position, rotation, and generating perspective projection of a simple wireframe city scene.

Key Features
Custom 3D Pipeline: Logic for 3D point transformation, projection, and rendering is handled without external 3D libraries (e.g., Three.js).

Matrix Algebra: Utilizes custom Matrix4x4 class for rotation (X, Y, Z axes) and combining transformation matrices.

Real-time Controls: Interactive keyboard controls allow the user to translate (Arrows), rotate (W, S, A, D, Q, E), and zoom (Z, X) the camera in real-time.

Scene: Renders a simple scene consisting of two rows of buildings and a detailed ground grid, set up for a top-down view.
