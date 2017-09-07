## Writeup Template

**Vehicle Detection Project**

The goals / steps of this project are the following:

* Perform a Histogram of Oriented Gradients (HOG) feature extraction on a labeled training set of images and train a classifier Linear SVM classifier
* Optionally, you can also apply a color transform and append binned color features, as well as histograms of color, to your HOG feature vector. 
* Note: for those first two steps don't forget to normalize your features and randomize a selection for training and testing.
* Implement a sliding-window technique and use your trained classifier to search for vehicles in images.
* Run your pipeline on a video stream (start with the test_video.mp4 and later implement on full project_video.mp4) and create a heat map of recurring detections frame by frame to reject outliers and follow detected vehicles.
* Estimate a bounding box for vehicles detected.

[//]: # (Image References)
[car]: ./output_images/car.png
[notcar]: ./output_images/notcar.png
[hog]: ./output_images/hog.png
[windows]: ./output_images/windows.png
[detection]: ./output_images/detection.png

## [Rubric](https://review.udacity.com/#!/rubrics/513/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Vehicle-Detection/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

This document.

### Histogram of Oriented Gradients (HOG)

#### 1. Explain how (and identify where in your code) you extracted HOG features from the training images.

The code for this step is contained in [includes/car_classification.py](./includes/car_classification.py).

The specific code to compute HOG, spatial and histogram features is contained in functions `get_hog_features`, `bin_spatial` and `color_hist` respectively. `change_colorspace` is used to change colorspace and experiment which colorspace produces the best classification and detection results. All these are the called by `extract_features_from_image` to extract the feature set on one image. This operation is then repeated in function `extract_features_from_image_list` on all images in the `cars` and `notcars` datasets.

The whole dataset available to train and test the classifier was composed of 8792 car and 8968 non-car images. 20% of these were set apart for testing. Here's an example:

![alt text][car] ![alt text][notcar]

I have tested HOG features computed with different colorspaces, channels on which HOG was applied, number of orientation bins, pixel per cell and cell per block. Here is an example using the `YCrCb` color space , computing HOG on all channels with `hog_orient=9`, `hog_pix_per_cell=8` and `hog_cell_per_block=2`:

![alt text][hog]

#### 2. Explain how you settled on your final choice of HOG parameters.

I tried various combinations of `colorspace`, `hog_channel`, `hog_orient`, `hog_pix_per_cell` and `hog_cell_per_block`, both comparing the classifier prediction accuracy computed on the test dataset and by observing their performance on portions of the video. The latter rather than the former proved more useful to evaluate of "good" the classifier was. In fact, while almost all tested combinations gave classification scores >0.97, their effectiveness in actually detecting cars in video frames was quite different.

In particular, I tried:
* `colorspace` YUV, YCrCb
* `hog_channel` 0, 'ALL'
* `hog_orient` 9, 11
* `hog_pix_per_cell`= 8, 16
* `hog_cell_per_block`= 2

#### 3. Describe how (and identify where in your code) you trained a classifier using your selected HOG features (and color features if you used them).

The code for this step is contained in function `train_classifier` in file [includes/car_classification.py](./includes/car_classification.py).

In this function features from the both `cars` and `notcars` datasets are extracted and scaled, labels (0 or 1) are assigned, train and test datasets are created and a linear SVM classifier is fitted. Both SVM with linear and rbf kernels were tried. Eventually, the linear solution (`LinearSVC`) was chosen for speed reasons. The whole processing is already slow enough. The prediction score on the test dataset is then computed and the obtained model, scaler and feature parameters are saved in a pickle file to be used later by the car detection algorthm.

### Sliding Window Search

#### 1. Describe how (and identify where in your code) you implemented a sliding window search.  How did you decide what scales to search and how much to overlap windows?

The code for this step is contained in class `VehicleDetector` in file [includes/car_detection.py](./includes/car_detection.py). The specific method is `search_in_image`.

The sliding windows search I implemented uses 4 different window sizes in the 4 different areas of the image, with scales of 1.0, 1.5, 2.0 and 2.5 (with respect to the default size of 64x64px). Smaller scales are used to detect cars further away on the horizon, increasing them as closer the cars are to the observer. Windows are 75% overlapping, this value was chosen as a compromise between a more accurate localization (higher overlapping) and an acceptable processing time (less windows, so less overlapping). A graphical description of the windows used is shown as follows.

![alt text][windows]

In order to obtain a more accurate localization of the cars while speeding up the processing, this kind of search covering the whole road area is actually performed only once every 50 frames (50 looked like a good compromise after a few tests on the video stream). In all other cases, the search area is limited to the area around which the last detections were. When a limited area is used, the number of search windows is reduced, hence speeding up the search. As I have observed that an overlap larger than 75%, 87.5%, is beneficial to detect cars that are further away, I temporarily increase it when searching in a limited area with the smallest window size (1.0x 64x64px). This produced smoother and less flickery localizations of distant cars.

Another step in the direction of a faster algorithm included, as described in the lessons, using an image-wide HOG feature extraction instead of rerunning the algorithm on each window.

#### 2. Show some examples of test images to demonstrate how your pipeline is working.  What did you do to optimize the performance of your classifier?

To summarize, I searched on 4 scales using YCrCb 3-channel HOG features plus spatially binned color and histograms of color in the feature vector. Plus, I used a dynamic search area every 50 frames and I smoothed the final results by averaging over the last 4 frames. Here are some example images:

![alt text][detection]

### Video Implementation

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (somewhat wobbly or unstable bounding boxes are ok as long as you are identifying the vehicles most of the time with minimal false positives.)

Here's a [link to my video result](./videos/project_video.mp4)


#### 2. Describe how (and identify where in your code) you implemented some kind of filter for false positives and some method for combining overlapping bounding boxes.



Once the image has been searched for cars, each new detection ("on" windows, where a car has been detected) is summed up in a frame heatmap and this information is appended to a FIFO list of the last N frames (a depth of 4 proved a good compromise between reaction time and smoothing effect). The best heatmap for this frame is then computed as the threesholded sum of all the elements in this FIFO list. In between, I also considered using gaussian blur to blur the heatmap to further smoothen the result, but it hasn't been adding much so I excluded it from the final version. This is a simple and effective approach as it acts as a spatial running average. I also considered other approaches involving detecting the crentroid of the hot areas and then tracking it but proved to be complicated to apply and quite tricky when one vehicle overtakes and cover another.





I recorded the positions of positive detections in each frame of the video.  From the positive detections I created a heatmap and then thresholded that map to identify vehicle positions.  I then used `scipy.ndimage.measurements.label()` to identify individual blobs in the heatmap.  I then assumed each blob corresponded to a vehicle.  I constructed bounding boxes to cover the area of each blob detected.  

Here's an example result showing the heatmap from a series of frames of video, the result of `scipy.ndimage.measurements.label()` and the bounding boxes then overlaid on the last frame of video:

### Here are six frames and their corresponding heatmaps:

![alt text][image5]

### Here is the output of `scipy.ndimage.measurements.label()` on the integrated heatmap from all six frames:
![alt text][image6]

### Here the resulting bounding boxes are drawn onto the last frame in the series:
![alt text][image7]



---

###Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Here I'll talk about the approach I took, what techniques I used, what worked and why, where the pipeline might fail and how I might improve it if I were going to pursue this project further.  

