# Sky Alignment Solution Overview
This strategy for calculating the center of a given image with it's celestial coordinates (Altitude and Azimuth) borrows from Blind astrometric calibration of arbitrary astronomical images, a paper from Astrometry.net, an open-source astrometric calibration library, and attempts to implement a solution for star-alignment inspired by their approach.

# Setup
Virtualenv, dataset, and input images sit in same directory as this notebook.

#### Environment + Libraries

`python3 -m virtualenv sky`  
`source sky/bin/activate`  
`python3 -m pip install -r requirements.txt`  

#### HYG Dataset
Download the hygdata_v3 dataset from:  
https://github.com/astronexus/HYG-Database

### Input Images
Load into same directory as this notebook.

# Implementation

#### Step 1: Quad-like shapes
Astrometry.net houses a dataset of "quads", series of four coordinates representing visible stars. A unique hash value is derived from these four coordinates and stored in Astrometry.net's quads database (~80gb). In Astrometry.net's alignment algorithm, they search this database to look for the best matching quads in relation to a give image. Instead of the computationally expensive creation of quads, this algorithm reaches for a shorthand approach.

This implementation will borrow from Astrometry's quads solution, but will leverage other inputs available (lat-lon and time). The steps are:

Build a dataset from the Yale Bright Star Catalog (YBSC) containing only the stars in the YBSC that should be visible given the lat-lon, and time.
Sample a fixed number of brightest stars from each constellation.
The strategy here is to leverage the fact that at any time you take a picture of the night sky, you are inevitably capturing a constellation, or a fragment of one. These constellations are also mostly made of stars visible to the unaided eye, so it seems promising they'll be captured in a variety of photographs. [4]

#### Step 2: Image Processing
In order to extract the coordinates of stars in an image, some processing needs to be done on the image input to remove non-star information (clouds, trees, foreground, etc.). This solution implements a high-pass filter discrete fourier transform on the image. [1] [2]

#### Step 3: Brightest Spot Sampling
The image's brightest spots should represent the apparent x-y coordinates of stars. This algorithm samples and then sub-samples these coordinates. Each sample contains a "brightest" spot, and each sub-sample contains a further sampling of a maximum radius around the initial bright spot.

The goal of sampling is to select a set of stars that is likely to be a fragment of a constellation we can match within the YBSC database. Similar to quad-matching, but a much shorter search.

#### Step 4: Procrustes Distance
This step iterates through the sub-samples (sorted by left-most center of mass): a 2d array of star coordinates, and each visible constellation (also sorted by left-most center of mass): a 2d array of star coordinates- and calculates the Procrustes Distance between them. This computes a distance matrix. The 2d index (i, j) of this matrix's lowest value corresponds to the shortest Procrustes distance (closest match) between samples and constellations. [3]

#### Step 5: Analyze Surrounding Matches
Given a best match, the surrounding stars in the image should have at least a certain amount of match with the surrounding constellations in the YBSC. Astrometry.net's paper uses a Bayesian decision process to reject or accept a hypothesized alignment. This algorithm will re-sample stars in the YBSC and the image, where the sample points will start as close as possible to the matching sub-sample's center of mass, and move outward. It will then compute a new procrustes distance at certain intervals, and if the distance continues to decrease as the new YBSC sample increases, we can accept the hypthosis. [4]

#### Step 6: Extrapolate Celestial Coordinates of Image Center
Once an accepted alignment is found, it means the algorithm is confident it can identify a star in the input image, using two identified stars and their right ascension/declination in the YBSC, we can triangulate the center of the image's right ascension and declination, and convert that to Altitude/Azimuth.

#### Notes:
Astrometry.net reports a near 100% classification rate, so this is a useful tool for checking work.  
http://nova.astrometry.net/user_images/3650177#annotated

## Sources:

[1] Discrete Fourier Transform:  
https://opencv-python-tutroals.readthedocs.io/en/latest/py_tutorials/py_imgproc/py_transforms/py_fourier_transform/py_fourier_transform.html

[2] Gaussian Blurring:  
https://opencv-python-tutroals.readthedocs.io/en/latest/py_tutorials/py_imgproc/py_filtering/py_filtering.html

[3] Procrustes Distance:  
https://graphics.stanford.edu/courses/cs164-09-spring/Handouts/paper_shape_spaces_imm403.pdf (p. 4)  
https://docs.scipy.org/doc/scipy/reference/generated/scipy.spatial.procrustes.html

[4] Astrometry.net:  
https://arxiv.org/pdf/0910.2233.pdf (p.18)
