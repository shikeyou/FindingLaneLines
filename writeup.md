#**Finding Lane Lines on the Road** 

##Writeup Template

###You can use this file as a template for your writeup if you want to submit it as a markdown file. But feel free to use some other method and submit a pdf if you prefer.

---

**Finding Lane Lines on the Road**

The goals / steps of this project are the following:

* Make a pipeline that finds lane lines on the road
* Reflect on your work in a written report


[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

###1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consisted of 5 steps:

1) Convert color image to grayscale

2) Apply Gaussian blur on image to remove high frequency data, otherwise the Canny edge detector might detect some of these noise as lines

3) Apply Canny edge detection to obtain thin line segments which represent edges

4) Mask out only the relevant area of interest using a cv2.polyfill. This step removes any line segments found outside the area of concern (directly in front of the car lane). 

5) Use Hough transform to find longer lines which represent combinations of the shorter line segments detected from the steps above

In order to draw a single line on the left and right lanes, I added new functions that will take Hough-transformed lines segments and converted them into a single left line and a single right line. In the instructions, I was asked to modify the `draw_lines()` function but I feel that it does its job of drawing lines as contracted. What should be changed is the coordinates of the lines that are passed into the `draw_lines()` function. I calculated the slopes for each line segment and grouped them based on whether they are positive (right line) or negative (left line). Only slopes near 1 +/- threshold are retained since we are mostly interested in slanted slopes. This helps to remove horizontal and vertical line segment outliers which will cause errors in the single left/right line calculations, and horizontal line segments is exactly the problem in the solidYellowLeft.mp4 video. Then the averages of these slopes are taken for each group to get a single slope for the left line and for the right line. I also calculate the average (x,y) position for each group.

Now to get a line to start from the bottom of the image, I calculated the necessary X value using the average (x,y) position and the line equation `y=mx+b`:

    bottom_point_x = round(avg_point_x + (height - avg_point_y) / average_slope)

All other variables are known, including the slope, the average (x,y) position and y coordinates at the bottom of the screen which is just the height of the image.

To find the top point to draw a line to, I first decided at which y-height the point should be at (using a percentage of height so that it is independent of the image size):

    perc_y_height = 0.6  #use a percentage of image height to specify the y position
    top_point_y = round(perc_y_height * height)

Then just use the same technique to find X value for the top point:

	top_point_x = round(bottom_point_x - (height - top_point_y) / average_slope)

Lastly, due to varying line segments detected per frame, the extracted left and right lines are very jittery since they have no relationship with the previous frame. I smoothed out this motion by blending averages of slopes, x and y positions of the current frame with the values accumulated from previous frames. This helped to smooth out the motion and gave much better results.

    prev_average_slope = (1-prev_frame_blend) * average_slope + prev_frame_blend * prev_average_slope

###2. Identify potential shortcomings with your current pipeline

One potential shortcoming of this pipeline is when the car negotiates a bendy lane or makes a turn, the curved lane markers would not be detected since the Hough transform that we are using only detects straight lines directly in front of the camera.

Another shortcoming could happen when the car goes up or down slopes:
When the car goes up a slope, the triangular lane marker shape would become taller and it might exit the static polygon mask that we are using.
When the car goes down a slope, the triangle would become shorter and more irrelevant line segments might enter the static polygon mask and confuse the system.

###3. Suggest possible improvements to your pipeline

One possible improvement to resolve the first shortcoming would be to use [Generalized Hough Transform](https://en.wikipedia.org/wiki/Generalised_Hough_transform) to detect curves. This is [available in OpenCV](http://docs.opencv.org/trunk/dc/d46/classcv_1_1GeneralizedHoughBallard.html) as well as [other open source codes](http://www.itriacasa.it/generalized-hough-transform/).

Another possible improvement would be to dynamically adjust the polygon mask based on the slopes obtained from the left and right lines. This will allow the mask to dynamically adjust for slopes and help resolve the second shortcoming above.