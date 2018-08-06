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

Then I did some other stuff and fit my lane lines with a 2nd order polynomial kinda like this:

![alt text][image9]
![alt text][image10]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in lines # through # in my code in `my_other_file.py`

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step in lines # through # in my code in `yet_another_file.py` in the function `map_lane()`.  Here is an example of my result on a test image:

![alt text][image11]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  
