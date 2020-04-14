## Writeup 

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./output_images/undist_output.png "Undistorted"
[image2]: ./output_images/undist_test3.png "Road Transformed"
[image3]: ./output_images/binary_test3.png "Binary Example"
[image7]: ./output_images/src.png "Output"
[image4]: ./output_images/birdeye_straight_lines1.jpg "Warp Example"
[image5]: ./output_images/detectedlanes_test3.jpg "Fit Visual"
[image6]: ./output_images/result_test3.jpg "Output"
[image8]: ./output_images/binary_output.png "Output"
[video1]: ./result.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

The code of each step can be found in the `"advanced_lane_lines.ipynb"` Jupyter notebook. Each step is defined in the Markdown cell before the code.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.


I start by collecting the "object points", which will be the (x, y, z) world coordinates of the chessboard corners. Here I aassume the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then use the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. An example of a distortion-corrected image.

To demonstrate the undistortion step, I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Thresholded binary image 

I use color transforms and gradients to create a thresholded binary image. 

- In the 4th code cell of the above mentioned jupyter notebook, I convert the RBG color space to HLS, in order to threshold the image with respect to different saturation values. Since the lane lines are bright white/yellow and have high saturation values, we threshold the S challel of the HLS colorspace to have values more than 90.
- In the next cell, I threshold with respect to the dorection of the gradient. The gradient in horizontal and vertical direction is calculated using the Sobel filter. Then, the direction is calculated as `$\text{tan}^{-1} \text{abs} (\frac{\nabla_y}{\nabla_x})$`. The gradient direction is thresholded to be in the range of [0.6, 1.5] radians.
- In the next cell, I threshold with respect to the magnitude of the gradient. For this, the gradient in horizontal and vertical direction is calculated using the Sobel filter. The gradient magnitude is `$\sqrt{\nabla_x^2 + \nabla_y^2}$`. The gradient magnitude is scaled to the [0, 255] range and is thresholded to be in the range [30, 100].
- In the next cell, I threshold the gradient seperately in horizontal direction and also in vertical direction. The absolute value of the gradient in horizontal and vertical direction is calculated using the Sobel filter. These are then scaled to lie in range [0, 255] and then thresholded to be in range [20,100] in the horizontal direction and [20,100] in the vertical.

At the end, I combine all these steps with the following logic 
`((gradx == 1) | (grady == 1) | (mag_binary == 1)) & ((dir_binary == 1) | (hls_binary == 1))`

Here's an example of my output for this step.

![alt text][image3]

#### 3. Perspective transform

The code for my perspective transform includes a function called `getM()`, which appears in the 11th code block in the jupyter notebook. I chose the hardcode the source and destination points in the following manner:

The source points were selected from the vertices of the following quadrilateral, shown in red.

![alt text][image7]

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 200, 720      | 200, 720        | 
| 575, 460      | 200, 100      |
| 700, 460     | 1080, 100      |
| 1100, 720      | 1080, 720        |

Using the source and destination points, I used the `getPerspectiveTransform()` function of OpenCV to find the transformation matrix M. I use this transformation matrix to warp an example image and to verify that the perspective transform orks as expected.

![alt text][image4]

#### 4. Identifying lane-line pixels and fitting their positions with a polynomial

In code cell 3, I find the lane pixels using the function `find_lane pixels`. It takes the warped binary image as input. I then calculate the histogram of the binary in Y direction just by summing the binary image in Y direction. The next idea is to find 2 peaks in the histogram, assuming that the starting points of the left and right lanes correspond to these peaks. Starting from this base, a sliding window is used to search and store the continuation of the lane pixels. 9 such sliding windows with margin of 100 and a threshold of minimum pixels of 50 was used. 
Starting from the base window, the imgae indices of nonzeros in the warped binary image are searched for and added to a list called `left_lane_inds` and `right_lane_inds` resp. Then, for the next window, it's center is updated to be the mean position of the found indices. 

Once the lane pixels are found, a second order polynomial is fit on each of the found lane pixels using NumPy's `polyfit` function

The sliding window approach is shown as follows. The red pixels are the detected left lane pixels and the blue ones are the right lane detected pixels. The green boxes are the sliding windows. The yellow lines show the quadratic polynomial fited on the lanes.

![alt text][image5]

#### 5. Calculate the radius of curvature of the lane and the position of the vehicle with respect to center.

The radius of curvature at any point $x$ of the function $x=f(y)$ is given as follows:

$$
\LARGE R_{curve} = \frac{[1 + (\frac{dx}{dy})^2]^{3/2}}{|\frac{d^2x}{dy^2}|}
$$
 
​	 

In the case of the second order polynomial $f(y)=Ay^2+By+C$, the first and second derivatives are:

$$
\large f'(y) = \frac{dx}{dy} = 2Ay+ B
$$

$$
\large f''(y) = \frac{d^2x}{dy^2} = 2Af
$$

So, our equation for radius of curvature becomes:

$$
\LARGE R_{curve} = \frac{(1 + (2Ay + B)^2)^{3/2}}{\left|2A \right|}
$$
​
The $x,y$ are supposed to be in real world coordinates, instead of pixel coordinates. So, we multiply with a scaling factor.

Assuming that the lane is about 30 meters long and 3.7 meters wide, the scaling factor is in vertical direction is 30/height and in horizontal direction is 3.7/(width-2*margin). (Margin is the one defined in the `dst` points in the `getM()` function. `margin=200`)

In the code block 18 of the Jupyter notebook, I define the scaling factor in X and Y direction and calculate the radius of curvature for both the lanes.

##### Vehicle position with respect to center:

Assuming the camera is mounted at the center of the car, such that the lane center is the midpoint at the bottom of the image between the two detected lines, the offset of the lane center from the center of the image (converted from pixels to meters) is the distance from the center of the lane to the center of the image.

In the code block 19, the lane center is calculated to be the midpoint at the bottom of the image between the two lines. The offset (in pixels) is the center of the image minus the lane center.

#### 6. An example image of the result plotted back down onto the road such that the lane area is identified clearly.

In code block 20 and 21, I use the inverse of the transformation matrix M to wrap the perspective transformed image, back to original. The result is then drawn onto this image as follows 

![alt text][image6]

---

### Pipeline (video)

#### 1. A link to my final video output.

Here's a [link to my video result](./result.mp4)

![alt text][video1]
---

### Discussion

#### 1. Problems / issues in the implementation of this project. Where will the pipeline likely fail?  What could be done to make it more robust?

- The assumption that the 2 maximums of the histogram of the biary image are the starting points of the lanes is not robust. If the starting point is detected somewhere else, the next searches will likely be wrong since they search within a window near the previously detected lane pixels.

- The thresholding is heavily cherry-picked and engineered. This could give good results on some images, while fail on the others, as shown in the figure below.

![alt text][image8]

- This method can be unreliable in other weather conditions such as rainy weather and under different illumination conditions such as during the night.

##### Suggetions for possible improvements
- Supervised learning methods can be used to automatically detect the lanes. But, it requires labelled data.
- Different use cases for each possible case need to be engineered.