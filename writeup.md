## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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

[image1]: ./test_images/test1.jpg "Road Transformed"
[image2]: ./output_images/undistort_test1.jpg "Undistorted"
[image3]: ./output_images/binary_combo_test1.jpg "Binary Example"
[image4]: ./output_images/warped_straight_lines2.jpg "Warp Example"
[image5]: ./output_images/color_fit_lines_test1.jpg "Fit Visual"
[image6]: ./output_images/example_output_test1.jpg "Output"
[video1]: ./output_videos/project_video_with_lanes.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2nd code cell of the IPython notebook "./advanced_lane_finding.ipynb".

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` is appended with a copy of it every time chessboard corners are detected in a calibration image.  Similarly, `imgpoints` is appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

The resultant `objpoints` and `imgpoints` are used to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image2]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image1]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I spent a lot of time tweaking this part and played around with gradients in both x and y directions for all channels of the image in HLS space. In the end I got better results by just trying to detect yellow colored lines (using a tuned color range in HLS space) and white lane marks (using color range in RGB space - HLS space didn't work as well for white color detection) and an additional thresholded combination (tuned on several examples) of S and L channels of HLS space.
The final combination was most robust with respect to shadows and other noise inducing elements (e.g. old lane markings on the pavement or roadside elements - like guard rails).
Here's an example of my output for this step.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `get_perspective_transform_matrices()`.  This function defines the source (`src`) and destination (`dst`) points, which were chosen to produce roughly straight lines (in the warped space) on the two straight line example images. These matrices were hard-coded for the 720x1280 images, although a simple multiplicative transformation would make it work for other image resolutions.

```
src = array([[  275.,   670.],
       [  596.,   450.],
       [  696.,   450.],
       [ 1060.,   670.]], dtype=float32)
dst = array([[  275.,   700.],
       [  275.,   100.],
       [ 1060.,   100.],
       [ 1060.,   700.]], dtype=float32)
```
I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I used a sliding window approach to find the pixels belonging to the lane markings in the warped binary image. This approach starts off by getting a histogram of the lower part of the image and identifying the peaks of the histogram (with a small clearance away from the image edges), which are expected to roughly correspond to the lane positions at the bottom of the image. Then a fixed number of windows (13) is traced from bottom of the image to the top. Each window's horizontal position is initially set to the previous window position (corresponding to the same lane - i.e. left or right) and then adjusted to the mean of the points in that window range (window size was tuned and set to 115 pixels). In order to deal with sharp curves (that would go out of bounds in the warped image), I set counter variables `none_left_count` and `none_right_count`, which record how many times has there been no points detected in the window so far IF the window is at the edge of the image. If that count exceeds 1 then I stop looking for more points of that lane - this is a heuristic used to prevent future windows from picking up noisy points (that could belong to the other lane).
All the points collected using this sliding window approach are grouped together and a 2nd order polynomial is fit to them (for left and right lanes separately). The result looks like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Radius of curvature is calculated in `measure_poly_curvature()` function with the pixels per meter values hard coded based on sample images.


#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

This step is implemented in `draw_lanes()` function. Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

The entire pipeline was combined in the class `LaneDetector` that in addition to combining all the steps also performs basic smoothing by taking the lane output for the current frame to be the median of the past 5 frames (including the current frame). This smoothing makes the lane detector perform more gracefully on frames that would otherwise introduce significant noise (e.g. change of pavement color or shadows from trees).
Here's a [link to my video result](./output_videos/project_video_with_lanes.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

As mentioned earlier, I spent majority of the time on trying to tune the color transform. Being one of the first steps in the overall lane detection pipeline, small changes to the color transform pipeline would be easily amplified down the line and result in poor lane detections. Although I was able to make the color transform work well on the project video by focusing on color detection (yellow and white), it definitely has its shortcomings which become obvious on the challenge videos where the color isn't as uniform (and the glare is pretty bad in some frame sequences) and the output video has many frames that would get "flooded" by the detected yellow or white spots. Initially, I tried to address these issues using gradients (of the color channels in HSL space) but this approach had its weaknesses too - being very eager to detect any sort of lines - e.g. cracks in the pavement or some discoloring (due to paint or shadows) and road edges.
Interestingly, I noticed that different color transform approaches (color detection vs gradient detection) worked better in different scenarios. This lead me to think that one possible solution to make color transform pipeline work in all cases is to have some automated way to decide when to switch between the different approaches. For example we could use a clustering algorithm (e.g. DBSCAN) to cluster the yellow/white points.. If these points seems to have very large (by area) clusters then we can switch to using gradient method. On the other hand if the clusters are long and skinny in shape, then these are likely good line detections and we can keep these.
