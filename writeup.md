#Reflection

#1.
My pipeline consists of the following steps.  First the image is converted to gray scale with the `grayscale()` helper function.  Then the image is blurred with the `gaussian_blur()` helper function.  Then edges are found with the `canny()` helper function.  Then a mask is created to focus on only the lane region with 

```
imshape = image.shape
vertices = np.array([[(200,imshape[0]),(460, 290), (490, 290), (imshape[1],imshape[0])]], dtype=np.int32)
cv2.fillPoly(mask, vertices, ignore_mask_color)
masked_edges = cv2.bitwise_and(edges, mask)
```
Then a Hough line transform is used to find lines in the mask region.  From trial and error, the following Hough transform parameters were used.

```
rho = 2
theta = np.pi / 180
threshold = 15
min_line_length = 15
max_line_gap = 20
```

This provides the lane line segments.  In order to extrapolate the lane lines into a single line for each lane, the `draw_lines()` function was modified in the following way.  First, the lane that the line segment belongs to can be inferred by the sign slope of the segment which can be calculated by simply dividing the rise by the run like so

```
for line in lines:
	for x1,y1,x2,y2 in line:
    	rise = y2-y1
        run = x2-x1
        slope= float(rise)/run
```

Then the topmost (i.e. closest to the top of the image) line segment of each lane is extracted along with its slope like so.  Note that only the lines with slopes in an acceptable range are considered to minimize false positives.
```
#right lane
if slope>0.0 and slope < 1.0:            
	if y1 < topmost_right_x1y1[1]:
    	topmost_right_x1y1=(x1,y1) 
        topmost_right_slope=slope
                    
#left lane
elif slope < 0.0 and slope > -1.0:
	if y1 < topmost_left_x1y1[1]:
    	topmost_left_x1y1=(x1,y1)
        topmost_left_slope=slope
```

Then, given the top-most point recorded and the corresponding slope, the line is extrapolated downwards towards the bottom of the image.  This is done using the equation for slope where the top point of the line , the slope and the bottom y coordinate (i.e. the height of the image) is known and the bottom x coordinate is solved for.  This was calculated by

```
#calculate the bottom most point for each line
bottom_right_x2 = (imshape[1] - topmost_right_x1y1[1] + topmost_right_x1y1[0]*topmost_right_slope)/float(topmost_right_slope)
bottom_left_x2 = (imshape[1] - topmost_left_x1y1[1] + topmost_left_x1y1[0]*topmost_left_slope)/float(topmost_left_slope)
```

Finally, both extrapolated lines are drawn by using the topmost lines as the starting points and the extrapolated endpoints.
```
#draw lane lines on image  
cv2.line(img,topmost_right_x1y1,(int(bottom_right_x2),imshape[1]),color,thickness)
cv2.line(img,topmost_left_x1y1,(int(bottom_left_x2),imshape[1]),color,thickness)
```

#2.
The largest shortcoming of my implementation is the mask and Hough parameters are such that the topmost lines found sometimes do not correspond to lanes (rather they belong to other objects like cars).  As a result, the extrapolated lines would also be incorrect.  This could potentially be fixed by tuning these parameters better but this would take more time than I have available at the moment.  

There are also a few shortcomings associated with this approach in general.  First, this approach can only detect straight lines so it would fail for a curved lane (i.e. a turn).  Second, I suspect that if the car is going uphill or downhill the region in the camera associated with the lane would change (i.e. it would get larger going uphill and smaller going downhill) - therefore the mask would have to be adjested in these cases.

#3.
As stated earlier this pipeline would be improved by better tuned Hough and mask parameters but this would take more time than I have available at the moment.  

Another Idea I had is the following. In most cases, a stiped lane would have at least 2 lines segments with the same slope (assuming it is not a curved lane).  I could have minimized false positives by only looking at segments that have at least one other segment with the same slope.  This way other lines not associated with lanes could have been filtered out.


