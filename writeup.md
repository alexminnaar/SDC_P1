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
    	slope = (y2 - y1)/float(x2-x1)
```

Then the coordinates of all lines for each lane are accumulated in order to be averaged later.
```
    right_x1s=[]
    right_x2s=[]
    right_y1s=[]
    right_y2s=[]
    
    left_x1s=[]
    left_x2s=[]
    left_y1s=[]
    left_y2s=[]
    
    if lines is not None and len(lines)>0:
        for line in lines:
            for x1,y1,x2,y2 in line:
            
                slope = (y2 - y1)/float(x2-x1)
           
                #if the slope is positive its the left lane, if its negative it is the right, if its zero then ignore it
                if slope >0.0:
                    left_x1s.append(x1)
                    left_x2s.append(x2)
                    left_y1s.append(y1)
                    left_y2s.append(y2)
                
                elif slope < 0.0:
                
                    right_x1s.append(x1)
                    right_x2s.append(x2)
                    right_y1s.append(y1)
                    right_y2s.append(y2)  
```
Then the average line for each lane is calculated along with its average slope

```
            left_x1_avg=np.mean(left_x1s)
            left_x2_avg=np.mean(left_x2s)
            left_y1_avg=np.mean(left_y1s)
            left_y2_avg=np.mean(left_y2s)
    
            right_x1_avg=np.mean(right_x1s)
            right_x2_avg=np.mean(right_x2s)
            right_y1_avg=np.mean(right_y1s)
            right_y2_avg=np.mean(right_y2s)

            avg_left_slope = (left_y2_avg - left_y1_avg) / (left_x2_avg - left_x1_avg)
            avg_right_slope = (right_y2_avg - right_y1_avg) / (right_x2_avg - right_x1_avg)
```
Then the average line is extrapolated downwards to the bottom of the image (`y = 960`) using the formula for slope.  Specifically, `x1`,`y1`, `y2`, and the slope are known and we solve for `x2`.  We do this for both lanes.
```
            new_left_x2= (960 - left_y1_avg+left_x1_avg*avg_left_slope) / avg_left_slope
            new_right_x2 = (960 - right_y1_avg+right_x1_avg*avg_right_slope) / avg_right_slope
```
Then we do the same thing and extrapolate upwards to `y=350` which is approximately the center of the image.
```
            new_left_x1 = -(left_y1_avg - 350 - left_x1_avg*avg_left_slope)/avg_left_slope
            new_right_x1 = -(right_y1_avg - 350 - right_x1_avg*avg_right_slope)/avg_right_slope
```

Then finally the extrapolated lines are drawn
```
            cv2.line(img,(int(new_left_x1),350),(int(new_left_x2),960),color,thickness)
            cv2.line(img,(int(new_right_x1),350),(int(new_right_x2),960),color,thickness)
```

#2.
One shortcoming with my implementation is that the endpoint for the upwards extrapolation is hardcoded rather than learned in a more rigorous way. Also, it is likely that the Hough and mask parameter tuning can be improved.

There are also a few shortcomings associated with this approach in general.  First, this approach can only detect straight lines so it would fail for a curved lane (i.e. a turn).  Second, I suspect that if the car is going uphill or downhill the region in the camera associated with the lane would change (i.e. it would get larger going uphill and smaller going downhill) - therefore the mask would have to be adjusted in these cases.

#3.
As stated earlier this pipeline would be improved by better tuned Hough and mask parameters but this would take more time than I have available at the moment.  

It could also be improved by being able to detect curved lines for curved lanes.  It would also be improved by dynamically changing the mask region while going up or downhill.


