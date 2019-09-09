
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

[image1]: ./output_images/undistort_output.png "Undistorted"
[image2]: ./output_images/test1_calibrated.jpg "Road Transformed"
[image3]: ./output_images/binary_combo_example.jpg "Binary Example"
[image4]: ./output_images/warped_straight_lines.jpg "Warp Example"
[image5]: ./output_images/color_fit_lines.jpg "Fit Visual"
[image6]: ./output_images/example_output.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup

All code is located in solution.ipynb. I will reference to cell numbers in this file.

#### 1. Provide a Writeup

Done.

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

In cell-2 I load calibration images and display them.

In cell-3 I compute calibration parameters as following:
- create array objp with chessboard corners indeces
- for each callibration image `img`
- - convert `img` to `gray` using cv2.cvtColor
- - find corners in `gray` image using `cv2.findChessboardCorners`
- - save corners indeces and corner coordinates to `objpoints` and `imgpoints`
- call `cv2.calibrateCamera` with `objpoints` and `imgpoints` to find the camera matrix and distortion coefficients

In cell-4 applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

I apply distortion correction in cell-6 and show original and corrected test image in cell-7. 

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

My solution is differ from standart pipline in two aspects:
- I found that it is better to use sobel transform with wide kernel (15 in my work). To save precision at long range I create binary image after birdeye transformation.
- To make polinomial fit more robust I do not do thresholding and binarization. Instead I use intensity in mask as weight for points in linear regression.

To make `mask` in my notation I use function `calc_mask()` in cell-11. It works as following:
- Convert input to HSV and create yellow mask.
- In the same way create white mask (with high witeness threshold)
- Convert input image to gray and compute `sobelx` mask with kernel size 15
- In the same way create sobelx mask for chanel S of image in HLS color space
- Sum up all masks and normalize to 1

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The perspective transform (birdeye) is implemented in cell-8.

Coordinates of the rectangle cornets are in `src` (for original image) and `dst` (in transformed image). Values are hardcoded.
Compute projection matrices M (forward) and Minv (backward) using `cv2.getPerspectiveTransform` 

Define two functions `birds_eye` and `anti_birds_eye`. Transformation matrices are saved as closure in function.

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Code for polynomial construction in cell-13.

There is two function. The first one is `fit_line_model`. It just construct polynomial for a given mask:
- Iterate over mask pixel:
- - if mask[y, x] > threshold (0.1):
- - - save x to array of targets X
- - - save y to array of features Y = [[y**2, y, 1]]
- - - save mask[y, x] to array of weights W
- Fit linear regression using sklearn. I used RANSACRegressor since it is liner regression robust to noise.

The second one is `line_search`. It drives search of the line in the following way:
- Initially we take buttom region of original mask (for y [150:250]). This region a)is near to car and well recognizeble b) cover intermittent road markings in any position c) line in the region is straight enough to specify its ROI, d) curvature of the line is enough to find aproximate line direction.
- Construct initial model (polynomial) using work mask
- Process blocks 50 lines of mask in y direction
- - predict x positions using current model
- - fill work mask in range 50-x+50
- - update model using new work mask
- return last model

Code in cell-14 is for debug. Visualisation of polynomial fitting. You can see original birdeye image, selected pixels of the mask (highlighted in red) and fitted polynomial (green line).

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in cell-15. I scale polynomial from pixels to meters and use formula from the lecture. Position is computed as averege x values of lines polynomials compared to midle of the birdeye (and scaled to centimeters).

Actual computations are done in `process_frame`

I take curvature and position at y=240 (it is about 2 meters in front of the car).


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Final image processing is done in `process_frame` function (cell=16). Line visualisation is done in `draw_result`.


![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_out.mp4)

I upload to copy of output to my cloud [link](https://cloud.mail.ru/public/4vYk/4u3RGfys9)

Notes about robustnes. I do not add any code for error detection and correction because there is no errors for line detection near the car and errors will not affect driving safity :)  Errors at long range all time are fixed in next frame and can be easily detected by measuring line width near and far the car (if difference is too big then the error and we can take next frame).



---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

- I thing that this pipeline will not work on roads with high curvature. But I don't test this case :)
- The pipeline do not work well in case of low contrast
- The pipeline is not robust to noise with vertical lines. 

Possible solutions:
- high curvature: Make more robust algorithm for inital work mask construction.
- low contrast: Either to make more complex line mask construction or to use deep networks for this task (better modern way)
- vertical lines: Use deep networks


