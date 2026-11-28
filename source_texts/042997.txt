# implementing-basic-X-Ray-and-MIP-volume-rendering-using-ray-casting
This project implements X-Ray and MIP rendering using ray casting to visualize 3D .vol datasets like ctHead.vol. Built with FLTK, it offers interactive features like rotation, scaling, and orthographic projections. X-Ray sums voxel intensities, while MIP highlights the highest, aiding scientific and medical exploration.
#define _CRT_SECURE_NO_DEPRECATE
#include "Application.h"
#include <FL/fl_file_chooser.H>
#include "Gui.h"
#include <cmath> 

extern Gui *gui;
Image curImage; // make inImage a global, need it in other classes
Volume vol;
double bbx[2];
double bby[2];
double bbz[2];
bool usingboundingbox = false;
double t1 = 0.0, t2 = 0.0;
double origin[3] = { 0.0, 0.0, 0.0 };
double direction[3] = { 0.0, 0.0, 1.0 };
double M_PI = 3.14159265358979323846;
ImageType currentImageType; // Default to MIP


struct Point3D {
	double x, y, z;
};

struct Vector3D {
	double x, y, z;
};

// Define the Ray structure
struct Ray {
	Point3D origin;
	Vector3D direction;
};

// the constructor method for the Application class
Application::Application()
{
  // initialize the image data structure
  curImage.nx=curImage.ny=curImage.n=curImage.ncolorChannels=0;

  // add more initialization here:
  vol.nx = vol.ny = vol.nz = vol.n = 0; vol.data = NULL;
}


// p is the starting position, ray is the ray direction vector
	// return t_front and t_back
double Application::vIntersectRaywithVolumeBoundingBox(double p[3], double ray[3], double& t_front, double& t_back)
{
	double bbx0, bbx1, bby0, bby1, bbz0, bbz1;	// bounding box
	double dev;
	bool useboundingbox = true;
	double t1, t2, mint, maxt;
	double tmin, tmax;

	// define vol
	t_front = -1e10;
	t_back = 1e10;
	dev = 0.0001;
	if (useboundingbox) {
		bbx0 = bbx[0] - dev;	bbx1 = bbx[1] + dev;
		bby0 = bby[0] - dev;	bby1 = bby[1] + dev;
		bbz0 = bbz[0] - dev;	bbz1 = bbz[1] + dev;
	}
	else {						    //vol
		bbx0 = 0 + dev;		bbx1 = vol.nx - 1 - dev;
		bby0 = 0 + dev;		bby1 = vol.ny - 1 - dev;
		bbz0 = 0 + dev;		bbz1 = vol.nz - 1 - dev;
	}

	if (fabs(ray[0]) <= 1e-5) {
		if (p[0]<bbx0 || p[0]>bbx1)
			return t_front, t_back;
	}
	if (fabs(ray[1]) <= 1e-5) {
		if (p[1]<bby0 || p[1]>bby1)
			return t_front, t_back;
	}
	if (fabs(ray[2]) <= 1e-5) {
		if (p[2]<bbz0 || p[2]>bbz1)
			return t_front, t_back;
	}
	if (fabs(ray[0]) > 1e-5) {	// intersect with bbx
		t1 = (bbx0 - p[0]) / ray[0];
		t2 = (bbx1 - p[0]) / ray[0];
		mint = min(t1, t2);
		maxt = max(t1, t2);
		t_front = max(mint, t_front);
		t_back = min(maxt, t_back);
	}
	if (fabs(ray[1]) > 1e-5) {	// intersect with bby
		t1 = (bby0 - p[1]) / ray[1];
		t2 = (bby1 - p[1]) / ray[1];
		mint = min(t1, t2);
		maxt = max(t1, t2);
		t_front = max(mint, t_front);
		t_back = min(maxt, t_back);
	}
	if (fabs(ray[2]) > 1e-5) {	// intersecgt with bbz
		t1 = (bbz0 - p[2]) / ray[2];
		t2 = (bbz1 - p[2]) / ray[2];
		mint = min(t1, t2);
		maxt = max(t1, t2);
		t_front = max(mint, t_front);
		t_back = min(maxt, t_back);
	}
	return t_front, t_back;
}

void  Application::ReadVolume() {
	int n;
	FILE* fp;
	char imageType[20], str[100];

	char* fn = fl_file_chooser("Specify a filename to READ from", "*.{vol}", "");
	if (fn == NULL)
		return;

	int nx = vol.nx;
	int ny = vol.ny;
	int nz = vol.nz;
	unsigned char* involume = NULL;

	fp = fopen(fn, "rb");
	fgets(str, 100, fp);
	sscanf(str, "%s", imageType);
	if (!strncmp(imageType, "P7", 2)) { // volume data
		// skip comments embedded in header
		fgets(str, 100, fp);
		while (str[0] == '#')
			fgets(str, 100, fp);
		// read volume dimensions 
		sscanf(str, "%d %d %d", &nx, &ny, &nz);
		n = nx * ny * nz;

		fgets(str, 100, fp);

		involume = (unsigned char*)malloc(n * sizeof(unsigned char));
		fread(involume, sizeof(unsigned char), n, fp);
	}
	fclose(fp);
	vol.nx = nx;
	vol.ny = ny;
	vol.nz = nz;
	vol.data = involume;
	// the intersection is returned as a pair of t values (t1,t2) such that the intersection points are:
	vIntersectRaywithVolumeBoundingBox(origin, direction, t1, t2);

	gui->DisplayWindow->redraw();
	gui->EditorWindow->redraw();
}


void Application::applyScaling(double factor) {
	scale = factor;
	if (currentImageType == XRay) {
		CreateXRayProjection(); // Generate X-ray image
	}
	else {
		GenerateMipImage(); // Generate MIP image
	}
}

void Application::applyRotation(double angleX, double angleY, double angleZ) {
	rotationX = angleX;
	rotationY = angleY;
	rotationZ = angleZ;
	if (currentImageType == XRay) {
		CreateXRayProjection(); // Generate X-ray image
	}
	else {
		GenerateMipImage(); // Generate MIP image
	}
}

void Application::setCurrentImageType(ImageType img_Type) {
	currentImageType = img_Type;
	if (currentImageType == XRay) {
		CreateXRayProjection(); // Generate X-ray image if selected
	}
	else {
		GenerateMipImage(); // Generate MIP image if selected
	}
}

// Getter function to retrieve the current image type (X-Ray or MIP)
ImageType Application::getCurrentImageType() {
	return currentImageType;
}


Matrix multiplyMatrices(const Matrix& A, const Matrix& B) {
	Matrix result = {};
	for (int i = 0; i < 4; ++i) {
		for (int j = 0; j < 4; ++j) {
			result.elements[i][j] = 0;
			for (int k = 0; k < 4; ++k) {
				result.elements[i][j] += A.elements[i][k] * B.elements[k][j];
			}
		}
	}
	return result;
}
Matrix Application::calculateRotationMatrix(double angleX, double angleY, double angleZ) {
	double radX = angleX * M_PI / 180.0;
	double radY = angleY * M_PI / 180.0;
	double radZ = angleZ * M_PI / 180.0;

	// Rotation matrices
	Matrix rotX = { {
		{1, 0, 0, 0},
		{0, cos(radX), -sin(radX), 0},
		{0, sin(radX), cos(radX), 0},
		{0, 0, 0, 1}
	} };

	Matrix rotY = { {
		{cos(radY), 0, sin(radY), 0},
		{0, 1, 0, 0},
		{-sin(radY), 0, cos(radY), 0},
		{0, 0, 0, 1}
	} };

	Matrix rotZ = { {
		{cos(radZ), -sin(radZ), 0, 0},
		{sin(radZ), cos(radZ), 0, 0},
		{0, 0, 1, 0},
		{0, 0, 0, 1}
	} };

	// Combine the rotations
	Matrix rotationMatrix = multiplyMatrices(rotX, rotY);
	rotationMatrix = multiplyMatrices(rotationMatrix, rotZ);

	return rotationMatrix;

}

void Application::GenerateMipImage()
{
	// Define the volume and image dimensions
	int volumeWidth = vol.nx, volumeHeight = vol.ny, volumeDepth = vol.nz;
	curImage.nx = static_cast<int>(volumeWidth * scale);  // Adjust width based on scaling factor
	curImage.ny = static_cast<int>(volumeHeight * scale); // Adjust height based on scaling factor
	curImage.n = curImage.nx * curImage.ny;
	curImage.ncolorChannels = 1;

	// Allocate memory for the image data
	delete[] curImage.data;  // Clean up old data
	curImage.data = new unsigned char[curImage.n * curImage.ncolorChannels];  // Allocate new data

	// Calculate the rotation matrix
	Matrix rotationMatrix = calculateRotationMatrix(rotationX, rotationY, rotationZ);

	// Find the center of the volume for easier rotation handling
	double centerX = volumeWidth / 2.0, centerY = volumeHeight / 2.0, centerZ = volumeDepth / 2.0;

	// Loop over the output image pixels
	for (int pixelX = 0; pixelX < curImage.nx; ++pixelX) {
		for (int pixelY = 0; pixelY < curImage.ny; ++pixelY) {
			double maxIntensity = 0.0;  // Store the maximum intensity for the current pixel

			// Convert the current pixel coordinates back to the original volume coordinates
			double originalX = pixelX / scale;
			double originalY = pixelY / scale;

			// Create a ray originating from the pixel's mapped position
			Ray ray;
			ray.origin.x = originalX;
			ray.origin.y = originalY;
			ray.origin.z = 0.0;  // Assume rays are initially cast along the Z-axis

			ray.direction.x = 0;
			ray.direction.y = 0;
			ray.direction.z = 1;  // The ray points in the positive Z direction

			// Traverse through the volume along the ray's path
			for (double t = 0.0; t < volumeDepth; t += 0.5) {  // Small steps to sample the volume
				// Compute the coordinates along the ray
				double sampleX = ray.origin.x + t * ray.direction.x;
				double sampleY = ray.origin.y + t * ray.direction.y;
				double sampleZ = ray.origin.z + t * ray.direction.z;

				// Translate the sample position to the volume's center
				sampleX -= centerX;
				sampleY -= centerY;
				sampleZ -= centerZ;

				// Apply rotation to the sample coordinates
				Point3D rotatedSample;
				rotatedSample.x = rotationMatrix.elements[0][0] * sampleX + rotationMatrix.elements[0][1] * sampleY + rotationMatrix.elements[0][2] * sampleZ;
				rotatedSample.y = rotationMatrix.elements[1][0] * sampleX + rotationMatrix.elements[1][1] * sampleY + rotationMatrix.elements[1][2] * sampleZ;
				rotatedSample.z = rotationMatrix.elements[2][0] * sampleX + rotationMatrix.elements[2][1] * sampleY + rotationMatrix.elements[2][2] * sampleZ;

				// Translate the rotated coordinates back from the center
				rotatedSample.x += centerX;
				rotatedSample.y += centerY;
				rotatedSample.z += centerZ;

				// Check if the rotated sample is within the volume bounds
				if (rotatedSample.x >= 0 && rotatedSample.x < volumeWidth &&
					rotatedSample.y >= 0 && rotatedSample.y < volumeHeight &&
					rotatedSample.z >= 0 && rotatedSample.z < volumeDepth) {

					// Calculate the voxel index and fetch the voxel intensity
					int voxelIndex = static_cast<int>(rotatedSample.z) * volumeWidth * volumeHeight +
						static_cast<int>(rotatedSample.y) * volumeWidth +
						static_cast<int>(rotatedSample.x);

					double voxelIntensity = vol.data[voxelIndex] / 255.0;  // Normalize intensity

					// Update the maximum intensity found along the ray
// Update the maximum intensity found along the ray
					if (voxelIntensity > maxIntensity) {
						maxIntensity = voxelIntensity;
					}
				}
			}

			// Store the maximum intensity in the output image (scaled back to 0-255 range)
			curImage.data[pixelY * curImage.nx + pixelX] = static_cast<unsigned char>(maxIntensity * 255.0);
		}
	}

	// Redraw the GUI to reflect the updated image
	gui->DisplayWindow->redraw();
	gui->EditorWindow->redraw();
}



void Application::CreateXRayProjection() {
	// Setup dimensions for the volume and output image
	int volumeWidth = vol.nx;
	int volumeHeight = vol.ny;
	int volumeDepth = vol.nz;

	// Apply scaling to the output image dimensions
	int imageWidth = static_cast<int>(volumeWidth * scale);  // Scaled image width
	int imageHeight = static_cast<int>(volumeHeight * scale);  // Scaled image height
	curImage.nx = imageWidth;
	curImage.ny = imageHeight;
	curImage.n = imageWidth * imageHeight;
	curImage.ncolorChannels = 1;

	// Allocate memory for the output image
	delete[] curImage.data;
	curImage.data = new unsigned char[curImage.n * curImage.ncolorChannels];

	// Compute the rotation matrix
	Matrix rotationMatrix = calculateRotationMatrix(rotationX, rotationY, rotationZ);

	// Calculate the center of the volume for rotation
	double centerX = volumeWidth / 2.0;
	double centerY = volumeHeight / 2.0;
	double centerZ = volumeDepth / 2.0;

	// Process each pixel in the output image (scaled and rotated)
	for (int pixelX = 0; pixelX < imageWidth; ++pixelX) {
		for (int pixelY = 0; pixelY < imageHeight; ++pixelY) {
			// Map the scaled pixel coordinates to the original volume coordinates
			double originalX = pixelX / scale; // Map to original coordinates
			double originalY = pixelY / scale;

			// Initialize the ray originating from the scaled coordinates
			Ray currentRay;
			currentRay.origin.x = originalX;  // Origin is scaled to volume's size
			currentRay.origin.y = originalY;
			currentRay.origin.z = 0;

			currentRay.direction.x = 0;
			currentRay.direction.y = 0;
			currentRay.direction.z = 1;

			float totalVoxelValue = 0.0f;

			// Traverse the ray through the volume and accumulate voxel values
			for (int depth = 0; depth < volumeDepth; ++depth) {
				// Compute the sample position along the ray
				double sampleX = currentRay.origin.x;
				double sampleY = currentRay.origin.y;
				double sampleZ = currentRay.origin.z + depth;

				// Translate to the center of the volume before rotation
				sampleX -= centerX;
				sampleY -= centerY;
				sampleZ -= centerZ;

				// Apply rotation to the sample position
				Point3D rotatedSample;
				rotatedSample.x = rotationMatrix.elements[0][0] * sampleX +
					rotationMatrix.elements[0][1] * sampleY +
					rotationMatrix.elements[0][2] * sampleZ;
				rotatedSample.y = rotationMatrix.elements[1][0] * sampleX +
					rotationMatrix.elements[1][1] * sampleY +
					rotationMatrix.elements[1][2] * sampleZ;
				rotatedSample.z = rotationMatrix.elements[2][0] * sampleX +
					rotationMatrix.elements[2][1] * sampleY +
					rotationMatrix.elements[2][2] * sampleZ;

				// Translate back from the center of the volume after rotation
				rotatedSample.x += centerX;
				rotatedSample.y += centerY;
				rotatedSample.z += centerZ;

				// Ensure the sample is within the bounds of the volume
				if (rotatedSample.x >= 0 && rotatedSample.x < volumeWidth &&
					rotatedSample.y >= 0 && rotatedSample.y < volumeHeight &&
					rotatedSample.z >= 0 && rotatedSample.z < volumeDepth) {
					// Calculate the voxel index and accumulate the intensity
					int sampleIndex = static_cast<int>(rotatedSample.z) * volumeWidth * volumeHeight +
						static_cast<int>(rotatedSample.y) * volumeWidth +
						static_cast<int>(rotatedSample.x);
					float voxelIntensity = vol.data[sampleIndex] / 255.0f;
					totalVoxelValue += voxelIntensity;
				}
			}

			// Compute the average intensity and store it in the output image
			curImage.data[pixelY * imageWidth + pixelX] = static_cast<unsigned char>(totalVoxelValue * 255.0f / volumeDepth);
		}
	}

	// Redraw the GUI to display the updated image
	gui->DisplayWindow->redraw();
	gui->EditorWindow->redraw();
}
