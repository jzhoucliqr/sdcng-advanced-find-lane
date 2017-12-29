
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

[image1]: ./writeup_images/undistorted.jpg "Undistorted"
[image2]: ./writeup_images/undistorted_test6.jpg "Road Undistorted"
[image3]: ./writeup_images/binary_test6.jpg "Binary Example"
[image4]: ./writeup_images/transformed.jpg "Transformed Example"
[image5]: ./writeup_images/with_line.jpg "Fit Visual"
[image6]: ./writeup_images/poly_final.jpg "Output"
[video1]: ./project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points 


---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first code cell of the IPython notebook located in "p4.ipynb" .  

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

![alt text][image1]



### Pipeline (single images)

My pipeline is like this:

```
def pipeline(image):
    undist = cv2.undistort(image, mtx, dist, None, mtx)
    binary = do_threshold(undist, s_thresh=(100, 255), sx_thresh=(20, 100))
    warped, Minv = transform(binary)
    newwarp = findLane(warped, Minv)
    final = cv2.addWeighted(undist, 1, newwarp, 0.3, 0)
    return final
```

#### 1. Provide an example of a distortion-corrected image.

The camera calibration is done only once, it will generate the required matrix to do distort. The matrix is saved as global variable, so inside the pipeline, the first step is to do undistort. 

```
undist = cv2.undistort(image, mtx, dist, None, mtx)
```

The result is as the following image. We can check the up left corner of the tree, and right bottom corner of the white car to see the difference. Although it's hard to tell by eye whether the difference makes sense or not.

![alt text][image2]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

The binary image is generated by the function: 

```
binary = do_threshold(undist, s_thresh=(100, 255), sx_thresh=(20, 100))
```

I followed the lecture to use a combination of S channel color binary together with L channel gradient on 'x' direction to generate the binary image. 

I tried multiple other ways, for eg: try to use gray scale image instead of L channel to get the gradient, also try to use R / H channel to do color binary. But in the end this combination of S channel and gradient of L channel works best.

Here's an example of my output for this step on test6.jpg.

![alt text][image3]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

I tried to manually identify four points on the line in straight\_line2.jpg, then estimate their destinations based on the assumption that the two lines will be in vertical parralell after transformed, and they should be the same distance to the center of the image. The estimation works pretty well. I tried the sample points in the example_writeup, it doesnot work as shown in the writeup, even straight lines it can not transform well.

My final transformation points are like this:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 265,683      | 300,693        | 
| 1060,683      | 980,693      |
| 738,480     | 980,350     |
| 552, 480      | 300, 350        |

For the upper two destination points `y` value, I'm using `350` instead of `0`, because this way the transformation can pick up additional points which lies beyond my source points, to make the poly fit more accurate.

And the transformation on straight_line2.jpg works like this. We can see that the transformation pick up more points on top of the destination rectangle.

![alt text][image4]


#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I followed the lecture to identify lane pixels with sliding window then use `np.polyfit()` to fit a polynomial. It's done in step: 
```
newwarp = findLane(warped, Minv)
```
I'm using a Line() class to keep track of left and right line pixels identified. In the first frame, I take a histogram to find the peaks, then use sliding window to find the pixels. After the success fit in first frame, I keep track of the fit in the Line() object, then in following frames I directly search within a certain margin of the previous fitted lines, this will speed up the process. 

To make the fits more smooth, I'm using a moving average to combine current fitting and previous fitting.

I tried multiple ways to decide if the fitting is bad or not, like using the curvature difference / fitting parameter difference, but it's not easy to find a good threashold for the difference. 

In the end I use a simple algorithm based on the highest order of the fitting parameters. Since left line and right line should be parallel to each other, the highest order of `left_fit` and `right_fit` should have same sign, either both positive or negative. So in current fit, when `left_fit[0]` have different sign from previous `left_fit_previous[0]`, if `right_fit[0]` still have same sign with previous, then I assume left_fit changes too much, and assume it's a bad one. Same algorithm applies to `right_fit`.

When identified a bad fit, I reduced the moving average parameter by 1/4. I tried to completely ignore the bad fit, but it doesn't work well on some of the frames close to the curve. 

The code works like this: 

```
left_part = 0.2
right_part = 0.2
if ((left_fit[0] * leftLine.best_fit[0] > 0) & (right_fit[0] * rightLine.best_fit[0] < 0)):
    # left line ok, right line bad, ignore right_fit, use previous fit
    right_part = right_part / 4
elif ((left_fit[0] * leftLine.best_fit[0] < 0) & (right_fit[0] * rightLine.best_fit[0] > 0)):
    # left line bad, right line ok
    left_part = left_part / 4

left_fit = left_fit * left_part + leftLine.best_fit * (1 - left_part)
right_fit = right_fit * right_part + rightLine.best_fit * (1 - right_part)
```    

The final result looks like this:

![alt text][image5]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I followed the lecture to calculate the radius of curvature. Nothing new in this part. 

To calculate the offset, I get the x value of both lines at bottom of image, then get the difference of the mid point of the lines and mid point of image. This is based on the assumption that the camera is mounted in middle of the car. 

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

The method `findLane()` will return the restored rectangle on top of a binary image, then use `cv2.addWeighted()` to combine with the undistored original image to get the final result.

Inside `findLane()`, I plot on the image with the curvature of both left and right line, the offset, and the parameter of left_fit and right_fit. These provides good debug insights. 

Here is an example of my result on a test image:

![alt text][image6]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video_out.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

* Issue when get binary image

I tried multiple ways to combine different color space and gradient threshold to get a binary image. In the end the S channel with gradient on L channel works best. But still it picks up noise in some frames when there is shade on the road, since the S channel does not remove the shade. Tried with R channel, H channel which removes the shade, but they have some other issues. 

Can do more trial to figure out other ways of combination which might help.

* During perspective transform

The example transform points in the writeup does not work well. So I mannually choose four points to do the transform. It works ok but not perfect. 

For the lane width in destination, it doesn't matter much for straight lines, but it does matter near the curve, because if lane is too wide, then  big part of the curved line may get out of the image, then the polynomial fitting does not work well. 
  

* Line fitting

Since the binary image and perspective transform are not perfect, the input to line fitting have some noise, which cause some bad fittings. 

To identify bad fittings, I tried with curvature / difference of fitting parameters, which I couldn't make it work well. It's kinds of difficult to get a good threshold to measure the difference. So in the end I choose a much simple algorithm discussed earlier. But it's still have some issues. 

For the moving average of fitting parameters, if we use too much previous information, then during curvature it performs bad because the line actually changes a lot. But if we mainly rely on current fit, it again becomes not smooth. So again this is another parameter need more trial. 

 