# **Advanced Lane Finding Project**

## Writeup

**Build an Advanced Lane Finding Project**

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

[image1]: ./examples/Calibration.png "Calibration"
[image2]: ./examples/UndistortComparison.png "Undistort Comparison"
[image3]: ./examples/UndistortComparisonTest.png "Undistort"
[image4]: ./examples/ColorThreshold.png "Color Threshold Comparison"
[image5]: ./examples/ColorThresholdInd.png "Color Threshold I"
[image6]: ./examples/ColorThresholdInd2.png "Color Threshold II"
[image7]: ./examples/ColorThresholdInd3.png "Color Threshold III"
[image8]: ./examples/ColorWarp.png "Color Warp"
[image9]: ./examples/FindLines.png "Find Lines"
[image10]: ./examples/FindLines2.png "Find Lines Using Previous Frame"
[image11]: ./examples/Collage.png "Final Output"
[image12]: ./examples/Curve.png "Curve"
[image13]: ./examples/Radius.png "Radius"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 1st code cell of the IPython notebook located in "./Project.ipynb" under section '***Compute the camera calibration matrix and distortion coefficients given a set of chessboard images***'. 

I started by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the 3D world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection. 

I used `cv2.findChessboardCorners` to find the corners of the chessboard and `cv2.drawChessboardCorners` to draw the corners for visual verification as below. For some of the chessboard images the corners couldn't be found as the function couldn't detect the desired number of internal corners.

![alt text][image1]

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to a sample chessboard image using the `cv2.undistort()` function and obtained this result: 

![alt text][image2]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

I used the camera calibration matrix and distortion coefficients obtained during calibration step and applied it to an image using `cv2.undistort()`.

The code for this step is contained in the 4th code cell of the IPython notebook located in "./Project.ipynb" under section '***Apply a distortion correction to raw images***'. 

Here is the example of before and after applying of `undistort` function on a test image.
![alt text][image3]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I tried several combinations of color and gradient thresholds to generate a binary image (code cells 6 through 11 of the IPython notebook located in "./Project.ipynb" under the section '***Use color transforms, gradients, etc., to create a thresholded binary image***').  

I started with calculating Sobel gradient in both 'x' and 'y' and also magnitude and direction. It fared well on normal roads but produced lots of noise or loss of details when road contrast was changing rapidly. Then I used 'B' channel in CIELAB color space which performed better at detecting Yellow color under high contrast/shadow conditions. I also explored color selection and thresholding of Yellow and White colors in HSV/RGB color spaces. Finally I picked the combination of L-channel threshold of HSL, B-channel threshold of CIELAB and Yellow and White color thresholdings of HSV/RGB respectively. You can clearly see the details are improved and loss got reduced. This produced better results on `challenge_video.mp4`

Here's a comparison example of some of the different thresholds I've tried.
![alt text][image4]

Here I created a Color representation of various threshold methods I have used and we can clearly see their contributions under different light conditions. (L-Channel : red, B-Channel : green, Yellow/White Color Selection: blue)
![alt text][image5]
![alt text][image6]
![alt text][image7]



#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `perspective_transform()` (which appears in the 15th code cell of the IPython notebook under the section '***Apply a perspective transform to rectify binary image***').  The `perspective_transform()` function takes an image (`img`) as input, as calculates the source (`src`) and destination (`dst`) points.  I chose the hardcode the source and destination points in the following manner:

```python
	# define source points
    src = np.float32([[0.455*img.shape[1],0.639*img.shape[0]],
                      [0.547*img.shape[1],0.639*img.shape[0]],
                      [0.815*img.shape[1],0.944*img.shape[0]],
                      [0.203*img.shape[1],0.944*img.shape[0]]])

    # define destination points
    offset = img_size[0]*.30
    dst = np.float32([[offset, 0],
                      [img_size[0]-offset, 0],
                      [img_size[0]-offset, img_size[1]],
                      [offset, img_size[1]]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 582, 460      | 384, 0        | 
| 700, 460      | 896, 0      |
| 1043, 680     | 896, 720      |
| 260, 680      | 384, 720        |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image8]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

The two functions `find_lane_pixels()` and `search_around_poly()` performs all the necessary tasks to find and fit a polynomial to the lane lines in an image.

The first one  `find_lane_pixels()` can be found in the 19th code cell of the IPython notebook under the * **Find lane pixels using sliding windows method*** sub-section of '***Detect lane pixels and fit to find the lane boundary.***'

It takes a histogram of the bottom half of the image and tries to find the peak of the left and right halves of the histogram. These will be the starting points for the left and right lines. Then it uses nine windows of equal sizes for each lane, which can fit vertically in the image when they are placed on top of each other. These windows have fixed width to identify lane pixels and through iteration they are placed on top of each other (starting from bottom) such that each one is centered around the midpoint of the pixels from the window below. This effectively follows the lane lines up to the top of the image, and speeds up processing by searching only for activated pixels over a small portion of the image. Pixels belonging to each lane line are calculated and the Numpy library's polyfit() method is used to fit a second order polynomial to each set of pixels. Below is the example image which shows the above process as well the yellow colored representation of the calculated fit:
![alt text][image9]

The second one  `search_around_poly()` can be found in the 21st code cell of the IPython notebook under the * **Find lane pixels using margin from previous frame*** sub-section of '***Detect lane pixels and fit to find the lane boundary.***'

Repeatedly trying to find the lane pixels through sliding windows method for every single frame can be very non-effective while processing a video.
Instead of calculating the activated pixels in each window iteratively, this makes use of some pre-defined margin around the pixels found from a previous frame. Then it makes a highly targeted search in that area to determine the new lane pixels. This effectively speeds up processing by searching only for activated pixels over a predefined area. Pixels belonging to each lane line are identified and the Numpy polyfit() method is used to fit a second order polynomial to each set of pixels. Below is the example image which shows the above process as well the yellow colored representation of the calculated fit:
![alt text][image10]

Whenever the second method `search_around_poly()` doesn't yield any results we can always fallback to the first method `find_lane_pixels()` and repeat the process.

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Once we located the lane line pixels and used their x and y pixel positions to fit a second order polynomial curve which is as follows: 
![alt text][image12]
The radius of curvature can be calculated by using the below formula:
![alt text][image13]
This can be found in the 25th code cell of the IPython notebook under the * **Determine the curvature of the lane and vehicle position with respect to center.*** 

The position of the vehicle with respect to center can be obtained by calculating the difference between lane center and the image center, assuming that the camera was mounted at the center point of the car's windshield.
This can be found in the 26th code cell of the IPython notebook under the * **Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.*** 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The pipeline I've created is responsible for performing all the above steps as well as outputting the calculations back on the images in a visual manner. 
I've implemented this step in the function `pipeline()` which can be found in the 29th code cell of the IPython notebook under the * **Create Final Image Pipeline*** .  

Here is an example of my result for all test images:

![alt text][image11]

---

### Pipeline (video)
I have created a single pipeline for processing both image and video. It has optional parameters to display diagnostics using text and picture-in-picture.
#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Many of the methods I used were taken straight from the classroom. They were pretty useful and helped me to understand the different approaches I have took while completing this project. One problem I faced was miscalculations due to RGB/BGR conversions while using `cv2.imread` in my pipeline until I found that the frames from video files in my pipeline were RGB.

Although my pipeline performed better on the project video, it fails to detect the lane lines under challenging conditions like changes in road color, poor road details, glare and shadows etc. To improve the algorithm may be we need to create different models for different road conditions and apply the appropriate model after determining the type of road image.

I found this project to be quite challenging and while the basic calculations for many color thresholdings which I've learned in the classroom worked well on the project video , they failed miserably on the challenge videos. Later I've started implementing multiple combinations after spending some time in research and was able to get good results atleast on the project video and the challenge video. My approach failed terribly on the harder challenge video and I couldn't succeed to learn to setup a robust model to identify lanes under challenging conditions. I wish to learn more advanced techniques and optimize this pipeline to make it robust in coming future.
