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

[image1]: ./output_images/CameraCalibration.png "Undistorted"
[image2]: ./test_images/test2.jpg "Road Transformed"
[image3]: ./output_images/Color_Gradient_Threshold.png "Binary Example"
[image4]: ./output_images/warped_straight_line.png "Warp Example"
[image5]: ./output_images/Identify_laneline_pixels_polyfit.png "Fit Visual"
[image6]: ./output_images/result_plotted_ontheroad.png "Output"
[video1]: ./project_video_output.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---



### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "P2.ipynb" 

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I used a combination of color and gradient thresholds to generate a binary image based on S and L channel from HLS value.
From my standpoint, S channel works a fairly robust job of picking up the lines under very different color and contrast conditions. However, I also discovered this S channel might give us improper filtering when there is shadow on the road. Shadow also has really high S channel value, so I need to adjust threshold to avoid this problem. Also, I used SobelX on L channel, which was proper here because L channel represent lightness. Here's an example of my output for this step.  

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `warp()`. I made every title for all the functions above corresponding function, so reviewer can easily recognize what function is now doing.  The `warp()` function takes as inputs an image (`img`).  I chose the hardcode the source and destination points in the following manner:

```python
left_bottom = [202, img.shape[0]]
left_top = [595, 450]
right_top = [685, 450]
right_bottom = [1100, img.shape[0]]

src = np.float32([left_bottom, left_top, right_top, right_bottom])
dst = np.float32([[320,img.shape[0]], [320, 0], [960, 0],[960, img.shape[0]]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 202, 720      | 320, 720      | 
| 595, 450      | 320, 0        |
| 685, 450      | 960, 0        |
| 1100, 720     | 960, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Firstly, I did implement sliding windows and fit a polynomial for the first frame. Afterwards, if the first fitted polynomial was reasonable, I just search in a margin around previous line position. Therefore, it will be based on the first frame if we are on the second frame. Later on, it will search around fitted line position based on previous frame.

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I did this in 'Calculation of the radius of curvature of the lane and the position of the vehicle with respect to center' in my code in  "P2.ipynb".
I applied below code to calculate the radius of curvature. I assumed meter per pixel based on the lesson, and did polyfit for each left and right lane.
Curvature is calculated at the bottom of the image.
```python
ym_per_pix = 30/720
xm_per_pix = 3.7/700
left_fit_cr = np.polyfit(lefty*ym_per_pix, leftx*xm_per_pix, 2)
right_fit_cr = np.polyfit(righty*ym_per_pix, rightx*xm_per_pix, 2)
left_curve_rad = ((1 + (2*left_fit_cr[0]*y_eval*ym_per_pix + left_fit_cr[1])**2)**1.5) / np.absolute(2*left_fit_cr[0])
right_curve_rad = ((1 + (2*right_fit_cr[0]*y_eval*ym_per_pix + right_fit_cr[1])**2)**1.5) / np.absolute(2*right_fit_cr[0])
```
Also,we are assuming car is  at center of x-axis. Based on fitted left and right polynomials,I calculated pixel values for left and right lane at the bottom, which is used for getting position for lane center. Afterwards, I applied meter per pixel to get deviation of the midpoint of the lane from the center of the image. 
```python
car_position = binary_warped.shape[1]/2
l_fit_x_int = l_fit[0]*binary_warped.shape[0]**2 + l_fit[1]*binary_warped.shape[0] + l_fit[2]
r_fit_x_int = r_fit[0]*binary_warped.shape[0]**2 + r_fit[1]*binary_warped.shape[0] + r_fit[2]
lane_center_position = (r_fit_x_int + l_fit_x_int) /2
center_dist = (car_position - lane_center_position) * xm_per_pix
```

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

I implemented this step based on function in my code called `draw_lane()`, and `draw_data()`.  Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_output.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

I think most significant part in this project is how accurate we can detect left and right lanes regardless of surrounding condition. If the thresholded binary image is considerably accurate, everything beyond lane detection will work well unless there is no mistake.
However, this creating binary image is still not that intuitive for me even though I tried many different color space and gradient...
I utilized S channel and L channel for this, but I realized that this combination is not perfect for all the conditions we might face.
I can see my approach will not work if lane is really blur, or there is snowing or raining.  I think I will really need to focus on this lane detection part if I need to go further. If lane detection is working really well, curve fitting and others become reall reliable.
